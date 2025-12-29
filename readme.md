진도 기록겸.

## 12/23
1. CDC Data capture 절차 확인 
* [[ https://github.com/ywchae1209/DBMS_CDC/blob/master/pg.md ]]

## 12/24
1. Capture 설정 및 binary capture 출력 코드 
* [[ https://github.com/ywchae1209/DBMS_CDC/blob/master/proto.md ]]

## 12/26
1. protoco & format : 아래 url이하의 문서들
* [[ https://www.postgresql.org/docs/18/protocol.html ]]

2. binary format 문서: web crawler
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

## 12/29
1. Binary Record Parser ( 예정 : ~ 1/9 : iter#1 )
* 작업중 [[ https://github.com/ywchae1209/DBMS_CDC/blob/master/iter%231.md ]]

2. spec 조사
* [[https://github.com/ywchae1209/DBMS_CDC/blob/master/pg_spec_01.md]]
> * stream 모드(long or large tx)
> * 2 phase commit (2개의 tx를 1개로 tx로 처리하기)

3. proto_02.md
> * 아래 todo #1 70% 정도 구현

### todo #1 ( layer 1)
1. stream / normal 모드 반영한 decoder
2. 빠진 message 타입 추가 : Type, 2PC관련 type

### todo #2 ( layer 2) : decoderManager
1. relation meta이용한 처리
2. decoded message stream 처리
