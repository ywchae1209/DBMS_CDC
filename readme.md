# working history
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
* 구현 : ~ 2/13일 -> 3/27일로 (node간 시스템 구성에 필요한 기능 추가필요로)
  > 1. ~pgoutput binary 파서~ : **DeltaFlow_0.1.x**
  > 2. ~preprocessing: stream-structure/tag~ : **DeltaFlow_0.2.x**
  > 3. stateful processing (~table/col meta caching~) ( 2 -3주 정도)  : **DeltaFlow_0.3.x**
  > 4. ~serde : serialize & deserialize ( 2 - 4MD)~ : **DeltaFlow_0.4.x**
  > 5. network/distribute : akka ( 5 - 6 MD) : **DeltaFlow_0.5.x**
  > 6. apply( stream 모드 spooler ) : custom(? 1 ~ 2Week) : **DeltaFlow_0.6.x**
  > 7. 필수설정 ( 3 ~ 4MD) : **DeltaFlow_0.7.x**
  >    
  > * 각 단계는 **실행가능한 executable**로 (x.1.x : middle version으로 표기) 

* out-of-scope
> 1. 설정관련 처리
> 2. initial 동작 부분 : snapshot +  start capturing 어떻게..?

---
# 2/5

### architecutre sketch


![스케치](https://github.com/ywchae1209/DBMS_CDC/blob/master/architecture01.png "스케치")



### Network Streaming 서비스 테스트 결과
* gRPC (통신protocol) + protobuf (직렬화/역직렬화) : OK 
* "OK~ 진행시켜~" (아마도 현실적인 최적 balance?)

#### 대상
1. size : 2.76GB (2,969,567,232 바이트)
2. Record : 30,135,623
3. 평균 메지지 Size = 98.5 바이트 ( meta-overhead 포함 : 30바이트 가량 포함)
 
#### 환경
1. Local Server-client : OS의 네트웍 stack을 경유하지만, Network overhead는 최소화
2. Server-Client : 자원 share ( Memory, CPU 등 자원사용량 중첩)

* Notebook : LG전자 2023 그램 16 16ZD90R-EX7VK
> * CPU: 인텔 13세대 코어 i7-1360P (2.2GHz)
> * GPU: NVIDIA GeForce RTX 3050 Laptop GPU (외장 그래픽)
> * RAM: 32GB LPDDR5 (온보드)
> * 저장장치: 1TB NVMe SSD

* back-pressure 적용
> * client의 요청에 따라 server의 out속도 연동
> * client 요청속도 최대화를 위해 recevied message 갯수만 세도록 함

* 순서의존적 이벤트처리
> * server-node의 streaming thread는 single-fixed-thread 적용
> * client-node의 streaming thread는 single-fixed-thread 적용 
> * 물리적 서비스 상한선 존재함 : 50 ~ 100개이내 Thread ( node의 OS 의존적 )

#### 결과 
1. CPU      : 15% 내외
2. Memory   : 150 MB 내외
3. Duration : 59 second

| Item      | #             | Unit   |
|-----------|---------------|--------|
| Size      | 2,969,567,232 | byte   |
| Count     | 30,135,623    |        |
| Mean size | 98.54009761   | byte   |
| Duration  | 59            | second |

#### Metric & 평가

##### Metric
  
| Metric | Result              |
|--------|---------------------|
| EPS    |           510,773   |
| MBPS   |              48.00  |

1. EPS는 상한치에 가깝고, ( 51만 EPS)
2. Byte-ThroughPut은 더 높아질 것으로( 최소 50 MBps 이상) 예상됨.( 극히 작은 Message)

##### 평가
1. 목표에 부합하는 것으로 판단됨
2. 자원사용 적정선

> * 통신처리의 Source/Client 사용자원 적정선 ( 15%, 200MB )
> * Chunk-base 통신이므로, 통신기능에서 사용되는 메모리는 일정 수준으로 유지될 것. (large message가 섞이더라도)
> * 네트웍 자원 적정선 : 1 Gbps(125MB/s) Network 경우, 약 38%의 Band(48MB/s) 사용

#### 제한 및 Todo

1. single-node 테스트 ( server, client node **분리테스트** )
2. Streaming 서비스 대상(target-node)의 개수를 늘릴 경우 **Cost 선형성 테스트**
3. Streaming 메시지 Mix : Large Message 포함( Chunk단위 분할 스트리밍 적용되어 있음.)
4. 기능검토 : Throttling 등 **자원 사용량 사용상한치 조절**

---
# 2/4

###  grpcurl : gRPC 테스트 툴
* tool [https://github.com/fullstorydev/grpcurl]]

* powershell (Window) command
```
// 문자열 escaping이 틀리는 경우가 많아서 기록차.

grpcurl.exe -plaintext -import-path ./src/main/protobuf -proto task_result.proto -d '{\"secret\": \"\", \"topic\": \"\"}' 127.0.0.1:8080 io.flux.postgres.proto3.v0.BlockStreamService/GetStream
```

```
// gRPC 서비스중인 내용의 목록을 요청할 수도 있음( 오호~)

grpcurl -plaintext localhost:8080 list
```


---
# 2/3

### Streaming over Network
* 간단하게 만들어서 local test : 15만~20만 EPS정도 (over HTTP)
> * In-memory 동작은 25만~30만 언더리.
> * 통신속도 올릴 방법 찾아봐야 겠음.
> * over gRPC(Http/2)와 비교해볼 예정.

* **To-Read**
[[https://pekko.apache.org/docs/pekko-grpc/current/index.html]]

```
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 3465M    0 3465M    0     0  22.2M      0 --:--:--  0:02:35 --:--:--     0
30135296
```
---
# 2/2

### ~pump CDC, Que~ : done (Infinite Read 함수)
  
### 종료신호시, PGReplicationStream read-offset처리
* 중단시 IgnoreSignal 추가 (received vs applied gap처리)
* IgnoreSignal 발생시켜야 되는 **테스트 케이스** 필요. :: 찾기가 힘드네 -> 나중에 찾는 걸로.

---
# 2/1

* ChronicleQue의 Thread Affinity (특정 Thread 고정으로 read/write해야 하는 제한)문제해결 해야 됨.
* -> 해결함
* Infinite Read 함수 손봐야 함. (내일)

---
# 1/28 ~ 1/30

## ~중간 정돈/조정~
* library 공부하면서 급하게 작성된 코드 정돈겸 refactoring

* 대상
> 1. SpeTest : code base 변경 : fs2 --> pekko
> 2. PgCodecs
> 3. PersistQue 
> 4. PersistQueSource
> 5. PersistQueSink
> 6. QueRelay


---
# 1/27

## 2. TxLog streaming over TCP
> 다양한 기능확장이 예상되므로, over HTTP로 변경할까 함.

 1. **Control plane과 Data plane**으로 역할 구분

| 구분    | 프로토콜     | 주요 기능                                            |
|---------|--------------|------------------------------------------------------|
| Control | HTTP / JSON  | 상호 인증, 큐 목록 조회, 세션 생성, 현재 오프셋 확인 |
| Data    | TCP / Binary | DataMessage 고속 스트리밍, Ack/Nack, 실시간 백프레셔 |

2. Consideration 
 - firewall NACL 등으로 일방향 connect 시나리오
 - skipTo offset : offset 저장은 client역


---
# ~ 1/26

## 1. serde format: ProtoBuf  
* 완료

## 2. TxLog streaming over TCP
* Akka 라이브러리 이용하여 proto-typing 해봄

* TCP server/client + akka-streaming 구조로 진행 예정.
> 1. 프로토타입 : akka 라이브러리 역시 learning-curve 있음.
> 2. protocol과 layer별 역할 구분 고민 깊게 해봐야 될 듯.

---
# 1/21
* serde format 변경 중.
  ~ protoBuf 작업 중 (2일 정도 소요 예상)

* 후보 format
 
| 항목           | MessagePack           | Protobuf                | FlatBuffers                   |
|----------------|-----------------------|-------------------------|-------------------------------|
| 바이트 크기    | 작음                  | **가장 작음 (압축률 최고)** | 큼 (Padding/Offset 때문)      |
| 객체 복원 속도 | 보통                  | **매우 빠름**               | 보통 (객체 복사 시 이점 상실) |
| 직접 접근 속도 | 느림 (전체 파싱 필요) | 보통 (전체 파싱 필요)   | 압도적 1위 (Zero-copy)        |
| 합타입 지원    | 쉬움 (Map/Array 형태) | 보통 (oneof 사용)       | 보통 (union 사용)             |
| 스키마 관리    | 불필요 (Schemaless)   | 필수 (IDL 정의)         | 필수 (IDL 정의)               |

* IDL정의를 할 경우
> * 형식 정의 문서작성하고 code반영하는데 시간이 걸림.
> * 대신 다른 format 변이가 유리할 것(해보진 않았으나...)
> * format 관리가 유리할 것.
  
* size overhead
> * FlatBuffers는 메시지당 200byte 정도였음. (해본 결과. 수천만건 정도는 감안해야 되니.. overhead가 크게 느껴짐.)
> * MessagePack은 30byte언더리 일듯 (추정; 안해봤음)
> * **ProtoBuf**는 40byte 언더리(테스트 해본 결과; 꽤 괜찮음)  
>   file Que에 read/write 속도테스트( 작은 많은 메시지) = 약 25만 EPS 정도 (1,000만건 ; 40초 가량, filesize는 1.3 GB 정도)

```
MessagePack은 그 나름대로 유리한 점이 있으나,
(쉽고, 형식이 자유롭다는 것 정도가 장점이지만)
제대로 처리한다고 가정하면 IDL 기반 포맷이 맞음. (뽀대도 나고) 
```

*  Note: for DDL Parser
> *  DDL 문법을 파악하는게 숙제인데... 찾아보니 python 라이브러리가 있어서 기록차.
> * [[https://github.com/xnuinside/simple-ddl-parser?tab=readme-ov-file]]

---
# 1/20

* 작업한 거
> * **Serde** :: TaskResult용 FlatBuffer 정의파일(.fbs)과 생성코드 사용한 serde 모듈 + 테스트 및 디버깅
> * **PersistQue** :: local file 큐 (Serde포함) ; 잡다한 처리 추가해야 될 듯.
> * **PersistQueSource** :: Stream Source Processer (PersistQue to message Stream)
> * **PersistQueSink** :: Stream Sink Processer ( message Stream to PersistQue)

* Note: Akka는 유료라이선스로 변경되서, Pekko (akka의 오픈소스 fork버전) 사용.

* 내일할 것
> * over-Network으로 Stream-Pipeline연결짓기.


### Todo-list structural component (구조모듈)
* 아래의 구조모듈 선택지별 비교 및 optimize
   ( 구현일정상 3단계 일부 ~ 5단계 )  
> 1. local que ( Chronicle Que + Serde )
> 2. Stream processing
> 3. N/W processing
> 4. Spooling 

---
# 1/19

* Que Size문제 (계속)  
 > FlatBuffers ( protobuf 보다 성능/압축률 좋다는 Serde format) proto coding.


---
# 1/15

* Chronicle Queue 테스트 ( Private용도의 Kafka유사)  
  --> 저장size 과다 issue(쉽게 GB가 넘어감)  
  --> 어떻게 해결하는 게 좋을 지.. 고민 중.  
  (전용 serde 코드를 짜자니..관리issue있을 것 같기도 하고.)  



---
# 1/14

* fs2(또는 Akka) library내에서 사용할 있도록 State처리 코드 구현 예정
> 1/13일 버전은 fs2/akka위에 사용 곤란하여.. ㅠ

* ~DDL 추적방법 조사~
> [[https://github.com/ywchae1209/DBMS_CDC/blob/master/pg_ddl.md]]

### 참고 
* [[https://github.com/2ndQuadrant/pglogical]] PG용 logical 복제 최초(?)로 내놓은 회사의 code 
* EnterpriseDB(2004-) :: 오픈 소스 데이터베이스 PostgreSQL을 기반으로 소프트웨어와 서비스를 제공하는 미국 회사이며 Postgres에 가장 큰 기여를 하는 회사
* 기능 및 제약사항 등 참고할 점 있을 듯.


---
# 1/13

* Stateful 처리 코드 proto
> 1. ~TxQueue용 TxState 구현 : Done~
> 2. ~TxSpool용 TxState 구현 : Done~


---
# 1/12

* 아래와 같은 2개의 모듈로 구성할까 싶음. (검토 중)
> * TxQueue (순서의존적 처리필요한 모든 메시지)
> * TxSpool (Stream Segment용)


---
# 1/9

* 구현방안 구상 

### next week ( ~ 1/16일)
> * Chronicle Queue를 이용한 stream-segment interleaving Proto-typing
> * Relation Map State 처리는 그 이후에.


## stateful processing 에서 해결해야 할 과제

### 1. Relation State Map 
* 배경
> 1. Relation은 컬럼 데이터를 해석 기준정보를 담고 있음(Table/Column Schema)
> 2. Relation을 식별정보로 RelationID를 갖고 있으나, 버전구분정보는 없음 --> **버전부여해야 함.**
> 3. Relation은 테이블 **변경이 없는 경우에도** 전송되기도 함 --> **변경여부 점검 필요**
> 4. Alter Table인 경우, 변경된 결과 Schema만 담고 있음 --> **변경내용 heuristic하게 판정 + 사용자 설정**
> 5. 1개의 트랜잭션내에서 여러번 Alter Table할 수도 있고, Rollback하면 Alter Table이력은 버려야 함.
> 6. 위와 같은 특성으로 순차처리를 해야하는 제약존재 (병렬처리 제약)

* 방안 : 일종의 State Monadic 처리하면 될 듯.
> 1. 버전정보 : 메시지 일련번호를 Relation 버전으로.
> 2. 해당 Transaction 내에서만 유효한 Local Relation State Map
> 3. Global Map 관리
> 4. 병렬처리 제약이 됨. --> 트랜젝션 내에서의 local 병행처리로.

### 2. Stream 모드의 Interleaving
* 배경
> 1. Stream모드는 한 트랜잭션을 Segment단위로 묶어서 전송함
> 2. 필요한 Segment가 모두 전송된 후, Commit/Abort가 전송됨.
> 3. Segment들 사이에 일반 트랜잭션이나 다른 트랜잭션의 Segment가 섞일 수 있음
> 4. 섞일 수 있는 스트림모드 트랜잭션의 최대 숫자는 DBMS의 동시접속 수 만큼.
> 5. Stream모드 on/off 가능한데, 처리속도 관점에서는 필수일듯
> 6. 섞인 트랜잭션들을 처리할 방안 필요.

* 필요한 기능 -- 
> 1. **Stream 세그먼트들을 pooling**한 후, Stream Abort/Commit에 따라 처리
* pooling을 in-memory와 Disk 혼합해서 처리해야 하므로, 복잡/불안정해질 수 있음
* **Chronicle Queue**(jvm base)를 Embed해서 처리 --> **proto코딩으로 점검 필요**
* 참고: RocksDB(C++)도 있으나, 지원OS제한으로 drop.

> 2. 타겟 DB에 **여러 connection**을 이용한 **semi-동시적용 시나리오**

> 3. 장애 시 replay등의 시나리오 따져봐야...
* kafka같은 메시지 브로커를 이용해서 replay 가능하도록 해야 할 듯.

### 3. 2PC 트랜잭션
  

---
# 1/8

* DeltaFlow_0.2 테스트 및 performance optimizing

### 성능 테스트
> 약 20~25 만EPS
* 1 단계의 처리: capture & tagging

> 1. 한계 - 순차처리방식의 limit-throughput 정도 의미
> 2. IO - binary 읽기 : bulk 처리 ( 128MB 또는 1024메시지 )
> 3. IO - 주 bottleneck은 `readPending`(API함수)
> 4. 처리 - binary decode는 후처리에서 하도록 함 (decoder 배정만)
> 5. 처리 - binary 복사는 view로 처리( near zero-cost-copy)
> 6. note - Timestamp는 2단계에서 해야 할 듯
> 7. note - `setAppliedLSN`, `setFlushedLSN` 호출 : notify PG to clean-up (API 함수)

---
## 1/7

* ~**구획정보를 담은 envelope 처리**~

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

