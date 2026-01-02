todo :: 2PC Messages are not tested..

# desc
## in coverage
* DDL : create/ drop/ alter/ truncate
* DML : insert/ update/ delete / in transaction( begin/commit, begin/rollback)
* stream mode : protocol version 2 이상 : insert/update/delete/ start/stop/abort/commit 
* 2PC(2 phase-commit) related version 3 ~ : beginPrepare/Prepare/commitPrepare/RollbackPrepare/StreamPrepare

## not in coverage
* Origin
* TypeMsg
* X-TypeMsg 

## PG 버전별 프로토콜과 특징
* 서버버전
```sql
show server_version_num;
```
* max_prepared_transactions
```sql
SHOW max_prepared_transactions;
-- 만약, 0이면, postgres.conf 변경후 pg restart.
```

1. 서버버전으로 프로토콜 버전 정할 수 있다.
2. 2 이상의 버전이면 Stream mode on 할 수 있다. ( .withSlotOption("streaming", "on") )
3. 3 이상의 버전이고, max_prepared_transactions가 0보다 크면 2PC 옵션 on ( .withSlotOption("two_phase", true)


| 버전   (server_version_num) | 프로토콜 버전 (proto_version) |                      주요 추가 기능                      |             비고             |   |
|:---------------------------:|:-----------------------------:|:--------------------------------------------------------:|:----------------------------:|---|
| PG 10   (≥100000)           | 1                             | 논리적 복제 최초 도입 (pgoutput), 기본 I/U/D   태그 지원 | 최초 버전                    |   |
| PG 11   (≥110000)           | 1                             | TRUNCATE 복제 지원                                       | T 태그 추가                  |   |
| PG 13   (≥130000)           | 1                             | Partitioned Table 복제 최적화                            | 파티션 루트 기준 복제 가능   |   |
| PG 14   (≥140000)           | 2                             | Binary 모드 지원, Streaming 옵션 추가                    | S, E, P, K, r 태그 추가      |   |
| PG 15   (≥150000)           | 3                             | Two-Phase Commit 지원, Column/Row Filtering 도입         | 필터링 기능 최초 추가        |   |
| PG 16   (≥160000)           | 4                             | Standby 서버에서 논리적 복제 가능, 필터링 구조 확장      | 대기 서버에서 슬롯 생성 가능 |   |
| PG 17   (≥170000)           | 4                             | 성능 최적화 및 확장 기능 유지                            | 최신 안정화 버전             |   |

* 2 PC 
```sql
SHOW max_prepared_transactions;
```

# Test
## 1. DDL

### 1) create table
```sql
CREATE TABLE test_logical_rep (
    id SERIAL PRIMARY KEY,
    t_text TEXT,
    t_int INTEGER,
    t_numeric NUMERIC,
    t_timestamp TIMESTAMP WITH TIME ZONE,
    t_boolean BOOLEAN,
    t_jsonb JSONB,
    t_uuid UUID,
    t_bytea BYTEA
);
```
* out 
> begin - commit (without interim contents)

### 2) drop table 
```sql
drop table test_logical_rep;
```
* out
> begin - commit (without interim contents)

### 3) alter table
```sql
ALTER TABLE test_logical_rep ADD COLUMN t_new_col VARCHAR(50);
```
* out
> begin - commit (without interim contents)

### 4) truncate
```sql
truncate table test_logical_rep;
```
* out
> begin - truncate - commit

## 2. dml + truncate

### inserts 
```sql
INSERT INTO test_logical_rep (t_text)        VALUES ('Hello Logical');       
INSERT INTO test_logical_rep (t_int )        VALUES (111);                   
INSERT INTO test_logical_rep (t_numeric )    VALUES (123456789.0123456789);                      
INSERT INTO test_logical_rep (t_timestamp )  VALUES ('2024-05-20 14:30:00+09');            
INSERT INTO test_logical_rep (t_boolean )    VALUES (true);                      
INSERT INTO test_logical_rep (t_jsonb )      VALUES ('{"status": "active", "count": 10}');                   
INSERT INTO test_logical_rep (t_uuid)        VALUES ('a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11');
INSERT INTO test_logical_rep (t_bytea )      VALUES ('\x0102030405060708090a0b0c0d0e0f007f80ff');
```
* out :
> 1: begin-(relation)-insert-commit
> 2: begin-insert-commit

### delete
```sql
delete from test_logical_rep;
```
* out
> begin - delete - delete... - commit


### inserts tx
```sql
begin;
INSERT INTO test_logical_rep (t_text)        VALUES ('Hello Logical');
INSERT INTO test_logical_rep (t_int )        VALUES (111);
INSERT INTO test_logical_rep (t_numeric )    VALUES (123456789.0123456789);
INSERT INTO test_logical_rep (t_timestamp )  VALUES ('2024-05-20 14:30:00+09');
INSERT INTO test_logical_rep (t_boolean )    VALUES (true);
INSERT INTO test_logical_rep (t_jsonb )      VALUES ('{"status": "active", "count": 10}');
INSERT INTO test_logical_rep (t_uuid)        VALUES ('a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11');
INSERT INTO test_logical_rep (t_bytea )      VALUES ('\x0102030405060708090a0b0c0d0e0f007f80ff');
commit;
```
* out
> begin - insert - insert... - commit

* note: duplicate insertion is allowed in this table.


### insert with rollback

```sql
begin;
INSERT INTO test_logical_rep (t_text)        VALUES ('Hello Logical');
INSERT INTO test_logical_rep (t_int )        VALUES (111);
INSERT INTO test_logical_rep (t_numeric )    VALUES (123456789.0123456789);
INSERT INTO test_logical_rep (t_timestamp )  VALUES ('2024-05-20 14:30:00+09');
INSERT INTO test_logical_rep (t_boolean )    VALUES (true);
INSERT INTO test_logical_rep (t_jsonb )      VALUES ('{"status": "active", "count": 10}');
INSERT INTO test_logical_rep (t_uuid)        VALUES ('a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11');
INSERT INTO test_logical_rep (t_bytea )      VALUES ('\x0102030405060708090a0b0c0d0e0f007f80ff');
rollback;
```
* out
> no-replication message

### some special case : to be added
```sql
-- NaN ; Not A Number : special numeric data
INSERT INTO test_logical_rep (t_numeric )    VALUES ('NaN');

-- preserve scale
INSERT INTO test_logical_rep (t_numeric )    VALUES ('0.00000');
```
* out
> begin - insert (NaN = Bytes_8(0x00000000c0000000)) - commit

```sql
-- preserve scale
INSERT INTO test_logical_rep (t_numeric )    VALUES ('0.00000');
INSERT INTO test_logical_rep (t_numeric )    VALUES ('0.00');
```

* out
> 1. begin - insert (Bytes_8(0x0000000000000005)) - commit
> 2. begin - insert (Bytes_8(0x0000000000000002)) - commit


### update

```sql
begin;
UPDATE test_logical_rep SET t_text        = 'Hello Logical' Where id = 2;
UPDATE test_logical_rep SET t_int         = 111 Where id = 3;
UPDATE test_logical_rep SET t_numeric     = 123456789.0123456789 Where id = 4;
UPDATE test_logical_rep SET t_timestamp   = '2024-05-20 14:30:00+09' Where id = 5;
UPDATE test_logical_rep SET t_boolean     = true Where id = 6;
UPDATE test_logical_rep SET t_jsonb       = '{"status": "active", "count": 10}' Where id = 7;
UPDATE test_logical_rep SET t_uuid        = 'a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11' Where id = 8;
UPDATE test_logical_rep SET t_bytea       = '\x0102030405060708090a0b0c0d0e0f007f80ff' Where id = 1;
commit;
```
* out
> begin - update - update... - commit

## 3. Stream mode 

### 사전확인사항
1. capture slot option ( Capture 소스 코드 확인할 것. )
> .withSlotOption("proto_version", 3)  
> .withSlotOption("streaming", "on")  
2. logical_decoding_work_mem 크기 줄이기 (default: 64MB)
> SET logical_decoding_work_mem = '64kB'

### insert 테스트
##### * 전역모드로 logical_decoding_work_mem 크기 줄이기
```sql
-- sql
-- 시스템 전체 최소값으로 설정
ALTER SYSTEM SET logical_decoding_work_mem = '32kB'; 
SELECT pg_reload_conf();
```

### insert 

#### 트랜젝션 내에 row를 다량 넣어서, logical_decoding_work_mem 초과시키기
```sql
-- 트랜잭션 내에서 10,000개의 작은 행 삽입
BEGIN;

INSERT INTO test_logical_rep (t_text)
SELECT 'stream_test_line_' || g
FROM generate_series(1, 10000) g;    

-- 여기서 커밋하지 말고 캡쳐로그 확인.
COMMIT;
-- 커밋 후의 캡쳐로그 확인.
```

##### * Large Column 레코드를 넣어서, logical_decoding_work_mem 초과시키기

* 주의) Toast설정으로 잘 안될 수 있음.
* 아래 `#####  TOAST 데이터 처리방식을 바꾸얼 볼 것` 을 참고

```sql
-- 약 100kB 크기의 텍스트 하나를 삽입 (설정값 64kB를 즉시 초과)
INSERT INTO test_logical_rep (t_text) VALUES (repeat('A', 1024 * 100));
INSERT INTO test_logical_rep (t_text) VALUES (repeat('ABCDE', 20000));
INSERT INTO test_logical_rep (t_text) VALUES (repeat('B', 1024 * 100));
```

#####  TOAST 데이터 처리방식을 바꾸얼 볼 것.
( TOAST : 아주큰 컬럼을 테이블 밖에 두는 방식 )

```
PostgreSQL은 약 2KB가 넘는 큰 데이터가 들어오면
성능을 위해 자동으로 압축을 하거나,
테이블 본문이 아닌 별도의 TOAST 테이블에 저장.
1.
스트림 모드는
"메모리 내에서 트랜잭션을 재구성하다가 한계에 도달했을 때" 발생.

2.
큰 컬럼의 경우, Toast 저장방식에 따라
logical_decoding_work_mem 초과가 잘 안됨.
```

* `Toast 저장 방식`
 
| 전략     | 압축 | 외부 저장(TOAST)  |설명|
|----------|------|---------------|---|
| PLAIN    |   X  | X             |압축도 안 하고 무조건 본문에 저장 (크기 제한 있음)|
| EXTENDED |   O  | O             |(기본값) 먼저 압축해보고, 그래도 크면 TOAST로 이동|
| EXTERNAL |   X  | O             |압축은 절대 안 함. 크기가 크면 바로 TOAST로 이동|
| MAIN     |   O  | X             |최대한 압축해서 본문에 저장하려고 노력함|

* 주의 `PLAIN`으로 하면, 컬럼 사이즈 limit(8 KB)초과시 `row is too big`에러 발생

* 위의 Large Column을 테스트하려면, 압축하지 않도록 아래와 같이 설정을 바꾸어 보라.
```sql
-- 특정 컬럼의 Toast방식 EXTERNAL로 바꾸기.
ALTER TABLE test_logical_rep ALTER COLUMN t_text SET STORAGE EXTERNAL;
```

### insert/delete/truncate/abort in stream mode
```sql
BEGIN;

-- 1. Stream Mode INSERT
-- 'stream_test_line_1' ~ 'stream_test_line_10000' 생성
INSERT INTO test_logical_rep (t_text)
SELECT 'stream_test_line_' || g
FROM generate_series(1, 10000) g;    

-- 2. Stream Mode UPDATE
-- 문자열에 'line_1' (즉 1, 10, 100... 등으로 시작하는 행들)이 포함된 행들을 업데이트
-- 행의 크기를 80KB 정도로 대폭 늘려 스트리밍을 강제.
UPDATE test_logical_rep 
SET t_text = t_text || '_' || repeat('update_data_', 6000)
WHERE t_text LIKE 'stream_test_line_1%';

-- 3. Stream Mode DELETE
-- 'stream_test_line_2'로 시작하는 행들을 삭제 (약 1,111개 행)
DELETE FROM test_logical_rep 
WHERE t_text LIKE 'stream_test_line_2%';

-- 4. Stream Mode TRUNCATE (실행 시 위 DML 내역이 복제 메시지에서 사라질 수 있음)
-- TRUNCATE test_logical_rep;

-- 5. 종료 제어 (둘 중 하나 선택)
-- COMMIT;   -- 'c' (Stream Commit) 메시지 발생
ROLLBACK;    -- 'A' (Stream Abort) 메시지 발생
```

## 4. 2 PC

* PG 버전 14이상이어야 함. (protocol_version >= 3)

### 사전확인사항
#### 1. `max_prepared_transactions` 설정 확인

0보다 큰값으로 설정되어 있어야 한다.
* 확인방법
```sql
SHOW max_prepared_transactions;
```

* 만약, 0이면 `postgresql.conf` 파일에 항목값을 `10` 이상으로 수정후 postgresql `restart`해야 함.  
#### 2. 복제슬롯 option "two_phase" 활성화
```sql
--sql
-- 마지막 옵션이 two-phase 활성화 여부
SELECT * FROM pg_create_logical_replication_slot('test_slot', 'pgoutput', false, true);
```

* 설정오류시 메시지
```
"option 'two_phase' is unknown": proto_version 3 미만인 경우
"two-phase commit is not supported by the slot": 슬롯생성시 two_phase를 true로 주지 않아서.
"prepared transactions are disabled":  max_prepared_transactions가 0인 경우
```

### 2PC 테스트 

```sql
-- [준비] 테이블 비우기
TRUNCATE TABLE test_logical_rep;

-- [단계 1] 트랜잭션 시작
BEGIN;

-- [단계 2] 아주 작은 데이터 1건 삽입
INSERT INTO test_logical_rep (t_text, t_int) 
VALUES ('2pc_only_test', 100);

-- [단계 3] 1단계: PREPARE (준비)
-- 여기서 'gid_test_001'은 나중에 커밋할 때 쓸 고유 식별자
PREPARE TRANSACTION 'gid_test_001';

-- 로그 확인 :  태그 'P' (Prepare) 메시지 수신

-- [단계 4] 2단계: COMMIT (최종 확정)
COMMIT PREPARED 'gid_test_001';
-- 로그 확인 :  태그 'K' (Commit Prepared) 메시지 수신


-- 또는 Rollback
-- 2단계: ROLLBACK (철회)
-- ROLLBACK PREPARED 'gid_test_abort_001';
-- 로그 확인 :  태그 'r' (Rollback Prepared) 메시지 수신
```

### 2PC + stream mode 테스트 

```sql
-- [세션 A] 2단계 커밋 테스트 시작
BEGIN;

-- 1. 스트림 모드를 유발할 만큼 큰 데이터 삽입
INSERT INTO test_logical_rep (t_text)
SELECT '2pc_test_' || g FROM generate_series(1, 5000) g;

-- 2. PREPARE 실행 (일반적인 COMMIT 대신 사용)
-- 'my_prepared_trx_1'은 전역적으로 고유한 식별자(GID)
PREPARE TRANSACTION 'my_prepared_trx_1';

-- 로그 확인 : 'P' (Prepare) 태그 메시지
-- 스트림 모드라면 S -> I -> E 이후 마지막에 'P'

-- 3. 최종 확정 (또는 ROLLBACK PREPARED)
COMMIT PREPARED 'my_prepared_trx_1';

-- 로그 확인 : 'K' (Commit Prepared) 태그 메시지
````
