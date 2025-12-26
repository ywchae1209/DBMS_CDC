```scala
package io.flux
package capture.pgoutput
import capture.pgoutput.PgBasicTypes.*

import scodec.*
import scodec.bits.*
import scodec.codecs.*

import java.time.ZoneOffset

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
  // Column Name - 런타임에는 String
  opaque type ColumnName = String

  object ColumnName {
    def apply(v: String): ColumnName = v

    extension (cn: ColumnName) {
      def toString: String = cn
    }
  }

  // --------------------------------------------------------------------------------
  // Type OID (Object ID) - 런타임에는 Int
  opaque type TypeOid = Int

  object TypeOid {
    def apply(v: Int): TypeOid = v

    extension (oid: TypeOid) {
      def toInt: Int = oid
    }
  }

  // --------------------------------------------------------------------------------
  // Attribute Type Modifier - 런타임에는 Int
  opaque type AtpMod = Int

  object AtpMod {
    def apply(v: Int): AtpMod = v

    extension (mod: AtpMod) {
      def toInt: Int = mod
    }
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
  // Namespace (Schema Name)
  opaque type Namespace = String

  object Namespace {
    def apply(v: String): Namespace = v

    extension (ns: Namespace) {
      def toString: String = ns
    }
  }

  // --------------------------------------------------------------------------------
  // TableName
  opaque type TableName = String

  object TableName {
    def apply(v: String): TableName = v

    extension (tn: TableName) {
      def toString: String = tn
    }
  }

  // --------------------------------------------------------------------------------
  opaque type PgTimestamp = Long

  object PgTimestamp {
    def apply(value: Long): PgTimestamp = value

    extension (ts: PgTimestamp) {
      def toLong: Long = ts

      def +(other: Long): PgTimestamp = ts + other
      def -(other: PgTimestamp): Long = ts - other

      def format: String = {
        val pgEpochMs = 946684800000L
        val millis = (ts / 1000) + pgEpochMs
        java.time.Instant.ofEpochMilli(millis)
          .atOffset(java.time.ZoneOffset.UTC)
          .toString
      }

      def toUnixMillis: Long = (ts / 1000) + 946684800000L
    }
  }  // --------------------------------------------------------------------------------
  opaque type PgLsn = Long
  object PgLsn {
    def apply(l: Long): PgLsn = l

    extension (l: PgLsn) {
      def value: Long = l
      def format: String = s"${(l >>> 32).toHexString.toUpperCase}/${l.toInt.toHexString.toUpperCase}"
    }
  }

  // --------------------------------------------------------------------------------
  opaque type PgXid = Int

  object PgXid {
    def apply(i: Int): PgXid = i

    extension (i: PgXid) {
      def value: Int = i
    }
  }

}

// Model
// --------------------------------------------------------------------------------
sealed trait PgOutputMessage { def tag: Char }

case class Begin(finalLsn: PgLsn, commitTsMicros: PgTimestamp, xid: PgXid) extends PgOutputMessage {
  val tag = 'B'
  override def toString: String = s"Begin(finalLsn: ${finalLsn.format}, commitTsMicros: ${commitTsMicros.format}, xid: $xid)"
}

// when application call pg_logical_emit_message, contents directly written to WAL.
// Used for "user-defined-meta-data" or "event-signal" insert to WAL
// such as application-check-point, audit-tracking-log, system-synchronize
// todo :: xid ~ only present for streamed transactions.
case class LogicalMessage( transactional: Boolean, lsn: PgLsn, prefix: String, content: ByteVector) extends PgOutputMessage {
  val tag = 'M'
  override def toString: String = s"LogicalMessage( transactional: $transactional, lsn: ${lsn.format}, prefix: $prefix, content: $content)"
}

case class Commit(flags: Int, commitLsn: PgLsn, endLsn: PgLsn, commitTsMicros: PgTimestamp) extends PgOutputMessage {
  val tag = 'C'
  override def toString: String = s"Commit(flags: $flags, commitLsn: ${commitLsn.format}, endLsn: ${endLsn.format}, commitTsMicros: $commitTsMicros)"
}

// todo :: xid ~ only present for streamed transactions.
// see : https://www.postgresql.org/docs/14/protocol-logicalrep-message-formats.html
case class Relation(
//                   xid: Option[PgXid],
                     relId: RelId, namespace: Namespace, name: TableName, replicaIdentity: ReplicaIdentity, columns: Vector[RelationColumn]) extends PgOutputMessage {
  val tag = 'R'
  override def toString: String = s"Relation(relId: $relId, namespace: $namespace, name: $name, replicaIdentity: $replicaIdentity, columns: " +
    columns.mkString("[ ", ", ", "]")
}


case class RelationColumn(isKey: Boolean, name: ColumnName, typeOid: TypeOid, atpmod: AtpMod) {
  override def toString: String = s"Column(flags: $isKey, name: $name, typeOid: $typeOid, atttypmod: $atpmod)"
}

// todo :: xid ~ only present for streamed transactions.
// see : https://www.postgresql.org/docs/14/protocol-logicalrep-message-formats.html
// todo :: case class Type(...)

// todo :: xid ~ only present for streamed transactions.
case class Insert(relId: RelId, newRow: TupleData) extends PgOutputMessage {
  val tag = 'I'
  override def toString: String = s"Insert(relId: $relId, newRow: $newRow)"
}

// todo :: xid ~ only present for streamed transactions.
case class Update(relId: RelId, oldRowOrKey: Option[TupleData], newRow: TupleData) extends PgOutputMessage {
  val tag = 'U'
  override def toString: String = s"Update(relId: $relId, oldRowOrKey: ${oldRowOrKey.mkString}, newRow: $newRow )"
}

// todo :: xid ~ only present for streamed transactions.
case class Delete(relId: RelId, oldRowOrKey: Option[TupleData]) extends PgOutputMessage {
  val tag = 'D'
  override def toString: String = s"Delete(relId: $relId, oldRowOrKey: ${oldRowOrKey.mkString})"
}

// todo :: xid ~ only present for streamed transactions.
case class Truncate(relIds: Vector[RelId], options: TruncateOptions) extends PgOutputMessage {
  val tag = 'T'
  override def toString: String = relIds.mkString("Truncate(relIds: [", " ,", "], ")  + s"options: $options)"
}

/**
 *  for BDR(Bi-Directional Replcation): used to prevent loop update.
 */
case class Origin(lsn: PgLsn, name: String) extends PgOutputMessage {
  val tag = 'O'
  override def toString: String = s"Origin(name: $name, lsn: ${lsn.format})"
}

// Stream Messages (v16+) todo :: below
// --------------------------------------------------------------------------------
case class StreamStart(xid: PgXid, segment: Int) extends PgOutputMessage { val tag = 'S' }

case class StreamStop() extends PgOutputMessage { val tag = 'E' }
case class StreamCommit(xid: Int, flag: Int, commitLsn: PgLsn, endLsn: PgLsn, commitTsMicros: Long) extends PgOutputMessage {
  val tag = 'c'
}
case class StreamAbort(xid: Int, subxid: Int) extends PgOutputMessage { val tag = 'A' }

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
object Codecs {

  lazy val cstring: Codec[String] = new Codec[String] {
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

  val relId: Codec[RelId] = int32.asInstanceOf[Codec[RelId]]
  val lsn: Codec[PgLsn] = int64.asInstanceOf[Codec[PgLsn]]
  val xid: Codec[PgXid] = int32.asInstanceOf[Codec[PgXid]]
  val ts: Codec[PgTimestamp] = int64.asInstanceOf[Codec[PgTimestamp]]

  val columnName: Codec[ColumnName] = cstring.asInstanceOf[Codec[ColumnName]]
  val typeOid: Codec[TypeOid] = int32.asInstanceOf[Codec[TypeOid]]
  val atpMod: Codec[AtpMod] = int32.asInstanceOf[Codec[AtpMod]]

  val namespace: Codec[Namespace] = cstring.asInstanceOf[Codec[Namespace]]
  val tableName: Codec[TableName] = cstring.asInstanceOf[Codec[TableName]]
  val replicaIdentity: Codec[ReplicaIdentity] = byte.xmap( ReplicaIdentity.fromByte, _.code )

  val truncateOptions: Codec[TruncateOptions] = uint8.xmap(TruncateOptions.apply, _.raw)

  // --------------------------------------------------------------------------------

  // ColumnValue: [Tag: 1byte] + [Length: Int32 (optional)] + [Data]
  val columnValue: Codec[ColumnValue] = byte.flatMap {
    case 'n' =>println("null"); provide(ColumnValue.Null)
    case 'u' =>println("unchangedToast"); provide(ColumnValue.UnchangedToast)
    case 't' =>println("text"); int32.flatMap(len => bytes(len).map(bv => ColumnValue.Text(bv.decodeUtf8.getOrElse(""))))
    case 'b' =>println("binary"); int32.flatMap(len => bytes(len).map(ColumnValue.Binary.apply))
    case other => fail(Err(s"Unknown column value tag: ${other.toChar}"))
  }

  val tupleData: Codec[TupleData] = vectorOfN(int16, columnValue).as[TupleData]
}

// Message Codecs
// --------------------------------------------------------------------------------
object MessageCodecs {
  import Codecs.*

  val beginCodec: Codec[Begin] =
    (lsn :: ts :: xid).as[Begin]

  val logicalMessageCodec: Codec[LogicalMessage] =
    ( bool(8) :: lsn :: cstring ::
      variableSizeBytes(int32, bytes) // Content (Length + Data)
    ).as[LogicalMessage]

  val commitCodec: Codec[Commit] =
    (uint8 :: lsn :: lsn :: ts).as[Commit]

  val relationColumnCodec: Codec[RelationColumn] =
    ( bool(8):: columnName :: typeOid :: atpMod ).as[RelationColumn]

  val relationCodec: Codec[Relation] =
    (relId :: namespace :: tableName :: replicaIdentity :: vectorOfN(uint16, relationColumnCodec)).as[Relation]

  val insertCodec: Codec[Insert] =
    (relId :: constant(BitVector('N'.toByte)) ~> tupleData).as[Insert]

  val updateCodec: Codec[Update] =
    relId.flatMap(rId =>
      byte.flatMap(tag =>
        (tag match {
          case 'K' | 'O' => (tupleData <~ constant(BitVector('N'.toByte))).flatMap(old => tupleData.map(newR => (Some(old), newR)))
          case 'N' => tupleData.map(newR => (None, newR))
          case _ => fail(Err(s"Expected K, O or N in Update, but got ${tag.toChar}"))
        }).map(res => Update(rId, res._1, res._2))
      ) )

  val deleteCodec: Codec[Delete] =
    relId.flatMap(rId =>
      byte.flatMap(tag =>
        (tag match {
          case 'K' | 'O' => tupleData.map(Some(_))
          case _ => fail(Err(s"Expected K or O in Delete, but got ${tag.toChar}"))
        }).map(old => Delete(rId, old))
      ) )

  val truncateCodec: Codec[Truncate] =
    (int32.flatMap(n => vectorOfN(provide(n), relId)) :: truncateOptions).as[Truncate]

  val originCodec =
    (lsn :: cstring).as[Origin]

  // v16 Streaming
  val streamStartCodec: Codec[StreamStart] = (xid :: uint8).as[StreamStart]
  val streamCommitCodec: Codec[StreamCommit] = (int32 :: uint8 :: lsn :: lsn :: int64).as[StreamCommit] // flags skip logic simplified
}

// Multiplexer
// --------------------------------------------------------------------------------
object PgOutputParser {
  import MessageCodecs.*

  def decodeMessage(bits: BitVector): Attempt[DecodeResult[PgOutputMessage]] = {
    for {
      header <- byte.decode(bits)
      tag    = header.value.toChar
      payload = header.remainder
      result <- tag match {
        case 'B' => beginCodec.decode(payload)
        case 'M' => logicalMessageCodec.decode(payload)
        case 'C' => commitCodec.decode(payload)
        case 'R' => relationCodec.decode(payload)
        case 'I' => insertCodec.decode(payload)
        case 'U' => updateCodec.decode(payload)
        case 'D' => deleteCodec.decode(payload)
        case 'T' => truncateCodec.decode(payload)
        case 'O' => originCodec.decode(payload)

        ////
        case 'S' => streamStartCodec.decode(payload)
        case 'E' => provide(StreamStop()).decode(payload)
        case 'c' => streamCommitCodec.decode(payload)
        case other => Attempt.failure(Err(s"Unknown pgoutput tag: $other"))
      }
    } yield result
  }

  def decodeAll(bits: BitVector): Either[String, Vector[PgOutputMessage]] = {
    def loop(rem: BitVector, acc: Vector[PgOutputMessage]): Attempt[Vector[PgOutputMessage]] = {
      if (rem.isEmpty) Attempt.successful(acc)
      else decodeMessage(rem).flatMap(res => loop(res.remainder, acc :+ res.value))
    }
    loop(bits, Vector.empty).toEither.left.map(_.messageWithContext)
  }
}

```
