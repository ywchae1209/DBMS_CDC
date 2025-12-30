* package 분할
* format-document to source
* layer 1 Message-codecs complete
* **not-test**

## PgCodecs.scala
```scala
package io.flux
package capture.postgresql
import capture.postgresql.PgFields.*
import capture.postgresql.PgCodecs.*
import capture.postgresql.PgOutputMsg.*

import cats.effect.IO
import fs2.{Pipe, Stream}
import org.postgresql.replication.PGReplicationStream
import scodec.*
import scodec.bits.*
import scodec.codecs.*

import java.time.ZoneOffset
import scala.concurrent.duration.*
import scala.util.Try

// todo
object BinaryValueConverter {


  // Postgres Timestamp: start from 2000-01-01 00:00:00
  private val PG_EPOCH_DIFF = 946684800000L

  /*
      21          INT2 (SmallInt)    2 bytes     Short (Big-endian)
      23          INT4 (Integer)     4 bytes     Int (Big-endian)
      20          INT8 (BigInt)      8 bytes     Long (Big-endian)
      700         FLOAT4 (Real)      4 bytes     Float.intBitsToFloat(readInt)
      701         FLOAT8 (Double)    8 bytes     Double.longBitsToDouble(readLong)
      16          BOOL               1 byte      0x01이면 true, 0x00이면 false
      25 / 1043   TEXT / VARCHAR     Variable    UTF-8 문자열로 디코딩
      1114        TIMESTAMP          8 bytes     2000-01-01 기준의 microseconds (Long)
      1184        TIMESTAMPTZ        8 bytes     위와 동일 (UTC 기준)
      1700        NUMERIC            Variable

   */
  def convert(oid: Int, bytes: ByteVector): Any = {
    val buffer = bytes.toByteBuffer

    oid match {
      case 23 => buffer.getInt      // Int4
      case 20 => buffer.getLong     // Int8
      case 21 => buffer.getShort    // Int2
      case 16 => buffer.get() != 0  // Bool
      case 25 | 1043 =>             // Text / VarChar
        bytes.decodeUtf8.getOrElse("")

      case 700 => buffer.getFloat   // Float4
      case 701 => buffer.getDouble  // Float8

      case 1114 =>                  // Timestamp (Nanos/Micros handling)
        val micros = buffer.getLong
        val millis = (micros / 1000) + PG_EPOCH_DIFF
        java.time.Instant.ofEpochMilli(millis)
          .atOffset(ZoneOffset.UTC)
          .toLocalDateTime

      case 1184 =>                  // TimestampTZ
        val micros = buffer.getLong
        val millis = (micros / 1000) + PG_EPOCH_DIFF
        java.time.Instant.ofEpochMilli(millis).atOffset(ZoneOffset.UTC)

      case _ =>
        // 처리되지 않은 타입은 원본 바이트 또는 텍스트 시도
        bytes.decodeUtf8.getOrElse(bytes.toHex)
    }
  }

  import scodec.bits.ByteVector

  import java.math.{BigDecimal, BigInteger, MathContext}

  /*
      https://github.com/postgres/postgres/blob/master/src/backend/utils/adt/numeric.c
      Offset	Field	  Type    Description
      0       ndigits	Int16   digits 배열의 요소 개수
      2       weight	Int16   첫 번째 digit의 10000진수 가중치
      4       sign    Int16   0(POS), 0x4000(NEG), 0xC000(NaN)
      6       dscale	Int16   소수점 이하 자릿수 (Display Scale)
      8~	    digits	Int16[] 각 요소가 0~9999인 10000진수 배열
   */
  def decodeNumeric(bytes: ByteVector): BigDecimal = {
    val buffer = bytes.toByteBuffer

    // 1. 헤더 읽기
    val nDigits = buffer.getShort
    val weight  = buffer.getShort
    val sign    = buffer.getShort
    val dscale  = buffer.getShort

    if (sign == 0xC000.toShort) return null // NaN 처리

    // 2. Base-10000 숫자 배열을 하나의 큰 정수로 합치기
    var unscaledValue = BigInteger.ZERO
    for (_ <- 0 until nDigits) {
      val digit = buffer.getShort
      unscaledValue = unscaledValue.multiply(BigInteger.valueOf(10000)).add(BigInteger.valueOf(digit))
    }

    // 3. 지수(Power) 계산
    // 실제 값 = unscaledValue * 10000 ^ (weight - nDigits + 1)
    val exponent = weight - nDigits + 1
    val factor = BigInteger.valueOf(10000).pow(Math.abs(exponent))

    var resBigDecimal = if (exponent >= 0) {
      new BigDecimal(unscaledValue.multiply(factor))
    } else {
      new BigDecimal(unscaledValue).divide(new BigDecimal(factor), MathContext.DECIMAL128)
    }

    // 4. 부호 및 소수점(dscale) 최종 조정
    if (sign == 0x4000.toShort) resBigDecimal = resBigDecimal.negate()

    // dscale은 소수점 아래 자릿수를 강제함
    resBigDecimal.setScale(dscale, java.math.RoundingMode.HALF_UP)
  }

}

/**
 * [[https://www.postgresql.org/docs/18/protocol-logicalrep-message-formats.html]]
 */
object PgCodecs {

  // --------------------------------------------------------------------------------
  // implicit conversion
  // --------------------------------------------------------------------------------
  given decoderToCodec[A]: Conversion[Decoder[A], Codec[A]] with {
    def apply(dec: Decoder[A]): Codec[A] = new Codec[A] {
      def sizeBound: SizeBound = SizeBound.unknown
      def encode(value: A): Attempt[BitVector] = Attempt.failure(Err("encode not supported"))
      def decode(bits: BitVector): Attempt[DecodeResult[A]] = dec.decode(bits)
    }
  }


  /**{{{
   * tag : Byte1('M')
   *
   * Int32 (TransactionId) -- Xid of the transaction (only present for streamed transactions) see: X_LogicalMsg
   * --------------------------------------
   * Int8   -- Flags; Either 0 for no flags or 1 if the logical decoding message is transactional.
   * Int64 (XLogRecPtr)
   * String -- The prefix of the logical decoding message.
   * Int32  -- Length of the content.
   * Byte_n -- The content of the logical decoding message.
   * }}}
   * [[https://www.postgresql.org/docs/18/protocol-logicalrep-message-formats.html#PROTOCOL-LOGICALREP-MESSAGE-FORMATS-MESSAGE]]
   *
   * see: [[PgCodecs.x_logicalMsgCodec]]
   */
  val logicalMsgCodec: Codec[LogicalMsg] =
    ( bool(8) :: c_lsn :: c_prefix :: variableSizeBytes(int32, bytes) ).as[LogicalMsg]

  /**{{{
   * tag : Byte1('M')
   *
   * in streamed transaction
   * }}}
   * see : [[PgCodecs.logicalMsgCodec]]
   */
  val x_logicalMsgCodec:Codec[X_LogicalMsg] = (c_xid :: logicalMsgCodec).as[X_LogicalMsg]

  /**{{{
   * tag: Byte1('B')
   * --------------------------------------
   * Int64 (XLogRecPtr) --  The final LSN of the transaction.
   * Int64 (TimestampTz) -- Commit timestamp of the transaction. PostgreSQL epoch (2000-01-01).
   * Int32 (TransactionId) --  Xid of the transaction.
   * }}}
   * [[https://www.postgresql.org/docs/18/protocol-logicalrep-message-formats.html#PROTOCOL-LOGICALREP-MESSAGE-FORMATS-BEGIN]]
   */
  val beginCodec: Codec[Begin] =
    (c_lsn :: c_ts :: c_xid).as[Begin]

  /**{{{
   * tag: Byte1('C')
   * --------------------------------------
   * Int8(0) -- Flags; currently unused.    :: todo
   * Int64 (XLogRecPtr) -- The LSN of the commit.
   * Int64 (XLogRecPtr) -- The end LSN of the transaction.
   * Int64 (TimestampTz) -- * Commit timestamp of the transaction. PostgreSQL epoch (2000-01-01).
   * }}}
   * [[https://www.postgresql.org/docs/18/protocol-logicalrep-message-formats.html#PROTOCOL-LOGICALREP-MESSAGE-FORMATS-COMMIT]]
   */
  val commitCodec: Codec[Commit] =
    (uint8 :: c_lsn :: c_lsn :: c_ts).as[Commit]

  /**{{{
   * repeat in Relation msg.
   * --------------------------------------
   * Int8 -- Flags for the column. Currently can be either 0 for no flags or 1 which marks the column as part of the key.
   * String -- Name of the column.
   * Int32 (Oid) -- OID of the column's data type.
   * Int32 -- Type modifier of the column (atttypmod).
   * }}}
   * # see [[PgCodecs.relationCodec]]
   */
  val relationColumnCod: Codec[RelationColumn] =
    ( bool(8):: c_columnName :: c_typeOid :: c_atpMod ).as[RelationColumn]

  /**{{{
   * tag: Byte1('R')
   *
   * Int32 (TransactionId) -- Xid of the transaction (only present for streamed transactions). see: x_Relation
   * --------------------------------------
   * Int32 (Oid) -- OID of the relation.
   * String -- Namespace (empty string for pg_catalog).
   * String -- Relation name.
   * Int8 -- Replica identity setting for the relation (same as relreplident in pg_class).
   * Int16 -- Number of columns.
   * [RelationColumn] -- repeat
   * }}}
   *
   * see [[PgCodecs.relationColumnCod]]
   *
   * [[https://www.postgresql.org/docs/18/protocol-logicalrep-message-formats.html#PROTOCOL-LOGICALREP-MESSAGE-FORMATS-RELATION]]
   */
  val relationCodec: Codec[Relation] =
    (c_relId :: c_namespace :: c_tableName :: c_replicaIdentity :: vectorOfN(uint16, relationColumnCod)).as[Relation]

  /**{{{
   * tag: Byte1('R')
   *
   * in streamed transaction
   * }}}
   * see : [[PgCodecs.relationCodec]]
   */
  val x_relationCodec  :Codec[X_Relation  ] = (c_xid :: relationCodec).as[X_Relation]          // R

  /**{{{
   * tag : Byte1('Y')
   *
   * Int32 (TransactionId) -- Xid of the transaction (only present for streamed transactions). see: X_TypeMsg
   *
   * Int32 (Oid) -- OID of the data type.
   * String -- Namespace (empty string for pg_catalog).
   * String -- Name of the data type.
   * }}}
   * [[https://www.postgresql.org/docs/18/protocol-logicalrep-message-formats.html#PROTOCOL-LOGICALREP-MESSAGE-FORMATS-TYPE]]
   *
   */
  val typeMsgCodec: Codec[TypeMsg] =
    (c_typeOid :: c_namespace :: c_typeName).as[TypeMsg]

  /**{{{
   * tag : Byte1('Y')
   *
   * in streamed transaction
   * }}}
   * see : [[PgCodecs.typeMsgCodec]]
   */
  val x_typeMsgCodec:Codec[X_TypeMsg] = (c_xid :: typeMsgCodec).as[X_TypeMsg]

  /**{{{
   * ColumnValue
   * --------------------------------------
   * byte1 -- value type ==
   *           | n (Null)
   *           | u (unchanged TOAST; actual value is not sent)
   *           | t (text format)
   *           | b (binary)
   * int32 -- Length of the column value. ( binary or text type)
   * Byte_n -- The value of the column (binary or text)
   * }}}
   */
  val columnValueCod: Codec[ColumnValue] = byte.flatMap {
    case 'n' => println("null"); provide(ColumnValue.Null)
    case 'u' => println("unchangedToast"); provide(ColumnValue.UnchangedToast) // todo
    case 't' => println("text"); int32.flatMap(len => bytes(len).map(bv => ColumnValue.Text(bv.decodeUtf8.getOrElse(""))))
    case 'b' => println("binary"); int32.flatMap(len => bytes(len).map(ColumnValue.Binary.apply))
    case o   => println(s"unknown: $o"); fail(Err(s"Unknown column value tag: ${o.toChar}"))
  }

  /**{{{
   * TupleData
   * --------------------------------------
   * Int16 -- Number of columns.
   * [columnValue] -- repeat
   * }}}
   * [[https://www.postgresql.org/docs/18/protocol-logicalrep-message-formats.html#PROTOCOL-LOGICALREP-MESSAGE-FORMATS-TUPLEDATA]]
   *
   * see [[PgCodecs.columnValueCod]]
   */
  val tupleDataCod: Codec[TupleData] = vectorOfN(int16, columnValueCod).as[TupleData]

  /**{{{
   * tag : Byte1('I')
   *
   * Int32 (TransactionId) -- Xid of the transaction (only present for streamed transactions). see: X_Insert
   * --------------------------------------
   * Int32 (Oid) -- OID of the relation corresponding to the ID in the relation message.
   * Byte1('N') -- Identifies the following TupleData message as a new tuple.
   * TupleData -- TupleData message part representing the contents of new tuple.
   * }}}
   * [[https://www.postgresql.org/docs/18/protocol-logicalrep-message-formats.html#PROTOCOL-LOGICALREP-MESSAGE-FORMATS-INSERT]]
   *
   * # see : [[PgCodecs.tupleDataCod]]
   */
  val insertCodec: Codec[Insert] =
    (c_relId :: constant(BitVector('N'.toByte)) ~> tupleDataCod).as[Insert]

  /**{{{
   * tag : Byte1('I')
   *
   * in streamed transaction
   * }}}
   * see : [[PgCodecs.insertCodec]]
   */
  val x_insertCodec: Codec[X_Insert] = (c_xid :: insertCodec).as[X_Insert]

  /**{{{
   * tag : Byte1('U')
   *
   * Int32 (TransactionId) -- Xid of the transaction (only present for streamed transactions). see: X_Update
   *
   * --------------------------------------
   * Int32 (Oid) -- OID of the relation corresponding to the ID in the relation message.
   * (Byte1 ~ TupleData) -- type of following TupleData submessage
   *          | K ( key ) only present if the update changed data in any of the column(s) that are part of the REPLICA IDENTITY index
   *          | O (old tuple ) only present if table in which the update happened has REPLICA IDENTITY set to FULL.
   *          ~ TupleData (contents of the old tuple or primary key. ) only present if the previous is 'O' or 'K
   *
   * Byte1('N') -- indicate following TupleData as a new tuple.
   * TupleData -- contents of a new tuple.
   * }}}
   * [[https://www.postgresql.org/docs/18/protocol-logicalrep-message-formats.html#PROTOCOL-LOGICALREP-MESSAGE-FORMATS-UPDATE]]
   * see : [[PgCodecs.tupleDataCod]]
   *
   */
  val updateCodec: Codec[Update] =
    c_relId.flatMap(rId =>
      byte.flatMap(tag =>
        (tag match {
          case 'K' | 'O' => (tupleDataCod <~ constant(BitVector('N'.toByte))).flatMap(old => tupleDataCod.map(newR => (Some(old), newR)))
          case 'N' => tupleDataCod.map(newR => (None, newR))
          case _ => fail(Err(s"Expected K, O or N in Update, but got ${tag.toChar}"))
        }).map(res => Update(rId, res._1, res._2))
      ) )

  /**{{{
   * tag : Byte1('U')
   *
   * in streamed transaction
   * }}}
   * see : [[PgCodecs.updateCodec]]
   */
  val x_updateCodec: Codec[X_Update] = (c_xid :: updateCodec).as[X_Update]

  /**{{{
   * tag : Byte1('D')
   *
   * Int32 (TransactionId) -- Xid of the transaction (only present for streamed transactions). see: X_Delete
   *
   * --------------------------------------
   * Int32 (Oid) -- OID of the relation corresponding to the ID in the relation message.
   * Byte1 -- tupleData type
   *          | K (as key) present if the table in which the delete has happened uses an index as REPLICA IDENTITY.
   *          | O (as old tuple) present if the table in which the delete happened has REPLICA IDENTITY set to FULL.
   *
   * TupleData -- old tuple or primary key
   * }}}
   * [[https://www.postgresql.org/docs/18/protocol-logicalrep-message-formats.html#PROTOCOL-LOGICALREP-MESSAGE-FORMATS-DELETE]]
   */
  val deleteCodec: Codec[Delete] =
    c_relId.flatMap(rId =>
      byte.flatMap(tag =>
        (tag match {
          case 'K' | 'O' => tupleDataCod.map(Some(_))
          case _ => fail(Err(s"Expected K or O in Delete, but got ${tag.toChar}"))
        }).map(old => Delete(rId, old))
      ) )

  /**{{{
   * tag : Byte1('D')
   *
   * in streamed transaction
   * }}}
   * see : [[PgCodecs.deleteCodec]]
   */
  val x_deleteCodec: Codec[X_Delete] = (c_xid :: deleteCodec).as[X_Delete]

  /**{{{
   * tag : Byte1('T')
   *
   * Int32 (TransactionId) -- Xid of the transaction (only present for streamed transactions). see: X_Truncate
   *
   * --------------------------------------
   * Int32 --  Number of relations
   * Int8 --  Option bits for TRUNCATE: 1 for CASCADE, 2 for RESTART IDENTITY
   * Int32 (Oid) -- OID of the relation corresponding to the ID in the relation message. todo:: meaning..?
   *                This field is repeated for each relation.
   * }}}
   * [[https://www.postgresql.org/docs/18/protocol-logicalrep-message-formats.html#PROTOCOL-LOGICALREP-MESSAGE-FORMATS-TRUNCATE]]
   *
   * todo :: order check
   */
  val truncateCodec: Codec[Truncate] =
    (int32.flatMap(n => vectorOfN(provide(n), c_relId)) :: c_truncateOptions).as[Truncate]

  /**{{{
   * tag : Byte1('T') same as truncateCodec
   *
   * in streamed transaction
   * }}}
   * see : [[PgCodecs.truncateCodec]]
   */
  val x_truncateCodec: Codec[X_Truncate] = (c_xid :: truncateCodec).as[X_Truncate]

  /**{{{
   * tag : Byte1('O')
   *
   * --------------------------------------
   * Int64 (XLogRecPtr) -- The LSN of the commit on the origin server.
   * String -- Name of the origin.
   * }}}
   * [[https://www.postgresql.org/docs/18/protocol-logicalrep-message-formats.html#PROTOCOL-LOGICALREP-MESSAGE-FORMATS-ORIGIN]]
   *
   * note: that there can be multiple Origin messages inside a single transaction.
   */
  val originCodec =
    (c_lsn :: c_originName).as[Origin]


  /**{{{
   * tag : Byte1('S')
   *
   * --------------------------------------
   * Int32 (TransactionId) -- Xid of the transaction.
   * Int8 -- A value of 1 indicates this is the first stream segment for this XID, 0 for any other stream segment.
   * }}}
   * [[https://www.postgresql.org/docs/18/protocol-logicalrep-message-formats.html#PROTOCOL-LOGICALREP-MESSAGE-FORMATS-STREAM-START]]
  */
  val streamStartCodec: Codec[StreamStart] =
    (c_xid :: c_segment).as[StreamStart]

  /**{{{
   * tag : Byte1('E')
   *
   * no-body
   * }}}
   * [[https://www.postgresql.org/docs/18/protocol-logicalrep-message-formats.html#PROTOCOL-LOGICALREP-MESSAGE-FORMATS-STREAM-STOP]]
   */
  val streamStopCodec =
    provide(PgOutputMsg.StreamStop)

  /**{{{
   * tag : Byte1('c')
   *
   * --------------------------------------
   * Int32 (TransactionId) -- Xid of the transaction.
   * Int8(0) -- Flags; currently unused.
   * Int64 (XLogRecPtr) -- The LSN of the commit.
   * Int64 (XLogRecPtr) -- The end LSN of the transaction.
   * Int64 (TimestampTz) -- Commit timestamp of the transaction. PostgreSQL epoch (2000-01-01).
   * }}}
   * [[https://www.postgresql.org/docs/18/protocol-logicalrep-message-formats.html#PROTOCOL-LOGICALREP-MESSAGE-FORMATS-STREAM-COMMIT]]
   */
  val streamCommitCodec: Codec[StreamCommit] =
    (c_xid :: c_commitFlag :: c_lsn :: c_lsn :: c_ts).as[StreamCommit]


  /**{{{
   *
   * optional in streamAbortCodec
   * : present only when streaming is set to parallel.(protocol version 4~)
   *
   * Int64 (XLogRecPtr) -- The LSN of the abort operation,
   * Int64 (TimestampTz) -- Abort timestamp of the transaction, PostgreSQL epoch (2000-01-01).
   * }}}
   */
  val streamAbortSubCod: Codec[StreamAboutSub] = (c_lsn :: c_ts).as[StreamAboutSub]

  /**{{{
   * tag : Byte1('A')
   *
   * --------------------------------------
   * Int32 (TransactionId) -- Xid of the transaction.
   * Int32 (TransactionId) -- Xid of the subtransaction (will be same as xid of the transaction for top-level transactions).
   *
   * below is optional : present only when streaming is set to parallel.(protocol version 4~)
   * Int64 (XLogRecPtr) -- The LSN of the abort operation,
   * Int64 (TimestampTz) -- Abort timestamp of the transaction, PostgreSQL epoch (2000-01-01).
   * }}}
   * [[https://www.postgresql.org/docs/18/protocol-logicalrep-message-formats.html#PROTOCOL-LOGICALREP-MESSAGE-FORMATS-STREAM-ABORT]]
   */
  val streamAbortCodec: Codec[StreamAbort] = {
    (c_xid :: c_xid :: optional(provide(true), streamAbortSubCod)).as[StreamAbort]
  }

  /**{{{
   * tag : Byte1('b')
   * (protocol version 3~)
   *
   * --------------------------------------
   * Int64 (XLogRecPtr) -- The LSN of the prepare.
   * Int64 (XLogRecPtr) -- The end LSN of the prepared transaction.
   * Int64 (TimestampTz) -- Prepare timestamp of the transaction. PostgreSQL epoch (2000-01-01).
   * Int32 (TransactionId) -- Xid of the transaction.
   * String --  The user defined GID of the prepared transaction.
   * }}}
   * [[https://www.postgresql.org/docs/18/protocol-logicalrep-message-formats.html#PROTOCOL-LOGICALREP-MESSAGE-FORMATS-BEGIN-PREPARE]]
   */
  val beginPrepareCodec: Codec[BeginPrepare] =
    ( c_lsn :: c_lsn :: c_ts :: c_xid :: c_gid).as[BeginPrepare]

  /**{{{
   * tag : Byte1('P')
   *
   * --------------------------------------
   * Int8(0) -- Flags; currently unused.
   * Int64 (XLogRecPtr) -- The LSN of the prepare.
   * Int64 (XLogRecPtr) -- The end LSN of the prepared transaction.
   * Int64 (TimestampTz) -- Prepare timestamp of the transaction. PostgreSQL epoch (2000-01-01).
   * Int32 (TransactionId) -- Xid of the transaction.
   * String -- * The user defined GID of the prepared transaction.
   * }}}
   * [[https://www.postgresql.org/docs/18/protocol-logicalrep-message-formats.html#PROTOCOL-LOGICALREP-MESSAGE-FORMATS-PREPARE]]
   */
  val prepareCodec: Codec[Prepare] =
    ( byte :: c_lsn :: c_lsn :: c_ts :: c_xid :: c_gid).as[Prepare]

  /**{{{
   * tag : Byte1('K')
   *
   * --------------------------------------
   * Int8(0) -- Flags; currently unused.
   * Int64 (XLogRecPtr) -- The LSN of the commit of the prepared transaction.
   * Int64 (XLogRecPtr) -- The end LSN of the commit of the prepared transaction.
   * Int64 (TimestampTz) -- Commit timestamp of the transaction. PostgreSQL epoch (2000-01-01).
   * Int32 (TransactionId) -- Xid of the transaction.
   * String -- The user defined GID of the prepared transaction.
   * }}}
   * [[https://www.postgresql.org/docs/18/protocol-logicalrep-message-formats.html#PROTOCOL-LOGICALREP-MESSAGE-FORMATS-COMMIT-PREPARED]]
   */
  val commitPreparedCodec: Codec[CommitPrepared] =
    ( byte :: c_lsn :: c_lsn :: c_ts :: c_xid :: c_gid).as[CommitPrepared]


  /**{{{
   * tag : Byte1('r')
   *
   * Int8(0) -- Flags; currently unused.
   * Int64 (XLogRecPtr) -- The end LSN of the prepared transaction.
   * Int64 (XLogRecPtr) -- The end LSN of the rollback of the prepared transaction.
   * Int64 (TimestampTz) -- Prepare timestamp of the transaction. PostgreSQL epoch (2000-01-01).
   * Int64 (TimestampTz) -- Rollback timestamp of the transaction. PostgreSQL epoch (2000-01-01).
   * Int32 (TransactionId) -- Xid of the transaction.
   * String -- The user defined GID of the prepared transaction.
   * }}}
   * [[https://www.postgresql.org/docs/18/protocol-logicalrep-message-formats.html#PROTOCOL-LOGICALREP-MESSAGE-FORMATS-ROLLBACK-PREPARED]]
   */
  val rollbackPreparedCodec: Codec[RollbackPrepared] =
    ( byte :: c_lsn :: c_lsn :: c_ts :: c_ts :: c_xid :: c_gid).as[RollbackPrepared]

  /**{{{
   * tag : Byte1('p')
   *
   * Int8(0) -- Flags; currently unused.
   * Int64 (XLogRecPtr) -- The LSN of the prepare.
   * Int64 (XLogRecPtr) -- The end LSN of the prepared transaction.
   * Int64 (TimestampTz) -- Prepare timestamp of the transaction. PostgreSQL epoch (2000-01-01).
   * Int32 (TransactionId) -- Xid of the transaction.
   * String -- The user defined GID of the prepared transaction.
   * }}}
   * [[https://www.postgresql.org/docs/18/protocol-logicalrep-message-formats.html#PROTOCOL-LOGICALREP-MESSAGE-FORMATS-STREAM-PREPARE]]
   */
  val streamPrepareCodec: Codec[StreamPrepare] =
    ( byte :: c_lsn :: c_lsn :: c_ts :: c_xid :: c_gid).as[StreamPrepare]

  /**
   * in stream mode, use x_ codec
   */
  def decoder(isStream: Boolean): Map[Char, Codec[PgOutputMsg]] = Map(
    'B' -> PgCodecs.beginCodec,
    'C' -> PgCodecs.commitCodec,
    'M' -> (if(isStream) PgCodecs.x_logicalMsgCodec else PgCodecs.logicalMsgCodec),
    'R' -> (if(isStream) PgCodecs.x_relationCodec else PgCodecs.relationCodec),
    'Y' -> (if(isStream) PgCodecs.x_typeMsgCodec else PgCodecs.typeMsgCodec),
    'I' -> (if(isStream) PgCodecs.x_insertCodec else PgCodecs.insertCodec),
    'U' -> (if(isStream) PgCodecs.x_updateCodec else PgCodecs.updateCodec),
    'D' -> (if(isStream) PgCodecs.x_deleteCodec else PgCodecs.deleteCodec),
    'T' -> (if(isStream) PgCodecs.x_truncateCodec else PgCodecs.truncateCodec),
    'O' -> PgCodecs.originCodec,
    'S' -> PgCodecs.streamStartCodec,
    'E' -> PgCodecs.streamStopCodec,
    'c' -> PgCodecs.streamCommitCodec,
    'A' -> PgCodecs.streamAbortCodec,
    'b' -> PgCodecs.beginPrepareCodec,
    'P' -> PgCodecs.prepareCodec,
    'K' -> PgCodecs.commitPreparedCodec,
    'r' -> PgCodecs.rollbackPreparedCodec,
    'p' -> PgCodecs.streamPrepareCodec,
  )

  val normalDecoders = decoder(false)
  val streamDecoders = decoder(true)

  def getDecoder(tag: Char, isStream: Boolean): Codec[PgOutputMsg]
  = if(isStream) streamDecoders(tag) else normalDecoders(tag)
}

// --------------------------------------------------------------------------------
object PgOutputParser {

  // replication to fs2-Stream
  // note) set interval larger than 200 millis(ex: 5.seconds). i'll check at every 100 millis.
  def replicationSource( stream: PGReplicationStream,
                         interval: FiniteDuration = 500.millis )
  : Stream[IO, BitVector] = {

    val idleSleep = 100.millis

    def step(idleTime: FiniteDuration): Stream[IO, BitVector] =
      Stream.eval(IO.blocking(stream.readPending())).flatMap {
        case bytes if bytes != null =>
          Stream.emit(BitVector(bytes)) ++ step(0.millis)
        case _ =>
          if idleTime < interval then
            Stream.sleep[IO](idleSleep) >> step(idleTime + idleSleep)
          else
            println(".")  // todo
            Stream.eval(IO.blocking(stream.forceUpdateStatus())) >> Stream.sleep[IO](idleSleep) >> step(0.millis)
      }

    step(0.millis)
  }

  case class DecodeTask( payload: BitVector, isStream: Boolean, decoder: Codec[PgOutputMsg] )

  // assign codec
  val toDecodeTasks: Pipe[IO, BitVector, DecodeTask] =
    _.scan((false, Option.empty[DecodeTask])){ case ((isStream, _), bv) =>

      val tag = bv.getByte(0).toChar
      val mode = tag match
        case 'S'             => true    // Stream Start
        case 'E' | 'c' | 'A' => false   // Stream End/Commit/Abort
        case _               => isStream

      val d = getDecoder(tag, mode)
      val t = DecodeTask(bv, mode, d)
      (mode, Some(t))

    }.collect { case (_, Some(task)) => task }


  def decodeMessage(bits: BitVector)
  : Attempt[DecodeResult[PgOutputMsg]] = for {
    header <- byte.decode(bits)
    tag    = header.value.toChar
    payload= header.remainder
    result <- scala.util.Try(getDecoder(tag, false).decode(payload)).toEither
      .fold(
        e => Attempt.failure(Err(s"Unknown pgoutput tag($tag): ${e.toString}")),
        r => r )
  } yield result

  def decodeAll(bits: BitVector): Either[String, Vector[PgOutputMsg]] = {
    def loop(rem: BitVector, acc: Vector[PgOutputMsg]): Attempt[Vector[PgOutputMsg]] = {
      if rem.isEmpty then
        Attempt.successful(acc)
      else
        decodeMessage(rem).flatMap(res => loop(res.remainder, acc :+ res.value))
    }
    loop(bits, Vector.empty).toEither.left.map(_.messageWithContext)
  }
}

```

## PgOutputMsg.scala
```scala
package io.flux
package capture.postgresql

import capture.postgresql.PgFields.*
import capture.postgresql.PgOutputMsg.*

import scodec.*
import scodec.bits.*

/** 26 co-product types in Pg18 */
sealed trait PgOutputMsg {

  // todo
  override def toString: String = this match
    case LogicalMsg( isTransactional, lsn, prefix, content) =>
      s"LogicalMsg( transactional: $isTransactional, lsn: ${lsn.format}, prefix: $prefix, content: $content)"

    case Begin(finalLsn, commitTs, xid) =>
      s"Begin(finalLsn: ${finalLsn.format}, commitTs: ${commitTs.format}, xid: $xid)"

    case Commit(flags, commitLsn, endLsn, commitTs) =>
      s"Commit(flags: $flags, commitLsn: ${commitLsn.format}, endLsn: ${endLsn.format}, commitTs: $commitTs)"

    case Relation( relId, namespace, name, replicaIdentity, columns) =>
      s"Relation(relId: $relId, namespace: $namespace, name: $name, replicaIdentity: $replicaIdentity, columns: " +
      columns.mkString("[ ", ", ", "]")

    case TypeMsg( dataTypeId, schemaName, typeName) =>
      s"TypeMessage( dataTypeId: $dataTypeId, schemaName: $schemaName, typeName: $typeName)"

    case Insert(relId, neo) =>
      s"Insert(relId: $relId, neo: $neo)"

    case Update(relId, oldOrKey, neo) =>
      s"Update(relId: $relId, oldOrKey: ${oldOrKey.mkString}, neo: $neo )"

    case Delete(relId, oldOrKey) =>
      s"Delete(relId: $relId, oldOrKey: ${oldOrKey.mkString})"

    case Truncate(relIds, options) =>
      relIds.mkString("Truncate(relIds: [", " ,", "], ")  + s"options: $options)"

    case Origin(lsn, name) =>
      s"Origin(name: $name, lsn: ${lsn.format})"

    case StreamStart(xid, segment) => super.toString
    case StreamStop => "StreamStop"
    case StreamCommit(xid, flag, commitLsn, endLsn, commitTs) => super.toString
    case StreamAbort(xid, subxid, abortInParallel) => super.toString
    case X_LogicalMsg(xid, msg) => super.toString
    case X_Relation(xid, relation) => super.toString
    case X_Insert(xid, insert) => super.toString
    case X_Update(xid, update) => super.toString
    case X_Delete(xid, delete) => super.toString
    case X_Truncate(xid, truncate) => super.toString
    case X_TypeMsg(xid, typeMessage) => super.toString
    case BeginPrepare(prepareLsn, endLsn, prepareTs, xid, gid) => super.toString
    case Prepare(flags, prepareLsn, endLsn, prepareTs, xid, gid) => super.toString
    case CommitPrepared(flags, commitLsn, endLsn, commitTs, xid, gid) => super.toString
    case RollbackPrepared(flags, prepareLsn, endLsn, prepareTs, rollbackTs, xid, gid) => super.toString
    case StreamPrepare(flags, prepareLsn, endLsn, prepareTs, xid, gid) => super.toString
    case o => "unknown: " + super.toString

}

object PgOutputMsg {

  /** {{{
   *  when application call pg_logical_emit_message, contents directly written to WAL.
   *  Used for "user-defined-meta-data" or "event-signal" insert to WAL
   *  such as application-check-point, audit-tracking-log, system-synchronize
   * }}}
   * see [[PgCodecs.logicalMsgCodec]]
   */
  case class LogicalMsg(transactional: Boolean, lsn: PgLsn, prefix: Prefix, content: ByteVector) extends PgOutputMsg

  /** see [[PgCodecs.beginCodec]] */
  case class Begin(finalLsn: PgLsn, commitTsMicros: PgTimestamp, xid: PgXid) extends PgOutputMsg

  /** see [[PgCodecs.commitCodec]] */
  case class Commit(flags: Int, commitLsn: PgLsn, endLsn: PgLsn, commitTsMicros: PgTimestamp) extends PgOutputMsg

  /** see [[PgCodecs.relationColumnCod]] */
  case class RelationColumn(isKey: Boolean, name: ColumnName, typeOid: TypeOid, atpmod: AtpMod) {
    override def toString: String = s"Column(flags: $isKey, name: $name, typeOid: $typeOid, atttypmod: $atpmod)"
  }
  /** see [[PgCodecs.relationCodec]] */
  case class Relation(relId: RelId, namespace: Namespace, name: TableName, replicaIdentity: ReplicaIdentity, columns: Vector[RelationColumn]) extends PgOutputMsg

  /** see [[PgCodecs.typeMsgCodec]] */
  case class TypeMsg(typeOid: TypeOid, schemaName: Namespace, typeName: TypeName) extends PgOutputMsg

  /** see [[PgCodecs.insertCodec]] */
  case class Insert(relId: RelId, neo: TupleData) extends PgOutputMsg

  /** see [[PgCodecs.updateCodec]] */
  case class Update(relId: RelId, oldOrKey: Option[TupleData], newRow: TupleData) extends PgOutputMsg

  /** see [[PgCodecs.deleteCodec]] */
  case class Delete(relId: RelId, oldOrKey: Option[TupleData]) extends PgOutputMsg

  /** see [[PgCodecs.truncateCodec]] */
  case class Truncate(relIds: Vector[RelId], options: TruncateOptions) extends PgOutputMsg

  /** for BDR(Bi-Directional Replcation): used to prevent loop update. :: see [[PgCodecs.originCodec]] */
  case class Origin(lsn: PgLsn, name: OriginName) extends PgOutputMsg

  // Stream Messages :: only present for streamed transactions; since protocol version 2.
  // --------------------------------------------------------------------------------

  /** see [[PgCodecs.streamStartCodec]] */
  case class StreamStart(xid: PgXid, segment: Segment) extends PgOutputMsg

  /** see [[PgCodecs.streamStopCodec]] */
  case object StreamStop extends PgOutputMsg

  /** see [[PgCodecs.streamCommitCodec]] */
  case class StreamCommit(xid: PgXid, flag: CommitFlag, commitLsn: PgLsn, endLsn: PgLsn, commitTs: PgTimestamp) extends PgOutputMsg

  /** present only when streaming is set to parallel; since version 4 :: see :: [[PgCodecs.streamAbortSubCod]] */
  case class StreamAboutSub(abortLsn: PgLsn, abortTs: PgTimestamp)

  /** see [[PgCodecs.streamAbortCodec]] */
  case class StreamAbort(xid: PgXid, subxid: PgXid, abortInParallel: Option[StreamAboutSub]) extends PgOutputMsg

  /** see [[PgCodecs.x_logicalMsgCodec]] */
  case class X_LogicalMsg(xid: PgXid, msg: LogicalMsg) extends PgOutputMsg

  /** see [[PgCodecs.x_relationCodec]] */
  case class X_Relation(xid: PgXid, relation: Relation) extends PgOutputMsg

  /** see [[PgCodecs.x_insertCodec]] */
  case class X_Insert(xid: PgXid, insert: Insert) extends PgOutputMsg

  /** see [[PgCodecs.x_updateCodec]] */
  case class X_Update(xid: PgXid, update: Update) extends PgOutputMsg

  /** see [[PgCodecs.x_deleteCodec]] */
  case class X_Delete(xid: PgXid, delete: Delete) extends PgOutputMsg

  /** see [[PgCodecs.x_truncateCodec]] */
  case class X_Truncate(xid: PgXid, truncate: Truncate) extends PgOutputMsg

  /** see [[PgCodecs.x_typeMsgCodec]] */
  case class X_TypeMsg(xid: PgXid, typeMessage: TypeMsg) extends PgOutputMsg

  // 2 PC related Message -------------------------------------

  /** Begin Prepare: 2PC 트랜잭션의 시작 :: see [[PgCodecs.beginPrepareCodec]] */
  case class BeginPrepare(prepareLsn: PgLsn, endLsn: PgLsn, prepareTs: PgTimestamp, xid: PgXid, gid: GID) extends PgOutputMsg

  /** Prepare: 2PC 준비 완료 :: see [[PgCodecs.prepareCodec]] */
  case class Prepare( flags: Byte, prepareLsn: PgLsn, endLsn: PgLsn, prepareTs: PgTimestamp, xid: PgXid, gid: GID ) extends PgOutputMsg

  /** Commit Prepared: 준비된 트랜잭션 확정 :: see [[PgCodecs.commitPreparedCodec]] */
  case class CommitPrepared(flags: Byte, commitLsn: PgLsn, endLsn: PgLsn, commitTs: PgTimestamp, xid: PgXid, gid: GID ) extends PgOutputMsg


  /** Rollback Prepared: 준비된 트랜잭션 취소 :: see [[PgCodecs.rollbackPreparedCodec]] */
  case class RollbackPrepared( flags: Byte, prepareLsn: PgLsn, endLsn: PgLsn, prepareTs: PgTimestamp, rollbackTs: PgTimestamp, xid: PgXid, gid: GID ) extends PgOutputMsg

  /** Stream Prepare: 스트리밍 중인 트랜잭션의 준비 :: see [[PgCodecs.streamPrepareCodec]] */
  case class StreamPrepare( flags: Byte, prepareLsn: PgLsn, endLsn: PgLsn, prepareTs: PgTimestamp, xid: PgXid, gid: GID ) extends PgOutputMsg

  /** see [[PgCodecs.tupleDataCod]] */
  case class TupleData(values: Vector[ColumnValue]) {
    override def toString: String = values.mkString("TupleData( values: [", ", ", "])")
  }

  /** see [[PgCodecs.columnValueCod]] */
  sealed trait ColumnValue
  object ColumnValue {
    case object Null extends ColumnValue
    case object UnchangedToast extends ColumnValue
    case class Text(value: String) extends ColumnValue
    case class Binary(bytes: ByteVector) extends ColumnValue
  }
}
```

## PgFields.scala
```scala
package io.flux
package capture.postgresql

import scodec.*
import scodec.bits.*
import scodec.codecs.*

/**
 * primitive type-field in pgoutput-plugin
 *
 * [[https://www.postgresql.org/docs/current/protocol-logicalrep-message-formats.html]]
 */
object PgFields {

  // --------------------------------------------------------------------------------
  private val cstring_utf8: Codec[String] = new Codec[String] {
    def sizeBound = SizeBound.unknown
    def encode(value: String) = Attempt.successful(BitVector(value.getBytes("UTF-8")) ++ BitVector.lowByte)
    def decode(bits: BitVector) = {

      val bv = bits.bytes

      val idx = bv.indexOfSlice(ByteVector(0))
      if (idx < 0) Attempt.failure(Err("CString terminator not found"))
      else {
        val strBytes = bv.take(idx)
        val rest = BitVector(bv.drop(idx + 1))
        strBytes.decodeUtf8 match {
          case Right(s)  => Attempt.successful(DecodeResult(s, rest))
          case Left(err) => Attempt.failure(Err(err.getMessage))
        }
      }
    }
  }


  // --------------------------------------------------------------------------------
  case class TruncateOptions(raw: Int) {
    def isCascade: Boolean = (raw & 0x01) != 0

    def isRestartIdentity: Boolean = (raw & 0x02) != 0

    // v16에서 추가된 옵션들
    def isOnly: Boolean = (raw & 0x10) != 0

    def isRecursive: Boolean = (raw & 0x20) != 0

    override def toString: String =

      val opts = List(
        if (isCascade) Some("CASCADE") else None,
        if (isRestartIdentity) Some("RESTART IDENTITY") else None,
        if (isOnly) Some("ONLY") else None,
        if (isRecursive) Some("RECURSIVE") else None
      ).flatten
      if (opts.isEmpty) s"NONE($raw)" else s"${opts.mkString(", ")}($raw)"
  }

  /**
   * uint8
   */
  val c_truncateOptions: Codec[TruncateOptions] = uint8.xmap(TruncateOptions.apply, _.raw)

  // --------------------------------------------------------------------------------
  sealed trait ReplicaIdentity {
    def code: Byte
  }

  /**
   * byte
   */
  val c_replicaIdentity: Codec[ReplicaIdentity] = byte.xmap( ReplicaIdentity.fromByte, _.code )

  object ReplicaIdentity {
    case object Default extends ReplicaIdentity { val code = 'd'.toByte }
    case object Nothing extends ReplicaIdentity { val code = 'n'.toByte }
    case object Full extends ReplicaIdentity { val code = 'f'.toByte }
    case object Index extends ReplicaIdentity { val code = 'i'.toByte }

    def fromByte(b: Byte): ReplicaIdentity = b.toChar match {
      case 'd' => Default
      case 'n' => Nothing
      case 'f' => Full
      case 'i' => Index
      case _ => Nothing // 정의되지 않은 경우 기본값
    }
  }

  // opaque types
  // --------------------------------------------------------------------------------
  // Int
  // --------------------------------------------------------------------------------
  /**
   * A value of 1 indicates this is the first stream segment for this XID, 0 for any other stream segment.
   */
  opaque type CommitFlag = Int // from int8

  /**
   * int8
   */
  val c_commitFlag: Codec[CommitFlag] = int8.asInstanceOf[Codec[CommitFlag]]

  object CommitFlag {
    def apply(i: Int): CommitFlag = i

    extension (i: CommitFlag) {
      def value: Int = i
      def toInt: Int = i
    }
  }

  // --------------------------------------------------------------------------------
  /**
   * A value of 1 indicates this is the first stream segment for this XID, 0 for any other stream segment.
   */
  opaque type Segment = Int // from int8


  /**
   * int8
   */
  val c_segment: Codec[Segment] = int8.asInstanceOf[Codec[Segment]]

  object Seqment {
    def apply(i: Int): Segment = i

    extension (i: Segment) {
      def value: Int = i
      def toInt: Int = i
    }
  }

  // --------------------------------------------------------------------------------
  opaque type PgXid = Int

  /**
   * int32 -- Xid of transaction
   */
  val c_xid: Codec[PgXid] = int32.asInstanceOf[Codec[PgXid]]

  object PgXid {
    def apply(i: Int): PgXid = i

    extension (i: PgXid) {
      def value: Int = i
      def toInt: Int = i
    }
  }

  // --------------------------------------------------------------------------------
  // Type OID (Object ID)
  opaque type TypeOid = Int

  /**
   * int32
   */
  val c_typeOid: Codec[TypeOid] = int32.asInstanceOf[Codec[TypeOid]]

  object TypeOid {
    def apply(v: Int): TypeOid = v

    extension (oid: TypeOid) {
      def value: Int = oid
      def toInt: Int = oid
    }
  }

  // --------------------------------------------------------------------------------
  // Attribute Type Modifier
  opaque type AtpMod = Int


  /**
   * int32
   */
  val c_atpMod: Codec[AtpMod] = int32.asInstanceOf[Codec[AtpMod]]

  object AtpMod {
    def apply(v: Int): AtpMod = v

    extension (mod: AtpMod) {
      def value: Int = mod
      def toInt: Int = mod
    }
  }

  // --------------------------------------------------------------------------------
  // Relation ID (Table OID)
  opaque type RelId = Int

  /**
   * int32
   */
  val c_relId: Codec[RelId] = int32.asInstanceOf[Codec[RelId]]

  object RelId {
    def apply(v: Int): RelId = v

    extension (id: RelId) {
      def toInt: Int = id
    }
  }

  // --------------------------------------------------------------------------------
  // String
  // --------------------------------------------------------------------------------
  // prefix of the logical decoding message.
  opaque type Prefix = String

  /**
   * cstring_utf8
   */
  val c_prefix: Codec[Prefix] = cstring_utf8.asInstanceOf[Codec[Prefix]]

  object Prefix {
    def apply(v: String): Prefix = v

    extension (cn: Prefix) {
      def value: String = cn
      def toString: String = cn
    }
  }

  // --------------------------------------------------------------------------------
  // Column Name
  opaque type ColumnName = String

  /**
   * cstring_utf8
   */
  val c_columnName: Codec[ColumnName] = cstring_utf8.asInstanceOf[Codec[ColumnName]]

  object ColumnName {
    def apply(v: String): ColumnName = v

    extension (cn: ColumnName) {
      def value: String = cn
      def toString: String = cn
    }
  }

  // --------------------------------------------------------------------------------
  // Namespace (Schema Name) : empty-string for pg_catalog
  opaque type Namespace = String

  /**
   * cstring_utf8
   */
  val c_namespace: Codec[Namespace] = cstring_utf8.asInstanceOf[Codec[Namespace]]

  object Namespace {
    def apply(s: String): Namespace = s

    extension (s: Namespace) {
      def toString: String = s
    }
  }

  // --------------------------------------------------------------------------------
  // TypeName : name fo the data type
  opaque type TypeName = String

  /**
   * cstring_utf8
   */
  val c_typeName: Codec[TypeName] = cstring_utf8.asInstanceOf[Codec[TypeName]]

  object TypeName {
    def apply(s: String): TypeName = s

    extension (s: TypeName) {
      def toString: String = s
    }
  }

  // --------------------------------------------------------------------------------
  // OriginName : name fo the orgin
  opaque type GID = String

  /**
   * cstring_utf8
   */
  val c_gid: Codec[GID] = cstring_utf8.asInstanceOf[Codec[GID]]

  object GID {
    def apply(s: String): GID = s

    extension (s: GID) {
      def toString: String = s
    }
  }
  // --------------------------------------------------------------------------------
  // OriginName : name fo the orgin
  opaque type OriginName = String

  /**
   * cstring_utf8
   */
  val c_originName: Codec[OriginName] = cstring_utf8.asInstanceOf[Codec[OriginName]]

  object OriginName {
    def apply(s: String): OriginName = s

    extension (s: OriginName) {
      def toString: String = s
    }
  }

  // --------------------------------------------------------------------------------
  // TableName
  opaque type TableName = String

  /**
   * cstring_utf8
   */
  val c_tableName: Codec[TableName] = cstring_utf8.asInstanceOf[Codec[TableName]]

  object TableName {
    def apply(s: String): TableName = s

    extension (s: TableName) {
      def toString: String = s
    }
  }

  // --------------------------------------------------------------------------------
  // Long
  // --------------------------------------------------------------------------------
  opaque type PgTimestamp = Long

  /**
   * int64 -- The value is in number of microseconds since PostgreSQL epoch (2000-01-01)
   */
  val c_ts: Codec[PgTimestamp] = int64.asInstanceOf[Codec[PgTimestamp]]

  object PgTimestamp {
    def apply(value: Long): PgTimestamp = value

    extension (ts: PgTimestamp) {
      def toLong: Long = ts
      def toUnixMillis: Long = (ts / 1000) + 946684800000L // pg ts start 2000/1/1

      def +(other: Long): PgTimestamp = ts + other
      def -(other: PgTimestamp): Long = ts - other

      def format: String = {
        val pgEpochMs = 946684800000L
        val millis = (ts / 1000) + pgEpochMs
        java.time.Instant.ofEpochMilli(millis)
          .atOffset(java.time.ZoneOffset.UTC)
          .toString
      }
    }
  }

  // --------------------------------------------------------------------------------
  opaque type PgLsn = Long

  /**
   * int64 -- LSN (XLogRecPtr) of transaction
   */
  val c_lsn: Codec[PgLsn] = int64.asInstanceOf[Codec[PgLsn]]

  object PgLsn {
    def apply(l: Long): PgLsn = l

    extension (l: PgLsn) {
      def value: Long = l
      def format: String = s"${(l >>> 32).toHexString.toUpperCase}/${l.toInt.toHexString.toUpperCase}"
    }
  }

}
```

### main.scala
```scala
package io.flux

import capture.postgresql.PgOutputParser
import capture.postgresql.PgCodecs.*

import cats.effect.{IO, Resource}

import java.util.Properties
import java.sql.DriverManager
import org.postgresql.{PGConnection, PGProperty}
import org.postgresql.replication.PGReplicationStream

import java.sql.Connection
import scala.io.StdIn.readLine

@main
def main(): Unit = {

  import cats.effect.unsafe.implicits.global

  println("Hello world!")

  //
  val pg   = "jdbc:postgresql://localhost:5432"
  val db   = "testdb"
  val url  = s"$pg/$db"
  val user = "testuser"
  val pass = "testpass"
  val slot = "io_flux_slot"
  val pub  = "io_flux_pub"

  // set up : pg replication
  Test.setup( url, user, pass, slot, pub)

  println("start.. press any key to stop")

  // print binary capture
  val go = Test.capture( url, user, pass, slot, pub).start.unsafeRunSync()

  readLine()
  go.cancel.unsafeRunSync()
}

object Test {

  import org.postgresql.PGProperty
  import org.postgresql.jdbc.PgConnection
  import org.postgresql.replication.PGReplicationStream
  import scodec.bits.BitVector

  import java.nio.ByteBuffer
  import java.sql.DriverManager
  import java.util.Properties
  import java.util.concurrent.TimeUnit

  // --------------------------------------------------------------------------------
  private def executeQuery(conn: Connection, sql: String, msg: Option[String] = None): Unit = {
    val stmt = conn.createStatement()
    stmt.executeQuery(sql)
    msg.foreach(println)
  }
  private def executeUpdate(conn: Connection, sql: String, msg: Option[String] = None): Unit = {
    val stmt = conn.createStatement()
    stmt.executeUpdate(sql)
    msg.foreach(println)
  }

  private def executeIgnoreError(conn: Connection, sql: String, msg: Option[String] = None): Unit = {
    try {
      executeUpdate( conn, sql, msg)
    } catch {
      case e: Exception => println(s"Note: ${e.getMessage}")
    }
  }

  private def isSlotExists(conn: Connection, slotName: String): Boolean = {
    val sql = "SELECT 1 FROM pg_replication_slots WHERE slot_name = ?"
    val pstmt = conn.prepareStatement(sql)
    pstmt.setString(1, slotName)
    val rs = pstmt.executeQuery()
    rs.next()
  }

  // --------------------------------------------------------------------------------
  def setup(url: String,
            user: String,
            pass: String,
            slot: String,
            pub: String): Unit = {

    // 1. 일반 모드로 연결 (복제 옵션 없이) -- not containing "replication" property.
    val props = new Properties()
    props.setProperty("user", user)
    props.setProperty("password", pass)

    val conn = DriverManager.getConnection(url, props)
    try {
      // 2. create Publication : If exist, ignroe exception (IF NOT EXISTS는 PG 13+ 지원)
      executeIgnoreError(conn,
        s"CREATE PUBLICATION $pub FOR ALL TABLES",
        Some(s"Publication '$pub' checked/created."))

      // 3. create Replication Slot if not exist.
      if isSlotExists(conn, slot) then
        println(s"Slot '$slot' already exists.")
      else
        executeQuery(conn,
          s"SELECT pg_create_logical_replication_slot('$slot', 'pgoutput')",
          Some(s"Logical slot '$slot' created.")
        )

    } finally {
      conn.close()
    }
  }

  def capture(url: String,
              user: String,
              pass: String,
              slot: String,
              pub: String,
              waitMilli: Int = 100): IO[Unit] = {

    val connectionResource: Resource[IO, PgConnection]
    = Resource.make(IO.blocking {
      val props = new Properties()
      PGProperty.USER.set(props, user)
      PGProperty.PASSWORD.set(props, pass)
      PGProperty.ASSUME_MIN_SERVER_VERSION.set(props, "14")
      PGProperty.REPLICATION.set(props, "database")

      DriverManager.getConnection(url, props).asInstanceOf[PgConnection]
    })(conn => IO.blocking(conn.close()))

    connectionResource.use { conn =>
      IO.blocking {
        conn.getReplicationAPI()
          .replicationStream()
          .logical()
          .withSlotName(slot)
          .withSlotOption("proto_version", 2)
          .withSlotOption("publication_names", pub)
          .withSlotOption("binary", true)
          .start()
      }.flatMap { stream =>
        PgOutputParser.replicationSource(stream)
          .through(PgOutputParser.toDecodeTasks) // 디코더 전달
          .evalMap { t =>
            IO {
              val tag = t.payload.getByte(0).toChar
              val isStream = t.isStream
              println(s"$tag : isStream($isStream)")
            }
          }
          .compile
          .drain
          .handleErrorWith { t =>
            IO.blocking(println(s"에러 발생: ${t.getMessage}"))
          }
      }
    }
  }
}


```
