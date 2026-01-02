# desc
## in coverage
* DDL : create/ drop/ alter/ truncate
* DML : insert/ update/ delete / in transaction( begin/commit, begin/rollback)
* stream mode : protocol version 2 이상 

## Not-tested
* 2PC(2 phase-commit) related version 3~

# test 
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

### Stream mode 
#### insert 1

* 사전확인사항
1. capture slot option ( Capture 소스 코드 확인할 것. )
> .withSlotOption("proto_version", 3)  
> .withSlotOption("streaming", "on")  
2. logical_decoding_work_mem 크기 줄이기 (default: 64MB)
> SET logical_decoding_work_mem = '64kB'

##### * 전역모드로 logical_decoding_work_mem 크기 줄이기
```sql
-- 시스템 전체 최소값으로 설정
ALTER SYSTEM SET logical_decoding_work_mem = '32kB'; 
SELECT pg_reload_conf();
```

* 트랜젝션 내에 row를 다량 넣어서, logical_decoding_work_mem 초과시키기
```sql
-- 트랜잭션 내에서 10,000개의 작은 행 삽입
BEGIN;

INSERT INTO test_logical_rep (t_text)
SELECT 'stream_test_line_' || g
FROM generate_series(1, 10000) g;    

-- 여기서 커밋하지 말고 캡쳐로그 확인.

COMMIT;
```
##### * Large Column 레코드를 넣어서, logical_decoding_work_mem 초과시키기

```sql
-- Toast 모드 바꾸기
-- #####  TOAST 데이터 처리방식을 바꾸얼 볼 것. 을 참고.

ALTER TABLE test_logical_rep ALTER COLUMN t_text SET STORAGE EXTERNAL;
``

```sql
-- 약 100kB 크기의 텍스트 하나를 삽입 (설정값 64kB를 즉시 초과)
INSERT INTO test_logical_rep (t_text) VALUES (repeat('A', 1024 * 100));

-- 이 시점에서 이미 'S' 메시지가 송출되어야 하나 (안 보인다면)
-- 추가로 한 행 더 입력하면 다음 스트림 블록이 생성될 수 있음 --> 안나옴.
INSERT INTO test_logical_rep (t_text) VALUES (repeat('ABCDE', 20000));
INSERT INTO test_logical_rep (t_text) VALUES (repeat('B', 1024 * 100));
```
> * PG의 자동압축 및 TOAST 포인터 처리로 인해 stream 모드 메시지로 전송되지 않는다.

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

#####  TOAST 데이터 처리방식을 바꾸얼 볼 것.
( TOAST : 아주큰 컬럼을 테이블 밖에 두는 방식 )

* Toast 저장 방식
 
| 전략     | 압축 | 외부 저장(TOAST)  |설명|
|----------|------|---------------|---|
| PLAIN    |   X  | X             |압축도 안 하고 무조건 본문에 저장 (크기 제한 있음)|
| EXTENDED |   O  | O             |(기본값) 먼저 압축해보고, 그래도 크면 TOAST로 이동|
| EXTERNAL |   X  | O             |압축은 절대 안 함. 크기가 크면 바로 TOAST로 이동|
| MAIN     |   O  | X             |최대한 압축해서 본문에 저장하려고 노력함|

* PLAIN으로 하면, 컬럼 사이즈 limit(8 KB)초과시 `row is too big`에러 발생
```sql
-- 특정 컬럼의 Toast방식 EXTERNAL로 바꾸기.
ALTER TABLE test_logical_rep ALTER COLUMN t_text SET STORAGE EXTERNAL;
```

#### insert/delete/truncate/abort in stream mode
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

