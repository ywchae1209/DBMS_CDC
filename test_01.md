# desc
## in coverage
* DDL : create/ drop/ alter/ truncate
* DML : insert/ update/ delete / in transaction( begin/commit, begin/rollback)

## Not-tested
* stream mode : version 2~ 
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

