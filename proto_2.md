### todo
* document to code.
* complete codecs
* parallel :: apply fs2 par
* cost reduction :: how to reduce copy cost.



```scala
package io.flux
package capture.pgoutput
import capture.pgoutput.PgBasicTypes.*
import capture.pgoutput.PgCodecs.*
import capture.pgoutput.PgOutputMessage.*

import cats.effect.IO
import fs2.{Pipe, Stream}
import org.postgresql.replication.PGReplicationStream
import scodec.*
import scodec.bits.*
import scodec.codecs.*

import java.time.ZoneOffset
import scala.concurrent.duration.*
import scala.util.Try

object PgBasicTypes {

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

  // --------------------------------------------------------------------------------
  sealed trait ReplicaIdentity { def code: Byte }

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
  opaque type CommitFlag = Int       // from int8
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
  opaque type Segment = Int       // from int8
  object Seqment {
    def apply(i: Int): Segment = i
    extension (i: Segment) {
      def value: Int = i
      def toInt: Int = i
    }
  }

  // --------------------------------------------------------------------------------
  opaque type PgXid = Int
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
  object Namespace {
    def apply(s: String): Namespace = s
    extension (s: Namespace) {
      def toString: String = s
    }
  }

  // --------------------------------------------------------------------------------
  // TypeName : name fo the data type
  opaque type TypeName = String
  object TypeName {
    def apply(s: String): TypeName = s
    extension (s: TypeName) {
      def toString: String = s
    }
  }

  // --------------------------------------------------------------------------------
  // OriginName : name fo the orgin
  opaque type OriginName = String
  object OriginName {
    def apply(s: String): OriginName = s
    extension (s: OriginName) {
      def toString: String = s
    }
  }

  // --------------------------------------------------------------------------------
  // TableName
  opaque type TableName = String
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
  object PgLsn {
    def apply(l: Long): PgLsn = l
    extension (l: PgLsn) {
      def value: Long = l
      def format: String = s"${(l >>> 32).toHexString.toUpperCase}/${l.toInt.toHexString.toUpperCase}"
    }
  }

}

// Model
// --------------------------------------------------------------------------------
sealed trait PgOutputMessage {
  def tag: Char

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

    // todo
    case StreamStart(xid, segment) => super.toString
    case StreamStop() => "StreamStop"
    case StreamCommit(xid, flag, commitLsn, endLsn, commitTs) => super.toString
    case StreamAbort(xid, subxid, abortInParallel) => super.toString

    case X_LogicalMsg(xid, msg) => super.toString
    case X_Relation(xid, relation) => super.toString
    case X_Insert(xid, insert) => super.toString
    case X_Update(xid, update) => super.toString
    case X_Delete(xid, delete) => super.toString
    case X_Truncate(xid, truncate) => super.toString
    case X_TypeMsg(xid, typeMessage) => super.toString

  // todo
  // BeginPrepare
  // Prepare
  // CommitPrepared
  // RollbackPrepared
  // StreamPrepare
}

object PgOutputMessage {

  /** {{{
   *when application call pg_logical_emit_message, contents directly written to WAL.
   *Used for "user-defined-meta-data" or "event-signal" insert to WAL
   *such as application-check-point, audit-tracking-log, system-synchronize
   * }}}
   */
  case class LogicalMsg(transactional: Boolean, lsn: PgLsn, prefix: Prefix, content: ByteVector) extends PgOutputMessage { val tag = 'M' }

  case class Begin(finalLsn: PgLsn, commitTsMicros: PgTimestamp, xid: PgXid) extends PgOutputMessage { val tag = 'B' }
  case class Commit(flags: Int, commitLsn: PgLsn, endLsn: PgLsn, commitTsMicros: PgTimestamp) extends PgOutputMessage { val tag = 'C' }

  case class RelationColumn(isKey: Boolean, name: ColumnName, typeOid: TypeOid, atpmod: AtpMod) {
    override def toString: String = s"Column(flags: $isKey, name: $name, typeOid: $typeOid, atttypmod: $atpmod)"
  }
  case class Relation(relId: RelId, namespace: Namespace, name: TableName, replicaIdentity: ReplicaIdentity, columns: Vector[RelationColumn]) extends PgOutputMessage { val tag = 'R' }

  case class TypeMsg(typeOid: TypeOid, schemaName: Namespace, typeName: TypeName) extends PgOutputMessage { val tag = 'Y' }
  case class Insert(relId: RelId, neo: TupleData) extends PgOutputMessage { val tag = 'I' }
  case class Update(relId: RelId, oldOrKey: Option[TupleData], newRow: TupleData) extends PgOutputMessage { val tag = 'U' }
  case class Delete(relId: RelId, oldOrKey: Option[TupleData]) extends PgOutputMessage { val tag = 'D' }
  case class Truncate(relIds: Vector[RelId], options: TruncateOptions) extends PgOutputMessage { val tag = 'T' }

  /**
   * for BDR(Bi-Directional Replcation): used to prevent loop update.
   */
  case class Origin(lsn: PgLsn, name: OriginName) extends PgOutputMessage { val tag = 'O' }

  // Stream Messages
  // --------------------------------------------------------------------------------
  case class StreamStart(xid: PgXid, segment: Segment) extends PgOutputMessage { val tag = 'S' }
  case class StreamStop() extends PgOutputMessage { val tag = 'E' }
  case class StreamCommit(xid: PgXid, flag: CommitFlag, commitLsn: PgLsn, endLsn: PgLsn, commitTs: PgTimestamp) extends PgOutputMessage { val tag = 'c' }

  //
  // abortLsn & abortTs : present only when streaming is set to parallel; since version 4
  case class AbortInParallel(abortLsn: PgLsn, abortTs: PgTimestamp)
  case class StreamAbort(xid: PgXid, subxid: PgXid, abortInParallel: Option[AbortInParallel]) extends PgOutputMessage { val tag = 'A' }

  // Streamed Message -------------------------------------
  // only present for streamed transactions; since protocol version 2.
  case class X_LogicalMsg(xid: PgXid, msg: LogicalMsg) extends PgOutputMessage { val tag = 'M' }
  case class X_Relation(xid: PgXid, relation: Relation) extends PgOutputMessage { val tag = 'R' }
  case class X_Insert(xid: PgXid, insert: Insert) extends PgOutputMessage { val tag = 'I' }
  case class X_Update(xid: PgXid, update: Update) extends PgOutputMessage { val tag = 'U' }
  case class X_Delete(xid: PgXid, delete: Delete) extends PgOutputMessage { val tag = 'D' }
  case class X_Truncate(xid: PgXid, truncate: Truncate) extends PgOutputMessage { val tag = 'T' }
  case class X_TypeMsg(xid: PgXid, typeMessage: TypeMsg) extends PgOutputMessage { val tag = 'Y' }


  // -- tuple (row)
  case class TupleData(values: Vector[ColumnValue]) {
    override def toString: String = values.mkString("TupleData( values: [", ", ", "])")
  }

  // todo :: v18
  // https://www.postgresql.org/docs/18/protocol-logicalrep-message-formats.html
  // BeginPrepare
  // Prepare
  // CommitPrepared
  // RollbackPrepared
  // StreamPrepare
  //

  sealed trait ColumnValue
  object ColumnValue {
    case object Null extends ColumnValue
    case object UnchangedToast extends ColumnValue
    case class Text(value: String) extends ColumnValue
    case class Binary(bytes: ByteVector) extends ColumnValue
  }
}

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

// Common Codecs
// --------------------------------------------------------------------------------
object PgCodecs {

  private val emptyStreamStop = StreamStop()

  // --------------------------------------------------------------------------------
  private val cstring: Codec[String] = new Codec[String] {
    def sizeBound = SizeBound.unknown
    def encode(value: String) = Attempt.successful(BitVector(value.getBytes("UTF-8")) ++ BitVector.lowByte)
    def decode(bits: BitVector) = {
      val bv = bits.toByteVector
      val idx = bv.indexOfSlice(ByteVector(0.toByte))     // todo :: take while instead.
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
  // opaque types
  private val segment: Codec[Segment] = int8.asInstanceOf[Codec[Segment]]
  private val commitFlag: Codec[CommitFlag] = int8.asInstanceOf[Codec[CommitFlag]]

  private val xid: Codec[PgXid] = int32.asInstanceOf[Codec[PgXid]]
  private val typeOid: Codec[TypeOid] = int32.asInstanceOf[Codec[TypeOid]]
  private val atpMod: Codec[AtpMod] = int32.asInstanceOf[Codec[AtpMod]]
  private val relId: Codec[RelId] = int32.asInstanceOf[Codec[RelId]]

  private val prefix: Codec[Prefix] = cstring.asInstanceOf[Codec[Prefix]]
  private val columnName: Codec[ColumnName] = cstring.asInstanceOf[Codec[ColumnName]]
  private val namespace: Codec[Namespace] = cstring.asInstanceOf[Codec[Namespace]]
  private val typeName: Codec[TypeName] = cstring.asInstanceOf[Codec[TypeName]]
  private val originName: Codec[OriginName] = cstring.asInstanceOf[Codec[OriginName]]
  private val tableName: Codec[TableName] = cstring.asInstanceOf[Codec[TableName]]

  private val ts: Codec[PgTimestamp] = int64.asInstanceOf[Codec[PgTimestamp]]
  private val lsn: Codec[PgLsn] = int64.asInstanceOf[Codec[PgLsn]]

  // --------------------------------------------------------------------------------
  private val replicaIdentity: Codec[ReplicaIdentity] = byte.xmap( ReplicaIdentity.fromByte, _.code )
  private val truncateOptions: Codec[TruncateOptions] = uint8.xmap(TruncateOptions.apply, _.raw)

  // message codec
  // --------------------------------------------------------------------------------
  private val logicalMsgCodec: Codec[LogicalMsg] =
    ( bool(8) :: lsn :: prefix :: variableSizeBytes(int32, bytes) ).as[LogicalMsg] // Content (Length + Data)

  private val beginCodec: Codec[Begin] =
    (lsn :: ts :: xid).as[Begin]

  private val commitCodec: Codec[Commit] =
    (uint8 :: lsn :: lsn :: ts).as[Commit]

  private val relationColumnCodec: Codec[RelationColumn] =
    ( bool(8):: columnName :: typeOid :: atpMod ).as[RelationColumn]

  private val relationCodec: Codec[Relation] =
    (relId :: namespace :: tableName :: replicaIdentity :: vectorOfN(uint16, relationColumnCodec)).as[Relation]

  private val typeMsgCodec: Codec[TypeMsg] =
    (typeOid :: namespace :: typeName).as[TypeMsg]

  // --------------------------------------------------------------------------------

  // ColumnValue: [Tag: 1byte] + [Length: Int32 (optional)] + [Data]
  private val columnValue: Codec[ColumnValue] = byte.flatMap {
    case 'n' => println("null"); provide(ColumnValue.Null)
    case 'u' => println("unchangedToast"); provide(ColumnValue.UnchangedToast) // todo
    case 't' => println("text"); int32.flatMap(len => bytes(len).map(bv => ColumnValue.Text(bv.decodeUtf8.getOrElse(""))))
    case 'b' => println("binary"); int32.flatMap(len => bytes(len).map(ColumnValue.Binary.apply))
    case o   => println(s"unknown: $o"); fail(Err(s"Unknown column value tag: ${o.toChar}"))
  }

  private val tupleData: Codec[TupleData] = vectorOfN(int16, columnValue).as[TupleData]

  private val insertCodec: Codec[Insert] =
    (relId :: constant(BitVector('N'.toByte)) ~> tupleData).as[Insert]

  private val updateCodec: Codec[Update] =
    relId.flatMap(rId =>
      byte.flatMap(tag =>
        (tag match {
          case 'K' | 'O' => (tupleData <~ constant(BitVector('N'.toByte))).flatMap(old => tupleData.map(newR => (Some(old), newR)))
          case 'N' => tupleData.map(newR => (None, newR))
          case _ => fail(Err(s"Expected K, O or N in Update, but got ${tag.toChar}"))
        }).map(res => Update(rId, res._1, res._2))
      ) )

  private val deleteCodec: Codec[Delete] =
    relId.flatMap(rId =>
      byte.flatMap(tag =>
        (tag match {
          case 'K' | 'O' => tupleData.map(Some(_))
          case _ => fail(Err(s"Expected K or O in Delete, but got ${tag.toChar}"))
        }).map(old => Delete(rId, old))
      ) )

  // --------------------------------------------------------------------------------
  private val truncateCodec: Codec[Truncate] =
    (int32.flatMap(n => vectorOfN(provide(n), relId)) :: truncateOptions).as[Truncate]

  private val originCodec =
    (lsn :: originName).as[Origin]

  private val streamStartCodec: Codec[StreamStart] =
    (xid :: segment).as[StreamStart]

  private val streamStopCodec: Codec[StreamStop] =
    provide(emptyStreamStop)

  private val streamCommitCodec: Codec[StreamCommit] =
    (xid :: commitFlag :: lsn :: lsn :: ts).as[StreamCommit]

  private val abortInParallel: Codec[AbortInParallel] = (lsn :: ts).as[AbortInParallel]
  private val streamAbortCodec: Codec[StreamAbort] =
    (xid :: xid :: optional(provide(true), abortInParallel)).as[StreamAbort]

// todo ::::::::::::::::::::::::::::::::::::::::::: check doc.
// https://github.com/ywchae1209/DBMS_CDC/blob/master/PG18_logicalrep-message-formats.md
  val x_LogicalMsg:Codec[X_LogicalMsg] = (xid :: logicalMsgCodec).as[X_LogicalMsg]      // M
  val x_Relation  :Codec[X_Relation  ] = (xid :: relationCodec).as[X_Relation]          // R
  val x_Insert    :Codec[X_Insert    ] = (xid :: insertCodec).as[X_Insert]              // I
  val x_Update    :Codec[X_Update    ] = (xid :: updateCodec).as[X_Update]              // U
  val x_Delete    :Codec[X_Delete    ] = (xid :: deleteCodec).as[X_Delete]              // D
  val x_Truncate  :Codec[X_Truncate  ] = (xid :: truncateCodec).as[X_Truncate]          // T
  val x_TypeMsg   :Codec[X_TypeMsg   ] = (xid :: typeMsgCodec).as[X_TypeMsg]            // Y

// todo
// BeginPrepare
// Prepare
// CommitPrepared
// RollbackPrepared
// StreamPrepare

  val normalDecoders: Map[Char, Codec[PgOutputMessage]] = Map(
    'B' -> beginCodec,
    'M' -> logicalMsgCodec,
    'C' -> commitCodec,
    'R' -> relationCodec,
    'I' -> insertCodec,
    'U' -> updateCodec,
    'D' -> deleteCodec,
    'T' -> truncateCodec,
    'O' -> originCodec,
    'S' -> streamStartCodec,
    'c' -> streamCommitCodec,
    'E' -> streamStopCodec,
  )

  val streamDecoders = normalDecoders

  def getDecoder(tag: Char, isStream: Boolean): Codec[PgOutputMessage]
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

  case class DecodeTask( payload: BitVector, isStream: Boolean, decoder: Codec[PgOutputMessage] )

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
  : Attempt[DecodeResult[PgOutputMessage]] = for {
    header <- byte.decode(bits)
    tag    = header.value.toChar
    payload= header.remainder
    result <- scala.util.Try(getDecoder(tag, false).decode(payload)).toEither
      .fold(
        e => Attempt.failure(Err(s"Unknown pgoutput tag($tag): ${e.toString}")),
        r => r )
  } yield result

  def decodeAll(bits: BitVector): Either[String, Vector[PgOutputMessage]] = {
    def loop(rem: BitVector, acc: Vector[PgOutputMessage]): Attempt[Vector[PgOutputMessage]] = {
      if rem.isEmpty then
        Attempt.successful(acc)
      else
        decodeMessage(rem).flatMap(res => loop(res.remainder, acc :+ res.value))
    }
    loop(bits, Vector.empty).toEither.left.map(_.messageWithContext)
  }
}

```
