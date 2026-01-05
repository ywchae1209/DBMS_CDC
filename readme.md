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
  > 2. preprocessing: stream-structure/tag ( 2 - 3MD ) : **DeltaFlow_0.2.x**
  > 3. stateful processing (table/col meta caching) ( 2 -4 MD) : **DeltaFlow_0.3.x**
  > 4. serde : serialize & deserialize ( 2 - 4MD) : **DeltaFlow_0.4.x**
  > 5. network/distribute : akka ( 5 - 6 MD) : **DeltaFlow_0.5.x**
  > 6. 필수설정 ( 3 ~ 4MD) : **DeltaFlow_0.6.x**
  >    
  > * 각 단계는 **실행가능한 executable**로 (x.1.x : middle version으로 표기) 

* out-of-scope
> 1. 설정관련 처리
> 2. initial 동작 부분 : snapshot +  start capturing 어떻게..?

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

