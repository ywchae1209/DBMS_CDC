## proto code :: replication 설정을 적용하고, binary capture를 출력 

```scala
//build.sbt -- dependency 

// https://mvnrepository.com/artifact/org.postgresql/postgresql
libraryDependencies += "org.postgresql" % "postgresql" % "42.7.8"
```
```scala
// scala 3 code
package io.flux

import java.sql.Connection

@main
def main(): Unit = {

  val pg   = "jdbc:postgresql://localhost:5432"
  val db   = "testdb"
  val url  = s"$pg/$db"
  val user = "testuser"
  val pass = "testpass"
  val slot = "demo_slot1"
  val pub  = "demo_pub1"

  // set up : pg replication
  Test.setup( url, user, pass, slot, pub)

  // print binary capture
  Test.capture( url, user, pass, slot, pub)
}

object Test {

  import java.util.Properties
  import java.nio.ByteBuffer
  import java.util.concurrent.TimeUnit
  import org.postgresql.PGProperty
  import org.postgresql.jdbc.PgConnection
  import org.postgresql.replication.PGReplicationStream
  import scodec.bits.BitVector
  import java.sql.DriverManager

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
              waitMilli: Int = 100 ) = {



    val props = new Properties()
    PGProperty.USER.set(props, user)
    PGProperty.PASSWORD.set(props, pass)

    PGProperty.ASSUME_MIN_SERVER_VERSION.set(props, "14")
    PGProperty.REPLICATION.set(props, "database")

    val conn = DriverManager.getConnection(url, props).asInstanceOf[PgConnection]

    try {
      // 2. 복제 스트림 생성 (ChainedLogicalStreamBuilder)
      val stream: PGReplicationStream = conn.getReplicationAPI
        .replicationStream()
        .logical()
        .withSlotName( slot)
        .withSlotOption("proto_version", 2) // Binary 지원을 위해 2 이상
        .withSlotOption("publication_names", pub)
        .withSlotOption("binary", true) // 바이너리 출력 활성화
        .start()

      println("Stream started. Waiting for changes...")

      // 3. 메시지 루프
      while (true) {
        // Non-blocking으로 데이터 읽기
        val buffer: ByteBuffer = stream.readPending()

        if buffer == null then
          // 데이터가 없으면 Keep-alive를 위해 상태 업데이트 후 대기
          stream.forceUpdateStatus()
          TimeUnit.MILLISECONDS.sleep(waitMilli)

        else
          // todo :: 4. scodec 파서로 바이너리 데이터 해석
          val bits = BitVector(buffer)
          println(bits)
          println(bits.toHexDumpColorized)

          // todo : parse code here

          // 5. 서버에 처리 완료 보고 (LSN Feedback)
          // 이를 누락하면 서버의 WAL 로그가 삭제되지 않고 무한히 쌓임
          stream.setAppliedLSN(stream.getLastReceiveLSN)
          stream.setFlushedLSN(stream.getLastReceiveLSN)
          stream.forceUpdateStatus()
      }
    } finally {
      conn.close()
    }
  }
}
```

* output 샘플 : demo 테이블에 1개의 row를 넣었다.

```text
Note: ERROR: publication "demo_pub1" already exists
Slot 'demo_slot1' already exists.
Stream started. Waiting for changes...

BitVector(168 bits, 0x42000000000209a5e80002e9a9eb5c8d580000030f)
00000000  42 00 00 00 00 02 09 a5  e8 00 02 e9 a9 eb 5c 8d  │B......��..���\�│
00000010  58 00 00 03 0f                                    │X....│

BitVector(368 bits, 0x52000040067075626c69630064656d6f006400020169640000000017ffffffff00646174610000000019ffffffff)
00000000  52 00 00 40 06 70 75 62  6c 69 63 00 64 65 6d 6f  │R..@.public.demo│
00000010  00 64 00 02 01 69 64 00  00 00 00 17 ff ff ff ff  │.d...id.....����│
00000020  00 64 61 74 61 00 00 00  00 19 ff ff ff ff        │.data.....����│

BitVector(280 bits, 0x49000040064e000262000000040000000e620000000d74686973206973206669727374)
00000000  49 00 00 40 06 4e 00 02  62 00 00 00 04 00 00 00  │I..@.N..b.......│
00000010  0e 62 00 00 00 0d 74 68  69 73 20 69 73 20 66 69  │.b....this is fi│
00000020  72 73 74                                          │rst│

BitVector(208 bits, 0x4300000000000209a5e8000000000209a6180002e9a9eb5c8d58)
00000000  43 00 00 00 00 00 02 09  a5 e8 00 00 00 00 02 09  │C.......��......│
00000010  a6 18 00 02 e9 a9 eb 5c  8d 58                    │�...���\�X│

```


### 확인 필요한 정보

#### 1. DDL 발생시


| DDL           | capture	                | how to |
|---------------|-------------------------|----------------------------|
| ALTER TABLE	  | 'R' 메시지 재전송	            | 캐싱된 컬럼 정보(타입, 순서 등)를 즉시 갱신 |
| CREATE TABLE	  | 'R' + 'I' (첫 데이터 발생 시)	 | 신규 RelID를 맵에 등록하고 처리 시작    |
| DROP TABLE	  | 없음	(필요 시)               | 타임아웃 처리나 외부 메타데이터 확인       |
| TRUNCATE	'T'   | 메시지	(v11 이상)            | 모든 데이터를 지우는 로직 수행          | 

* create table은 이후 첫 입력시 변경 전달됨.
* drop table은 변경정보 없음

#### 2. PG14 이전버전은 binary out대신 text out을 한다고 함. :: 2개의 parser 필요할 듯.

#### 3. slot에 쌓이는 정보
> `FOR ALL TABLES` : 모든 테이블 변경사항이 포함됨.  
> primary key가 없는 테이블의 update, delete를 추가하려면  
> `ALTER TABLE table_name REPLICA IDENTITY FULL;`  -- table schema가 바뀌는 것은 아님...
 
#### 4. version check
* PG14 이전 버전 조사필요.


#### 5. binary parer : note
* ColumnValue에 있는 데이터는 RelationColumn(컬럼메타정보)를 이용해서 후처리해야 함.
* Capture데이터의 메타정보(Relation; 테이블을 Relation이라 부름)를 캐싱해서 병합처리해야 한다는 의미.
* Numeric 포맷 : 단순변환 곤란( Precision, Scala, Weight 등에 따라 달라짐)
* Timestamp 기준점 : Unix epoch (1970)이 아니라, 2000/1/1이라고 함.(오프셋보정 필요)

#### 6. PG vs Oracle -- 찾아 봐야 함(불확실)
* 클러스터 구조 ( RAC)
* 스토리지 (ASM)
* 고가용성 관리 ( Clusterware)
* 분산 query ( RAC parallel query)

1. RAC같은 기능이 있나?
> 거의 없다고 볼 수 있으나, PG-BDR 구성도 가능하다고 함.
> RAC는 일종의 Active-Active 구조, PG는 Active-Standby만 사용
> PG-BDR( Bi-Directional Replication) : https://blog.naver.com/techtrip/221853084418
> Citus(Extension), FDW(Foreign Data wrappder) : sharding ( 일종의 분산query 지원 -- 오라클 RAC parallel query)
> Patroni + HAProxy : 장애시 standby 승격 ( 일종의 고가용성 기능 -- 오라클에는 Clustware )

2. ASM 같은 기능이 있나?
> 내장은 없음
> PG 대안 : LVM(Logical Volumn Manager), ZFS/XFS, Tablespace( 테이블이나 인덱스를 다른 Disk/경로에 배치)
