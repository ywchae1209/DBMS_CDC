* working history
----

## Mile-stone plan draft
* start : 12/22일
* spec & 타당성 조사 : ~ 12/31일(6 MD)
  > * ~PG설치 및 SQL 수작업점검~
  > * ~pgoutput 연동설정 및 점검~
  > * ~SQL 및 WAL 동작확인~

* 일정 확정 : ~ 1/12일(5 MD + 2 MD; buf ) 
  > * sample SQL Data Set 준비
  > * Binary spec 확인
  > * **구현일정 확정**
* 구현 : ~ 2/13일 ( 20 MD + 3MD ; buf)
  > * iter #1 (PG 11)
  > * iter #2 (PG 14)
  > * iter #3 (PG 16)
  > * iter #4 ( merge)

----

## 12/30

1. **todo #1** 작업 1/2
> 1. stream / normal 모드 반영한 decoder :: 100% (7 / 7)
> 2. 빠진 message 타입 추가 : Type, 2PC관련 type :: 100% ( 5/ 5)
> 3. test 방법 문서 작성 : **예정(2일 소요추정)**

==> 작업 내용 : proto_2.md에 반영
> * [[https://github.com/ywchae1209/DBMS_CDC/blob/master/proto_2.md]]

### todo #3 
1. document to code ( 2~3일 소요 추정 ) 
> 1.  PG공식 문서 : 소스코드 오류 검증
> 2.  PG공식 문서 : 코드 주석 반영 


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

