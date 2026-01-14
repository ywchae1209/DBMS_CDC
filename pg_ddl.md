**Research only, Not-Tested**

* pgoutput은 truncate를 제외한 DDL을 제공하지 않음.

## DDL을 capture하는 방법
* Trigger기반

### 1. logical Message방법: DDL 변경이 CDC 메시지에 담기도록 하는 방법 
* 가장 좋은 방법인 듯
```sql
CREATE OR REPLACE FUNCTION emit_ddl_logical_message()
RETURNS event_trigger AS $$
BEGIN
    -- 트랜잭션과 함께 전송되는(transactional = true) 논리적 메시지 발생
    -- current_query: 사용자가 입력한 sql문 그대로(대소문자도 그대로)
    -- 주석포함됨.
    -- ex: [Prefix: ddl_event] ALTER TABLE users ADD COLUMN age int; -- 사용자 나이 추가
    -- 여러개의 명령이 하나로 올 수도 있음 
    -- ex: [Prefix: ddl_event] CREATE TABLE logs (id int); CREATE INDEX idx_logs_id ON logs(id);
    PERFORM pg_logical_emit_message(true, 'my_ddl_prefix', current_query());
END;
$$ LANGUAGE plpgsql;

CREATE EVENT TRIGGER trg_emit_ddl ON ddl_command_end
EXECUTE FUNCTION emit_ddl_logical_message();
```
* 단점
> 1. event-trigger만들어야 함.
> 2. 전역 객체 변경은 capture되지 않음( 트리거는 Database단위로 작동함 )
>   * Create/Alter/Drop **Role, TableSpace, Database**은 캡쳐되지 않음.

* 장점
> 1. PG에 부담 최소화 ( 성능 overhead 거의 없음)
> 2. source DB내에 추가되는 table등의 요소 없음
> 3. 추가적 storage 부담없음
> 4. 원자성 보장(트랜잭션의 일부로 기록됨)
> 5. 구조적 구분(구조데이터와 contents 데이터)


#### 참고: Oracle은 supplemental log에 원본 DDL-sql 포함되므로 더 간단한 셈.
* Oracle은 supplemental log에 원본 DDL-sql 포함되므로 trigger없어도 같은 처리 가능

### 2. 이력Table방법: DDL 변경이력 Table을 남기는 방법

```sql
CREATE TABLE ddl_log (
    id SERIAL PRIMARY KEY,
    event_tag TEXT,          -- 명령 종류 (예: CREATE TABLE, ALTER TABLE, DROP TABLE)
    object_type TEXT,        -- 객체 타입 (예: table, index, view)
    object_identity TEXT,    -- 객체 식별 정보 (예: public.my_table)
    query TEXT,              -- 실행된 SQL 문 원문
    last_lsn pg_lsn,         -- DDL 발생 시점의 WAL 위치 (pg_waldump 연동용)
    user_name TEXT DEFAULT CURRENT_USER, -- 실행한 사용자
    client_addr INET DEFAULT inet_client_addr(), -- 실행한 클라이언트 IP
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP -- 발생 시각
);

-- 인덱스 추가 (특정 LSN이나 시간대로 빠른 검색을 위함)
CREATE INDEX idx_ddl_log_lsn ON ddl_log(last_lsn);
CREATE INDEX idx_ddl_log_created_at ON ddl_log(created_at);

-----------------------------------------------------------------------
CREATE OR REPLACE FUNCTION log_ddl_info()
RETURNS event_trigger AS $$
DECLARE
    obj record;
    current_lsn pg_lsn;
BEGIN
    -- 현재 WAL의 위치(LSN)를 가져옴
    current_lsn := pg_current_wal_lsn();

    FOR obj IN SELECT * FROM pg_event_trigger_ddl_commands()
    LOOP
        INSERT INTO ddl_log (event_tag, object_type, object_identity, query, last_lsn)
        VALUES (TG_TAG, obj.object_type, obj.object_identity, current_query(), current_lsn);
    END LOOP;
END;
$$ LANGUAGE plpgsql;

CREATE EVENT TRIGGER trg_log_ddl ON ddl_command_end
EXECUTE FUNCTION log_ddl_info();
```

* 단점
> 1. logical Message방법의 단점 유사.
> 2. source DB내에 테이블 1개 추가됨.(귀찮고, 싫어할 요소)
> 3. 이력테이블 추가된 만큼의 성능 cost
> 4. 변경시점시의 Table정보와 matching시 불일치 발생할 가능성 존재 (lsn같은 필드 필수)

* **확인할 사항 :: Oracle(AFC)도 확인필요**
> **Rollback된 트랜젝션 내에 있었던 DDL**이 기록되는 지 여부와 그때의 Message처리

* 장점
> 1. source DB내에 schema 변경이력을 기록해 둠.(source에 두는 게 맞을지는 의문)
