연속된 캡쳐메시지를 `트랜잭션처리`를 하고 `stateful처리`(테이블/컬럼정보 반영 등)할 수 있도록
캡쳐메시지를 (1차 분류 정보를 담은) envelope에 담도록 한다.

---
# Summary
1. 대부분의 메시지들은 구간(Transaction) 내에서 등장한다.
2. 일부 메시지는 구간 밖에서 등장할 수도 있다. (독립가능 메시지: 물론 구간 내에서도 등장할 수 있음)
3. **Todo : 구간메시지(4유형)과 독립메시지(1유형)을 합친 5가지 유형의 Envelope을 정의할 예정**

* note) 간과하기 쉬운 점  
> Envelope을 만들기 위해서는 binary의 decoding이 선행되어야 함. (rapid 분류구현 불가)

---

# 구간 메시지

* 대부분의 메시지들은 구간(Transaction) 내에서 등장하여 원자성을 보장받음.

## 구간 시작과 끝

| 구간 명칭 | 시작 메시지 | 중간 단계 (Segment) | 1차 확정 (Prepare) | 최종 종료 메시지 | 비고 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **1. Normal** | Begin | (없음) | (없음) | Commit | 커밋된 트랜잭션만 스트림에 나타남 |
| **2. Stream** | Stream Start | Stream Stop ↔ Stream Start | (없음) | Stream Commit / Stream Abort | 대용량 트랜잭션 조각화 (Interleaving 가능) |
| **3. Normal 2PC** | Begin Prepare | (없음) | Prepare | Commit Prepared / Rollback Prepared | PREPARE 후 나중에 확정/취소 결정 |
| **4. Stream 2PC** | Stream Start | Stream Stop ↔ Stream Start | Stream Prepare(*) | Commit Prepared / Rollback Prepared | (*) Stream 데이터를 Prepare(2PC) 상태로 전환 |

---

## Interim 메시지 (구간 내 등장 가능 메시지)
| 구간 유형 | 시작 메시지 | 구간 내 반복 가능 메시지 (Interim/Segment) | 1차 확정 | 최종 종료 |
| :--- | :--- | :--- | :--- | :--- |
| **1. Normal** | Begin | Insert, Update, Delete, Truncate, Relation, Type, LogicalMsg, Origin | (없음) | Commit |
| **2. Stream** | Stream Start | [반복] (X_Insert, X_Update, X_Delete, X_Truncate, Relation, Type) → Stream Stop | (없음) | Stream Commit / Abort |
| **3. Normal 2PC** | Begin Prepare | Insert, Update, Delete, Truncate, Relation, Type, LogicalMsg, Origin | Prepare | Commit/Rollback Prepared |
| **4. Stream 2PC** | Stream Start | [반복] (X_Insert, X_Update, X_Delete, X_Truncate, Relation, Type) → Stream Stop | Stream Prepare | Commit/Rollback Prepared |

---

## Stream 구간의 Segment 구조
* 메시지는 Stream Abort 또는 Stream Commit 전에 Segment 단위의 `Stream Start / Stream Stop` 쌍이 여러 번 등장할 수 있다.

### [개념 흐름]
```text
StreamStart(XID 100, Segment 1)
  X_Insert(...)
StreamStop

StreamStart(XID 100, Segment 2)  <-- 데이터 없이 메타데이터만 갱신되거나 빈 조각인 경우
StreamStop

StreamStart(XID 100, Segment 3)
  X_Update(...)
StreamStop

StreamCommit(XID 100)           <-- 최종 확정 시점에 버퍼 방출
```

### [시나리오 예시: Stream Interleaving]

* 큰 트랜잭션(100) 사이에 다른 트랜잭션(200)이나 일반 트랜잭션(101)이 끼어드는 경우.

| 순서 | 메시지 종류       | 내용/설명                   | 비고                    |
|------|-------------------|-----------------------------|-------------------------|
| 1    | StreamStart(100)  | XID 100의 첫 번째 조각 시작 | XID 100 구간 시작       |
| 2    | X_Insert(100)     | 데이터 조각 1               |                         |
| 3    | StreamStop        | XID 100 일시 중단           | 버퍼 저장 후 대기       |
| 4    | Begin(101)        | 일반 트랜잭션 101 시작      | Normal 구간 끼어듦      |
| 5    | Insert(101)       | 데이터 전송                 |                         |
| 6    | Commit(101)       | 101 확정                    | 즉시 타겟 반영 가능     |
| 7    | StreamStart(200)  | XID 200의 첫 번째 조각 시작 | 다른 Stream 구간 끼어듦 |
| 8    | X_Update(200)     | 데이터 조각 1               |                         |
| 9    | StreamStop        | XID 200 일시 중단           |                         |
| 10   | StreamStart(100)  | XID 100의 두 번째 조각 재개 | XID 100 이어서 수신     |
| 11   | X_Delete(100)     | 데이터 조각 2               |                         |
| 12   | StreamCommit(100) | XID 100 최종 확정           | 100번 버퍼 전체 방출    |


* [구간 조합별 중첩(Interleaving) 가능 여부]

| 구간 조합       | 중첩(Interleaving) 가능 여부 | 비고                                                   | 비고                      |
|-----------------|------------------------------|--------------------------------------------------------|---------------------------|
| Normal - Normal | 불가능                       | Normal 트랜잭션은 직렬화되어 발송됨                    | 구간 시작                 |
| Normal - Stream | 가능                         | Stream 조각 사이에 Normal 트랜잭션이 쏙 들어올 수 있음 |                           |
| Stream - Stream | 가능                         | 서로 다른 XID의 세그먼트들이 번갈아 등장 가능          |                           |
| Stream - 2PC    | 가능                         | Prepare 단계의 메시지들이 Stream 조각 사이에 등장 가능 | 메모리에 저장만 하고 대기 |


# 독립가능 메시지

* 독립가능 메시지의 test case는 **확인 필요** ( 1/5일 현재)
 
## 1. Relation ('R')

* 역할: 테이블 구조(스키마) 정의.

* 특징: 특정 트랜잭션에 종속되지 않는 전역 캐시 정보. 스키마 변경(ALTER TABLE) 후 첫 DML 발생 시 혹은 PUBLICATION 추가 시 독립적으로 발생.


```sql
-- 1. 새로운 테이블 생성 (이 시점에는 캡처되지 않음)
CREATE TABLE public.independent_test (
    id SERIAL PRIMARY KEY,
    data TEXT
);

-- 2. Publication에 테이블 추가
-- 이 명령은 트랜잭션 밖에서 실행되며, 커넥션이 맺어져 있다면 
-- 스키마 정보를 동기화하기 위해 'Relation' 메시지가 독립적으로 전송될 수 있습니다.
ALTER PUBLICATION io_flux_pub ADD TABLE public.independent_test;

-- 3. (옵션) 테이블 컬럼 추가
-- 기존에 캐싱된 Relation 정보를 무효화하고 새로운 'Relation' 메시지를 유도합니다.
ALTER TABLE public.independent_test ADD COLUMN new_col INT;
```

## 2. Type 메시지('Y')

* 역할: 사용자 정의 데이터 타입(ENUM, Domain 등) 정보.

* 특징: pgoutput 기본 타입이 아닌 커스텀 타입 사용 시 발생. 보통 스트림 초기화 시점에 전송됨.
사용자 정의 데이터 타입 정보. 전역적으로 한 번만 전달되는 경우가 많음.

```sql
-- 1. 사용자 정의 ENUM 타입 생성
CREATE TYPE public.user_status AS ENUM ('active', 'inactive', 'pending');

-- 2. 해당 타입을 사용하는 테이블 생성 및 추가
CREATE TABLE public.user_profiles (
    id SERIAL PRIMARY KEY,
    username TEXT,
    status public.user_status
);

ALTER PUBLICATION my_pub ADD TABLE public.user_profiles;

-- 3. 타입 정보 강제 동기화 유도
-- 보통 새 세션이 연결되거나, 해당 타입을 사용하는 첫 DML이 발생하기 직전에 
-- 트랜잭션 구간 밖에서 'Type' 메시지가 독립적으로 전송됩니다.
```

## 3. Origin ('O')

* 역할: 복제 원본 서버 식별 정보.

* 특징: 다중 노드 복제 환경에서 데이터의 소스를 추적하기 위해 스트림 시작 시점에 주로 등장.
복제 원본 서버의 식별 정보. 스트림의 맨 앞이나 중간에 독립적으로 나타남.


```sql
-- 서버 A와 서버 B가 양방향 복제 중이라고 가정할 때,
-- 서버 B에서 서버 A로부터 복제된 데이터를 다시 다른 노드로 보낼 때 발생합니다.

-- 1. 복제 원본(Origin) 이름 생성
SELECT pg_replication_origin_create('node_seoul');

-- 2. 특정 세션의 Origin 설정 (관리자 레벨)
SELECT pg_replication_origin_session_setup('node_seoul');

-- 3. 이후 발생하는 데이터 스트림의 맨 앞부분에 
-- "이 데이터의 원천은 node_seoul이다"라는 정보가 구간 밖에서 전달됩니다.
```

## 4. Logical Message ('M') - Non-transactional

* 역할: 비트랜잭션 논리 메시지 전달.

* 특징: pg_logical_emit_message(false, ...) 호출 시 발생. Begin-Commit 구간과 무관하게 수신 즉시 처리해야 함.
transactional 플래그가 false인 메시지. 트랜잭션 구간 밖에서 언제든 나타나며, 수신 즉시 처리(Apply)되어야 함.

```sql
-- 트랜잭션을 시작하지 않은 상태에서 실행
-- pg_logical_emit_message(transactional, prefix, content)
SELECT pg_logical_emit_message(
    false,                         -- transactional: false (즉시 전송)
    'my_app_prefix',               -- prefix
    'External event happened!'     -- content
);
```

