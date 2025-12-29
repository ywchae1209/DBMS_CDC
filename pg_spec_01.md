spec 고민사항 #1

## stream 모드

* large 또는 long tx의 경우 유용

### 확인할 사항

#### 1. 언제 stream 모드로 진입/종료 되는지 ?

* 아래의 두가지 조건이 모두 충족될 때.
1. `postgresql.conf`에 설정한 메모리 임계치 넘었을 때
> * `logical_decoding_work_mem` : 기본값 64MB
> * 하나의 Tx가 이 겂을 초과하면 디스크에 써 둠. 
> * 이 설정은 slot에 적용되는 것이 아니라, connection별로 지정되는 설정
> * 
> * slot에 적용되는 설정은 `max_slot_wal_keep_size`이 것을 넘기면 slot이 무효화 된다고 함.
> * `max_slot_wal_keep_size`은 slot이 견뎌야 하는 최소 Disk용량
> * 대규모 트랜잭션이라면 1개의 트랜잭션을 저장할 만큼의 size가 되어야 함.
> * 메모리를 쓰는 것은 아니고, pg_wql 디렉토리에 16MB크기의 파일로 저장

2. client가 streaming 모드로 접속
> * `start_replication`을 할때, `streaming`을 `on` 또는 `true`로 설정
> * `off`이면, commit되는 시점에 한꺼번에 전송(`begin` ~ `commit`)

* 일반모드와의 차이
> * 아래 3의 메시지들에 xid가 포함되어 전송됨

* summary
 
| 구분	          | 일반 트랜잭션	          |스트리밍 트랜잭션|
|--------------|-------------------|-----|
| 활성화 조건     | 기본값               |work_mem 초과 + streaming=on 옵션|
| Relation 태그  | 'R' (Xid 없음)	     |'R' (Xid 포함)|
| 메시지 묶음     | Begin ... Commit	 |Stream Start ... Stream Stop|


* test
```sql
-- 1. 메모리 임계치를 아주 낮게 설정 (테스트용)
SET logical_decoding_work_mem = '64kB';

-- 2. 큰 트랜잭션 실행
BEGIN;
INSERT INTO my_table SELECT generate_series(1, 10000);
COMMIT;
```

* 참고 `postgresql.conf`
 
아래의 설정은 restart를 필요로 하는 설정이어서 프로그램에서 변경 불가 함.
```
wal_level = logical
max_replication_slots = 4
max_wal_senders = 4

```



#### 2. stream 모드시의 각 sub-stream(세그먼트) 구성 (경계?)
https://www.postgresql.org/docs/14/logicaldecoding-streaming.html


```
stream_start_cb(...);   <-- start of first block of changes
  stream_change_cb(...);
  stream_change_cb(...);
  stream_message_cb(...);
  stream_change_cb(...);
  ...
  stream_change_cb(...);
stream_stop_cb(...);    <-- end of first block of changes

stream_start_cb(...);   <-- start of second block of changes
  stream_change_cb(...);
  stream_change_cb(...);
  stream_change_cb(...);
  ...
  stream_message_cb(...);
  stream_change_cb(...);
stream_stop_cb(...);    <-- end of second block of changes

stream_commit_cb(...);  <-- commit of the streamed transaction
```

#### 3. stream 모드에서 conditional 또는 stateful 디코딩
* 아래의 메시지들은 stream모드에서 xid 필드 추가된다고.(구성달라짐)

| Name             | desc |
|------------------|------|
| (Logical)Message |      |
| Relation         |      |
| Type             |      |
| Insert           |      |
| Update           |      |
| Delete           |      |
| Truncate         |      |


#### 4. stream 모드의 조건부 codec으로 인한 성능 제약과 해결방법 

* 제약
> * 스트림 모드의 경우, 주요 메시지들의 구성이 달라지므로,
> * 스트림 모드 진입/종료시까지 모드에 따른 처리를 해야 함.
> * 순차처리를 해야 하므로, 병행처리가 불가능 해짐.
> * ( 개별 메시지 단위로 peek를 통한 판정이 가능하나, cost 커짐)
 
* 해결방법
> 1. tag정보를 이용하여 decoder 배정하는 기능 ( in single thread)
> 2. 배정된 decoder를 이용하여 decode( 병렬 처리 여부 선택 가배정된 decoder를 이용하여 decode( 병렬 처리 여부 선택 가능)

* stream segment단위 묶음도 생각해 볼 수 있으나, 메모리 요구량 부담 생김(크지는 않겠으나)

## 2 Phase Commit (2PC)

분산 환경에서 원자성(Atomicity)을 보장하기 위해 사용.  
여러 시스템이 동시에 성공하거나 동시에 실패해야 함.

```
PostgreSQL에서 Two-Phase Commit(2PC)은 
여러 데이터베이스나 분산 시스템이 하나의 트랜잭션처럼 동작하도록 보장하는 프로토콜. 

PREPARE TRANSACTION으로 트랜잭션을 준비한 뒤, 
외부 트랜잭션 관리자가 COMMIT PREPARED 또는 ROLLBACK PREPARED를 실행하여 
최종적으로 확정하거나 취소
```

### 순서

#### 1. parepare

`PREPARE TRANSACTION '트랜잭션ID';`

각 서브 트랜잭션을 준비 상태로 저장. 
이 시점 이후에는 COMMIT이나 ROLLBACK은 불가능.


#### 2. commit | rollabck

* 외부 트랜잭션 관리자가 모든 참여 노드의 준비 상태를 확인한 뒤,

`COMMIT PREPARED '트랜잭션ID';` → 최종 커밋

`ROLLBACK PREPARED '트랜잭션ID';` → 최종 롤백

### in Postgresql

* 지원 명령어: `PREPARE TRANSACTION`, `COMMIT PREPARED`, `ROLLBACK PREPARED`

* 표준 기반: X/Open XA 표준을 따르지만, 일부 드물게 쓰이는 기능은 구현하지 않음.

* 사용 사례:

> * 여러 PostgreSQL 서버 간 분산 트랜잭션
> * PostgreSQL과 다른 DBMS/메시지 큐 등 외부 시스템 간 트랜잭션 관리

* 주의점:

> * 준비된 트랜잭션은 시스템 장애 시에도 디스크에 남아 있어 관리자가 직접 커밋/롤백해야 할 수 있음.
> * 장시간 준비 상태로 두면 리소스 잠금이 발생할 수 있어 운영상 주의 필요.

### 굳이 2PC ?

* 어플리케이션에서 2개의 트랜잭션을 예외처리를 이용해서 
* 1개의 트랜잭션처럼 처리할 수도 있을 것 같다. (이 방식이 2PC보다 단순)  

그러나, 
> * 첫번째 트랜잭션을 커밋하고, 
> * 두번째 트랜잭션을 커밋하려는 사이에 오류가 발생하면 
> * 첫번째 트랜잭션을 rollback할 수 없게 된다. (즉, 트랜잭션의 원자성 보장안된다.)
> * 또한, DB수준의 장애 복구까지 가능하다고 함. ( pg_prepared_xacts 뷰에 따라 커밋/롤백 가능)

* Note
> * DB1과 DB2는 서로 상태를 공유하지 않는다.
> * 2PC도 완전하지는 않다. ( 3PC도 마찬가지.)
> * Postgresql은 2PC만 지원한다.(대부분 그렇다고. 오라클도 마찬가지라고 함.)

### sample

#### 1. prepare
* DB1 
```sql
BEGIN;

-- DB1: 계좌에서 1000원 출금
UPDATE accounts SET balance = balance - 1000 WHERE account_id = 1;

-- 준비 상태로 저장
PREPARE TRANSACTION 'txn_global_001';


```
* DB2
```sql
BEGIN;

-- DB2: 다른 계좌에 1000원 입금
UPDATE accounts SET balance = balance + 1000 WHERE account_id = 2;

-- 준비 상태로 저장
PREPARE TRANSACTION 'txn_global_001';

```

#### commit | rollback

* commit
```sql
-- DB1
COMMIT PREPARED 'txn_global_001';

-- DB2
COMMIT PREPARED 'txn_global_001';

```

* rollback
```sql
-- DB1
ROLLBACK PREPARED 'txn_global_001';

-- DB2
ROLLBACK PREPARED 'txn_global_001';

```

#### 2PC sample in java
```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.Statement;

public class TwoPhaseCommitExample {
    public static void main(String[] args) throws Exception {
        // 두 개의 PostgreSQL DB 연결
        Connection conn1 = DriverManager.getConnection(
                "jdbc:postgresql://db1.example.com:5432/testdb", "user", "password");
        Connection conn2 = DriverManager.getConnection(
                "jdbc:postgresql://db2.example.com:5432/testdb", "user", "password");

        // 자동 커밋 끄기
        conn1.setAutoCommit(false);
        conn2.setAutoCommit(false);

        String xid = "txn_global_001"; // 트랜잭션 ID (두 DB에서 동일해야 함)

        try {
            // DB1 작업
            PreparedStatement ps1 = conn1.prepareStatement(
                    "UPDATE accounts SET balance = balance - 1000 WHERE account_id = ?");
            ps1.setInt(1, 1);
            ps1.executeUpdate();

            // DB2 작업
            PreparedStatement ps2 = conn2.prepareStatement(
                    "UPDATE accounts SET balance = balance + 1000 WHERE account_id = ?");
            ps2.setInt(1, 2);
            ps2.executeUpdate();

            // 1단계: PREPARE TRANSACTION
            Statement st1 = conn1.createStatement();
            st1.execute("PREPARE TRANSACTION '" + xid + "'");

            Statement st2 = conn2.createStatement();
            st2.execute("PREPARE TRANSACTION '" + xid + "'");

            // 모든 DB에서 PREPARE 성공 시 → 2단계 COMMIT
            Statement st1Commit = conn1.createStatement();
            st1Commit.execute("COMMIT PREPARED '" + xid + "'");

            Statement st2Commit = conn2.createStatement();
            st2Commit.execute("COMMIT PREPARED '" + xid + "'");

            System.out.println("Two-phase commit 성공!");

        } catch (Exception e) {
            System.out.println("에러 발생, 롤백 수행: " + e.getMessage());

            // 실패 시 ROLLBACK PREPARED
            // COMMIT PREPARED을 이미 했다면, rollback이 되지 않음.
            Statement st1Rollback = conn1.createStatement();
            st1Rollback.execute("ROLLBACK PREPARED '" + xid + "'");

            Statement st2Rollback = conn2.createStatement();
            st2Rollback.execute("ROLLBACK PREPARED '" + xid + "'");
        } finally {
            conn1.close();
            conn2.close();
        }
    }
}

```

