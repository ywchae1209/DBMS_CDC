* working history
----

## Mile-stone plan draft
* start : 12/22일
* spec & 타당성 조사 : ~ 12/31일(6 MD)
  > * ~PG설치 및 SQL 수작업점검~
  > * ~pgoutput 연동설정 및 점검~
  > * ~SQL 및 WAL 동작확인~

* 일정 확정 : ~ 1/12일(5 MD + 2 MD; buf ) 
  > * ~테스트방법 정리~
  > * Binary spec 확인
  > * **구현일정 확정**
* 구현 : ~ 2/13일 ( 20 MD + 3MD ; buf)
  > 1. ~pgoutput binary 파서~ : **DeltaFlow_0.1.x**
  > 2. ~preprocessing: stream-structure/tag~ : **DeltaFlow_0.2.x**
  > 3. stateful processing (table/col meta caching) ( 2 -4 MD -> 1주-2주 정도)  : **DeltaFlow_0.3.x**
  > 4. serde : serialize & deserialize ( 2 - 4MD) : **DeltaFlow_0.4.x**
  > 5. network/distribute : akka ( 5 - 6 MD) : **DeltaFlow_0.5.x**
  > 6. 필수설정 ( 3 ~ 4MD) : **DeltaFlow_0.6.x**
  >    
  > * 각 단계는 **실행가능한 executable**로 (x.1.x : middle version으로 표기) 

* out-of-scope
> 1. 설정관련 처리
> 2. initial 동작 부분 : snapshot +  start capturing 어떻게..?

---
## 1/7

* **구획정보를 담은 envelope 처리**

### Envelope mode (안)
| 모드          | 대상 메시지 (태그) | 성격                    | 후처리 액션 (Assembler)                             |
|---------------|--------------------|-------------------------|-----------------------------------------------------|
| Independent   | 'R', 'Y', 'O', 'M' | 구간 없음 (스키마/타입) | 즉시 실행 (글로벌 캐시 업데이트)                    |
| Normal        | 'B' ~ 'C', 'A'     | 일반 트랜잭션 구간      | Active Buffer에 적재 후 Commit 시 Apply             |
| Stream        | 'S' ~ 'c', 'E'     | 대용량 스트림 구간      | Stream Buffer에 조각 적재 후 Commit 시 Apply        |
| Prepare       | 'K' ~ 'P'          | 2PC 준비 구간           | Active Buffer에 적재 후 'P' 수신 시 2PC Pool로 이동 |
| PrepareFinish | 's', 'r'           | 2PC 최종 신호           | 2PC Pool에서 데이터를 꺼내 최종 확정/삭제           |


### Note for next-layer
1. Stream pool, 2pc pool 2개는 있어야 함.
> 1. Stream pool : interleaving 될 수 있으므로
> 2. 2pc pool : prepare 이후 ~ 확정(commit prepare/rollback preapre)사이에 시각차 존재
> 3. normal pool : interleave없이 연속적이므로 pooling없이 반영가능하니 pooling여부는 선택적
> 4. Indepenent와 PrepareFinish: pooling할 필요 없을 듯. 

2. Stream pool에서 2pc pool로의 이동
> Stream Prepare는 Stream 모드의 끝 정보 중 하나인데,
> 이 메시지를 만나면 해당 Stream 트랜잭션의 메시지들은 2pc pool로 이동

3. clean-up by TTL 필요할 듯.
> 1. stream pool : TTL 5 - 10분정도
> 2. 2pc pool : TTL 수 시간 정도 필요할 듯 : 1시간 = 경고, 12시간-24시간 = 삭제 이런 식?

* 2PC 의 TTL이 오래 걸리는 경우
> 1. 사람이 승인한 후, 종결되도록 한 경우
> 2. 외부 코디네이터 장애 후, 복귀까지 기다려야 할 경우.

### schema(relation 정보) 캐싱 

* 가능한 상황
> * 하나의 트랜잭션 내에서 alter 가 여러번 발생할 경우 (노멀 또는 스트림 모드)
> * 트랜잭션내에 relation 정보가 없는 경우

* 해야할 조치
> 1. XID내의 정보는 순차처리하면서 해당 XID용 schema 캐시 생성
> 2. 1의 처리가 commit되면 global 캐시 반영 + 해당 XID용 schema 캐시 정리
> 3. 1의 처리가 중단되면 해당 XID용 schema 캐시만 정리
> 4. schema 정보 포함되지 않은 트랜잭션은 global 캐시에서 schema 찾아서 적용

* XID schame 캐시
> * 하나의 트랜잭션이 스트림 모드로 여러 세그먼트로 쪼개지는 경우,
> * 세그먼트들의 데이터 변환을 병렬로 하고 싶은데....
> * 세그먼트들을 병행 처리하는 데에는 순차처리 제약이 생기니 문제네....
> * 단일 세그먼트 내에서 병렬 처리하는 식으로 해야 할 듯.

* global schema 캐시
> * Map[RelationId, TreeMap[LastUpdatedLsn, Relation]]
> * schema 찾기 = RelationID 찾고, 현재 Lsn보다 이전에 update된 최종 Relation찾기

* 테스트 sql
```sql
BEGIN;

-- [Phase 1] 기존 구조로 데이터 입력 (컬럼 9개)
INSERT INTO test_logical_rep (t_text, t_int, t_boolean)
VALUES ('Phase 1: Original', 10, true);

-- [Phase 2] 구조 변경: 새로운 컬럼 추가 (ADD)
-- 복제 스트림에 첫 번째 'R' 메시지 발생 (컬럼 10개)
ALTER TABLE test_logical_rep ADD COLUMN t_temp_marker TEXT;

-- 변경된 구조로 데이터 입력
INSERT INTO test_logical_rep (t_text, t_int, t_temp_marker)
VALUES ('Phase 2: Added Column', 20, 'Temporally added');

-- [Phase 3] 구조 원복: 추가했던 컬럼 삭제 (DROP)
-- 복제 스트림에 두 번째 'R' 메시지 발생 (컬럼 9개로 원복)
ALTER TABLE test_logical_rep DROP COLUMN t_temp_marker;

-- 다시 원복된 구조로 데이터 입력
INSERT INTO test_logical_rep (t_text, t_int, t_boolean)
VALUES ('Phase 3: Back to Original', 30, false);

COMMIT;
```
---
## 1/5

* ~메시지 구간 구획을 위한 조사~ ( stream-structure/tag 용 )   
[[https://github.com/ywchae1209/DBMS_CDC/blob/master/envelope.md]]

* bugfix  
  ~DeltaFlow_0.1.2 : Truncate decoder (공식문서 반영 및 테스트)~

* **Todo: 구획정보를 담은 envelope 처리**

----
## 1/3

* ~Large 컬럼, Key/Index없는 테이블 처리방법 조사~  
  [[https://github.com/ywchae1209/DBMS_CDC/blob/master/toast(lob).md]]

----
## 1/2

* ~parser 기능 검증~
[[https://github.com/ywchae1209/DBMS_CDC/blob/master/test_01.md]]
> * DDL : create/ drop/ alter/ truncate
> * DML : insert/ update/ delete / in transaction( begin/commit, begin/rollback)
> * stream mode : protocol version 2 이상 : insert/update/delete/ start/stop/abort/commit 
> * 2PC(2 phase-commit) related version 3 ~ : beginPrepare/Prepare/commitPrepare/RollbackPrepare/StreamPrepare
> * 기타 : Logical Message, Type Message, X-Type Message, Origin

### todo #3 
> 1.  **PG공식 문서 : 문서 검증 :: next week

----
## 12/31

 * ~Layer 1 parser 테스트 프로그램 개발~
 [[https://github.com/ywchae1209/DBMS_CDC/blob/master/proto_03_test.md]]

----

## 12/30

1. **todo #1** 작업 1/2
> 1. ~stream / normal 모드 반영한 decoder :: 100% (7 / 7)~
> 2. ~빠진 message 타입 추가 : Type, 2PC관련 type :: 100% ( 5/ 5)~
> 3. **test 방법 문서 작성 : 예정(2~3일)**

==> 작업 내용 
> * [[https://github.com/ywchae1209/DBMS_CDC/blob/master/proto_3.md]]

### todo #3 
1. document to code ( 2~3일 소요 추정 ) 
> 1.  **PG공식 문서 : 소스코드 오류 검증 및 debugging**
> 2.  ~PG공식 문서 : 코드 주석 반영~
> 3. **toString : 보기좋게**

----
## 12/29
1. Binary Record Parser ( 예정 : ~ 1/9 : iter#1 )
* 작업중 [[ https://github.com/ywchae1209/DBMS_CDC/blob/master/iter%231.md ]]

2. spec 조사
* [[https://github.com/ywchae1209/DBMS_CDC/blob/master/pg_spec_01.md]]
> * ~stream 모드(long or large tx)~
> * ~2 phase commit (2개의 tx를 1개로 tx로 처리하기)~

3. proto_02.md
> * **아래 todo #1 70% 정도 구현반영**
> * [[https://github.com/ywchae1209/DBMS_CDC/blob/master/proto_2.md]]

### todo #1 ( layer 1)
> 1. stream / normal 모드 반영한 decoder
> 2. 빠진 message 타입 추가 : Type, 2PC관련

### todo #2 ( layer 2) : decoderManager
> 1. relation meta이용한 처리
> 2. decoded message stream 처리

----
## 12/26

1. protoco & format : 아래 url이하의 문서들
* [[ https://www.postgresql.org/docs/18/protocol.html ]]

2. ~binary format 문서: web crawler~
* 시각적으로 보고 싶어서 만든 크롤러 코드
* [[https://github.com/ywchae1209/DBMS_CDC/blob/master/code_scrape.md]]
  
* 1. Binary Record 구조 (크롤링 한것) -- FE/BE 통신 
  > * 11 [[https://github.com/ywchae1209/DBMS_CDC/blob/master/formatPG11.md]]
  > * 14 [[https://github.com/ywchae1209/DBMS_CDC/blob/master/formatPG14.md]]
  > * 18 [[https://github.com/ywchae1209/DBMS_CDC/blob/master/formatPG18.md]]

* 2. Binary Record 구조 (크롤링 한것) -- Capture Data ( via PGReplicationStream)
  > * 11 [[https://github.com/ywchae1209/DBMS_CDC/blob/master/PG11_logicalrep-message-formats.md]]
  > * 14 [[https://github.com/ywchae1209/DBMS_CDC/blob/master/PG14_logicalrep-message-formats.md]]
  > * 18 [[https://github.com/ywchae1209/DBMS_CDC/blob/master/PG18_logicalrep-message-formats.md]]


----
## 12/24

* [[ https://github.com/ywchae1209/DBMS_CDC/blob/master/proto.md ]]
* ~Capture 설정 및 binary capture 출력 코드~

----
## 12/23

* [[ https://github.com/ywchae1209/DBMS_CDC/blob/master/pg.md ]] 
* ~CDC Data capture 절차 확인~

