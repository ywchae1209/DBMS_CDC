# 설명과 사전준비

* 준비할 것 : docker 설치할 것.
* 설명
> postgresql을 docker에 설치하고  
> CDC capture를 위한 plugin인 pgoutput(Postgresql 내장)을 이용하여  
> CDC capture를 보는 절차를 설명

* 각 command는
>  `#cmd` : command 창에서 입력  
>  `#bash` : docker instance에 접속하여 bash에서 입력  
>  `-- psql` : psql (postgresql용 sql client에서 입력  
> `#...conf` : conf 파일에 편집할 내용  
 

# postgresql 준비하기 (in docker)
## 1. docker 버전 check
```cmd
#cmd

docker --version
```

## 2. 이미지 다운로드
```cmd
#cmd

docker pull postgres:14

docker pull postgres:11
docker pull postgres:latest
```
* caveat : name is `postgres` not `postgresql`


## 3. 이미지 확인
1. 컨테이너 실행
```cmd
#cmd

docker run -d --name pg14 -e POSTGRES_USER=testuser -e POSTGRES_PASSWORD=testpass -e POSTGRES_DB=testdb -p 5432:5432 postgres:14

docker run -d --name pg11 -e POSTGRES_USER=testuser -e POSTGRES_PASSWORD=testpass -e POSTGRES_DB=testdb -p 5432:5432 postgres:11
docker run -d --name pg   -e POSTGRES_USER=testuser -e POSTGRES_PASSWORD=testpass -e POSTGRES_DB=testdb -p 5432:5432 postgres:latest
```
* pg버전에 따라 pg14, pg11, pg를 사용
* 이하에서는 pg14로 설정


2. 접속확인
```cmd
#cmd

docker exec -it postgres14 psql -U testuser -d testdb
```

# postgresql replication 설정하기

## 1. pg 설정편집
* shell 접속
 ```cmd
#cmd

docker exec -it pg14 bash

docker exec -it pg11 bash
docker exec -it pg1  bash
    
```


## 2. 편집 -- 복제 설정

### 편집툴 설치 (vim) -- pg docker image 에 포함되어 있지 않을 수 있다.

```bash
#bash

apt-get update
apt-get install -y vim
```

### 편집툴 설치 (vim to Docker image) -- pg docker image 에 vim 추가하기

```cmd
#cmd 

FROM postgres:14
RUN apt-get update && apt-get install -y vim
```

* 이것저것 생각하면, 적당한 custom image 만드는 게 유리할 듯.


### 1. pg 설정 편집 및 적용(restart)

#### 1. postgresql.conf
* 편집
```bash
#bash

cd /var/lib/postgresql/data
vim postgresql.conf
```


```conf
# postgresql.conf
wal_level = logical
max_replication_slots = 4
max_wal_senders = 4
```


#### 2. pg_hba.conf
```bash
#bash

cd /var/lib/postgresql/data
vim pg_hba.conf 
```
* 개발용으로는 설정 바꾸지 않아도 무방하나.  추후를 생각하면


```conf
#pg_hba.conf
host replication testuser 127.0.0.1/32 md5
```

#### 3. restart 

변경설정 적용을 위한 restart
```cmd
#cmd

docker restart pg14
```
* /var 경로 이하는 유지된다고 함 ( 설정날라갈까 걱정 노노)


#### 4. 설정 확인

* shell 접속
 ```cmd
#cmd

docker exec -it pg14 bash

docker exec -it pg11 bash
docker exec -it pg   bash
```

```bash
#bash

psql -U testuser -d testdb
```

```psql
#psql

SHOW wal_level;
SHOW max_replication_slots;
SHOW max_wal_senders;
```

### 2. pgoutput 확인

source DB에 publication, slot설정을 하고, client에서 subscribe를 하도록 함.

* publication : CDC source table
* slot        : CDC record buffer에 해당 
  * slot에 저장되는 record많아지면, * out-of-memory 발생가능하다고 함. size 설정이 pg14(?)부터 추가되었다고 함(확인필요)
* subscribe   : CDC record를 전달받도록 지정

#### 1. source 

* shell 접속
 ```cmd
#cmd

docker exec -it pg14 bash

docker exec -it pg11 bash
docker exec -it pg   bash
```

* psql 접속
```bash
#bash

psql -U testuser -d testdb
```

* sample table 및 데이터 생성
```sql
-- psql

CREATE TABLE demo ( id SERIAL PRIMARY KEY, data TEXT );
INSERT INTO demo (data) VALUES ('hello'), ('world');
```

* publication 생성 및 확인 
```sql
-- psql
    
-- demo_pub  
CREATE PUBLICATION demo_pub FOR TABLE demo;     -- demo table만

       
-- 참고) 모든 테이블변경을 publication 설정하기
CREATE PUBLICATION demo_pub FOR ALL TABLES;
       
       
\dRp+ -- publication 확인 ( pg의 전용 커맨드)

SELECT * FROM pg_publication_tables; -- 대상테이블 목록
SELECT * FROM pg_publication;  -- 출판정보
```


* slot 생성 및 확인

```sql
-- psql

-- pgoutput 플러그인으로 생성한 CDC record를 물리적 복제슬롯인 demo_slot에 쌓도록 설정
-- demo_slot
SELECT * FROM pg_create_logical_replication_slot('demo_slot', 'pgoutput');

SELECT * FROM pg_replication_slots; -- 슬롯정보

```


* `2. client`의 설정(pg_recvlogical사용)의 경우, 다음의 설정은 필요없음
```sql
-- psql

-- start :: 
START_REPLICATION SLOT demo_slot LOGICAL 0/0 (proto_version '2', publication_names 'demo_pub', binary 'true');

```
##### capture 확인

* 아래의 `2. client`를 한 후, insert 등을 DML을 실행하고, `2. client`의 접속에서 출력되는 내용을 본다.

```sql
-- psql
  insert into demo(data) values ('this'), ('is'), ('first'), ('cdc test');
```

#### 2. client 

##### 1. 구독해서 CDC 정보 보기 

CDC record를 볼것이므로, psql접속을 하나 새로 열도록 한다.
* shell 접속
 ```cmd
#cmd

docker exec -it pg14 bash

docker exec -it pg11 bash
docker exec -it pg   bash
```

* psql 접속
```bash
#bash

psql -U testuser -d testdb
```

* 구독해서 CDC 정보 보기 
```sql

-- psql

pg_recvlogical -d "dbname=testdb" --slot demo_slot --start -f - -o proto_version=2 -o publication_names=demo_pub -o binary=true
               
```
