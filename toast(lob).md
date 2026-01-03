# TOAST ( The Over-sized Attribute Storage Technique)

* PG에서 일정 크기보다 큰 컬럼을 저장하는 방식 (오라클의 LOB 유사)
* 결론 
> 1. 대부분 조치가능하나, 성능상 문제될 상황 있음( 확인필요)
> 2. 다운스트림 앞에 캐싱 layer 추가 필요할 가능성 높음 (추가 고민 필요, key/index없는 테이블)

* 의견 : 오라클보다는 깔끔하게 해결가능 할 듯. ( LOB, key/index없는 테이블)

## 1. 설명
1. 큰 컬럼이 생성/갱신될 때, 그 컬럼의 값이 크더라도 capture data에 포함된다.(Toast로 별도 저장되더라도)
2. Toast에 저장된 컬럼값의 변경이 없는 경우, Unchanged Toast로 전달된다.

## 2. 문제상황
1. insert : sql변환시 문제발생 여지 없음
2. update : after row - sql변환시 문제없음, before row - sql 변환시 문제발생 가능
3. delete : sql변환시 문제발생 가능

* 설명
> 1. before 값( where절)은 갱신/삭제 대상을 정하는 용도.
> 2. before에 Unchanged Toast로 전달되면 갱신/삭제 대상 지정 곤란할 수 있음.

* 참고
> 오라클의 LOB는 where절에 사용되지 않으므로 해당없음

### before 값 전달을 위한 Option: 테이블 단위로 지정가능

```
1. REPLICA IDENTITY DEFAULT

    기본값은 PRIMARY KEY입니다.
    UPDATE/DELETE 이벤트에서 oldrow는 PK 컬럼 값만 포함

    * 추가) PK는 Unchanged Toast로 전달되지 않음

2. REPLICA IDENTITY USING INDEX

    특정 유니크 인덱스를 지정할 수 있고,
    이 경우 oldrow는 해당 인덱스 컬럼 값만 포함됨.

3. REPLICA IDENTITY FULL

    oldrow 전체 컬럼. (unchanged Toast 포함될 수 있음)
    PK가 없거나, PK만으로는 충분하지 않은 경우에 사용.

4. REPLICA IDENTITY NOTHING
   oldrow 정보 안보냄.

```

### Unchanged Toast로 전달될 수 있는 PG 기본 컬럼 타입

```
text
varchar(n) (길이가 큰 경우)
bytea
json, jsonb
xml
numeric (Precision/scale값이 큰 경우)
배열 타입 (예: int[], text[])
```
## 3. 해법

* new row : sql 변환시 문제 없음

* old row 식별
    1. pk가 있는 테이블 : 문제없음 (기본설정으로)
    2. index 지정 가능한 테이블 : 문제없음 (REPLICA IDENTITY USING INDEX)
    3. 1,2가 불가능한 경우 : 조치가능 (REPLICA IDENTITY FULL) :: Debezium등 모든 상용솔루션 

    * 3의 경우, 처리가능 여부와 별개로 캡쳐 레코드의 크기, 성능 cost 등 문제발생 여지 있음.

### REPLICA IDENTITY USING INDEX 적용법
```sql
-- 1. 유니크 인덱스 생성 
CREATE UNIQUE INDEX idx_user_identity ON user_profile (first_name, last_name, birth_date);

-- 2. 해당 인덱스를 리플리케이션 식별자로 지정
ALTER TABLE user_profile REPLICA IDENTITY USING INDEX idx_user_identity;
```

### Toast 위헙 테이블 식별
```sql
SELECT 
    n.nspname AS schema_name,      -- 스키마명
    t.relname AS table_name,       -- 테이블명
    a.attname AS column_name,      -- 컬럼명
    pg_catalog.format_type(a.atttypid, a.atttypmod) AS data_type, -- 실제 데이터 타입 (예: text, varchar(255))
    
    -- [핵심 1] 저장 전략 (attstorage) 분류
    CASE a.attstorage
        WHEN 'x' THEN 'extended (압축+외부저장: TOAST )'
        WHEN 'm' THEN 'main (가급적 내부저장: 압축만 수행)'
        WHEN 'p' THEN 'plain (고정길이: TOAST 불가)'
        WHEN 'e' THEN 'external (비압축+외부저장: TOAST )'
    END AS storage_type
    
FROM pg_attribute a                 -- 컬럼 정보가 담긴 시스템 카탈로그
JOIN pg_class t ON a.attrelid = t.oid -- 테이블 정보 조인
JOIN pg_namespace n ON t.relnamespace = n.oid -- 스키마 정보 조인

WHERE a.attnum > 0                  -- 시스템 컬럼(ctid, tableoid 등) 제외, 실제 사용자 컬럼만
  AND NOT a.attisdropped            -- 삭제된 컬럼(Dropped column) 제외
  AND t.relkind = 'r'               -- 일반 테이블(Ordinary Table)만 필터링 (뷰, 인덱스 제외)
  
  -- [핵심 2] 분석 제외 스키마
  AND n.nspname NOT IN ('information_schema', 'pg_catalog') 
  
  -- [핵심 3] 물리적 분리 가능성 (Out-of-line Ability) 필터링
  -- 'x'(extended)와 'e'(external) 전략만이 데이터가 커졌을 때 
  -- 메인 행(Row)을 떠나 별도의 TOAST 테이블로 이동하며, 이 경우 CDC 로그에서 'u' 마킹으로 나타납니다.
  AND a.attstorage IN ('x', 'e');
```

* out sample

| 버전   (server_version_num) | 프로토콜 버전 (proto_version) |                      주요 추가 기능                      |             비고             |   |
|:---------------------------:|:-----------------------------:|:--------------------------------------------------------:|:----------------------------:|---|
| PG 10   (≥100000)           | 1                             | 논리적 복제 최초 도입 (pgoutput), 기본 I/U/D   태그 지원 | 최초 버전                    |   |
| PG 11   (≥110000)           | 1                             | TRUNCATE 복제 지원                                       | T 태그 추가                  |   |
| PG 13   (≥130000)           | 1                             | Partitioned Table 복제 최적화                            | 파티션 루트 기준 복제 가능   |   |
| PG 14   (≥140000)           | 2                             | Binary 모드 지원, Streaming 옵션 추가                    | S, E, P, K, r 태그 추가      |   |
| PG 15   (≥150000)           | 3                             | Two-Phase Commit 지원, Column/Row Filtering 도입         | 필터링 기능 최초 추가        |   |
| PG 16   (≥160000)           | 4                             | Standby 서버에서 논리적 복제 가능, 필터링 구조 확장      | 대기 서버에서 슬롯 생성 가능 |   |
| PG 17   (≥170000)           | 4                             | 성능 최적화 및 확장 기능 유지                            | 최신 안정화 버전             |   |


### LOB vs Toast

|       분       |                          Oracle LOB                         |                           PostgreSQL TOAST                          |
|:--------------:|:-----------------------------------------------------------:|:-------------------------------------------------------------------:|
| 대상 데이터    | 4KB 이상 문자열, 바이너리, XML 등                           | varlena 타입(text, varchar, bytea, jsonb, xml, numeric, 배열 등)    |
| 저장 방식      | 별도 LOB 세그먼트에 저장 (BLOB, CLOB, NCLOB, BFILE)         | TOAST 테이블에 분리 저장, PGLZ 압축 기본                            |
| 접근 방식      | LOB Locator를 통해 접근, DB 내부 또는 외부 파일 참조 가능   | 일반 컬럼처럼 접근, 내부적으로 TOAST 테이블에서 불러옴              |
| Undo/Flashback | Undo 세그먼트 + Flashback 기능으로 과거 시점 조회 가능      | Undo 기반 Flashback Query 지원, TOAST 블록은 unchanged toast로 표시 |
| 성능 최적화    | SecureFile LOB: 압축, 암호화, Deduplication 지원            | TOAST: 압축(PGLZ), 외부 저장, 필요 시만 분리                        |
| 제약           | WHERE 절에서 LOB 직접 사용 제한적 (주로 LENGTH, IS NULL 등) | TOAST 컬럼은 WHERE 절에서 자유롭게 사용 가능                        |
| 최대 크기      | 수 TB~수십 TB (버전에 따라 8TB~128TB)                       | 최대 1GB per field (실제 TOAST로 분리 저장)                         |


* 사용자 관점

> 1. Oracle: LOB은 특별한 API/함수(DBMS_LOB)를 통해 다루는 경우가 많음.
> 2. PostgreSQL: TOAST는 투명하게 동작, 일반 컬럼처럼 SELECT/WHERE에 사용 가능.

* 복제/CDC 관점

> 1. Oracle: LOB 변경 시 Undo/Redo 로그에 기록, Flashback으로 과거 조회 가능.
> 2. PostgreSQL: TOAST 컬럼은 변경 없으면 unchanged toast로 표시 → CDC에서 캐싱 필요할 수 있음.

* 성능/압축

> 1. Oracle: SecureFile LOB은 고급 기능(압축, 암호화, 중복 제거) 제공.
> 2. PostgreSQL: 기본 PGLZ 압축, 필요 시 다른 압축 알고리즘 확장 가능.

* note
```
Oracle LOB: 관리 기능은 풍부하지만, WHERE 조건에서 직접 활용하기 어렵고 성능 비용이 큼.
PostgreSQL TOAST: 투명성은 높지만, CDC에서 unchanged toast 문제를 반드시 처리해야 함.
마이그레이션 시 주의: Oracle LOB → PostgreSQL TOAST로 옮길 때, 애플리케이션 레벨에서 접근 방식이 달라져 코드 수정 필요.

```
