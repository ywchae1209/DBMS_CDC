*
Postgresql CDC 캡쳐 프로그램.
( pgoutput의 binary capture를 parsing하여 출력하는 프로그램 )

* code
```
  DeltaFlow ( local git )
  revision : 14ad6170f40b44cebc6d66af1fc135af39681413
```

*
사용법

```
  java -jar DeltaFlow_0.1.1.jar [postgresUrl] [database] [user] [password] [slotName] [publicationName]

 옵션 예)
 pg = "jdbc:postgresql://localhost:5432"  -- postgresql URL
 db = "testdb"           -- capture 대상 DB
 user = "testuser"       -- user
 pass = "testpass"       -- password
 slot = "io_flux_slot"   -- 임의의 이름 주면 됨.
 pub = "io_flux_pub"     -- 임의의 이름 주면 됨.
```
*

[ 사용전 준비사항 ]

1. Postgresql 설치하기

2. Postgresql 설정
```
/var/lib/postgresql/data/postgresql.conf

# postgresql.conf
wal_level = logical     # wal 로그 기록하도록 설정
max_replication_slots = 4
max_wal_senders = 4
```

```
/var/lib/postgresql/data/pg_hba.conf

#pg_hba.conf
host replication testuser 127.0.0.1/32 md5 # 접속할 유저의 권한 설정
```

3. Postgresql restart ( 2.의 설정적용 )

4. Postgresql에 데이터 생성시키기. 

[ Sample output ]

```
--- postgresql pgoutput capture ---
i'll use : jdbc:postgresql://localhost:5432 testdb testuser testpass io_flux_slot io_flux_pub
option   : [postgresUrl] [database] [user] [password] [slotName] [publicationName]

--- postgresql setup : slot, replication for all tables ---
Note: ERROR: publication "io_flux_pub" already exists
Slot 'io_flux_slot' already exists.

--- postgresql capture start : press any key to stop ---
B       Begin(finalLsn: 0/1829B08, commitTs: 2025-12-29T07:57:19.080Z, xid: 759)
R       Relation(relId: 16433, namespace: public, name: ex1, replicaIdentity: Default, columns: [ Column(flags: true, name: idx, typeOid: 23, atttypmod: -1), Column(flags: false, name: body, typeOid: 25, atttypmod: -1)]
T       Truncate(relIds: [64], options: CASCADE, ONLY, RECURSIVE(49))
C       Commit(flags: 0, commitLsn: 0/1829B08, endLsn: 0/1829C78, commitTs: 820310239080318)
B       Begin(finalLsn: 0/1829E30, commitTs: 2025-12-29T07:57:37.109Z, xid: 760)
R       Relation(relId: 16433, namespace: public, name: ex1, replicaIdentity: Default, columns: [ Column(flags: true, name: idx, typeOid: 23, atttypmod: -1), Column(flags: false, name: body, typeOid: 25, atttypmod: -1)]
I       Insert(relId: 16433, neo: TupleData( values: [Bytes_4(0x00000005), Bytes_4(0x34343434)]))
C       Commit(flags: 0, commitLsn: 0/1829E30, endLsn: 0/1829E60, commitTs: 820310257109404)
B       Begin(finalLsn: 0/1829EE0, commitTs: 2025-12-29T07:57:44.811Z, xid: 761)
I       Insert(relId: 16433, neo: TupleData( values: [Bytes_4(0x00000006), Bytes_4(0x34343434)]))
C       Commit(flags: 0, commitLsn: 0/1829EE0, endLsn: 0/1829F10, commitTs: 820310264811032)
B       Begin(finalLsn: 0/1829F90, commitTs: 2025-12-29T07:57:46.968Z, xid: 762)
I       Insert(relId: 16433, neo: TupleData( values: [Bytes_4(0x00000007), Bytes_4(0x34343434)]))
C       Commit(flags: 0, commitLsn: 0/1829F90, endLsn: 0/1829FC0, commitTs: 820310266968036)
B       Begin(finalLsn: 0/182A058, commitTs: 2025-12-29T07:57:51.867Z, xid: 763)
I       Insert(relId: 16433, neo: TupleData( values: [Bytes_4(0x00000008), Bytes_4(0x33333333)]))
C       Commit(flags: 0, commitLsn: 0/182A058, endLsn: 0/182A088, commitTs: 820310271867428)
B       Begin(finalLsn: 0/182A180, commitTs: 2025-12-29T07:58:06.812Z, xid: 764)
I       Insert(relId: 16433, neo: TupleData( values: [Bytes_4(0x00000009), Bytes_66(0x313131313131313131313131313131313131313131313131313131313131313131313131313131313131313131313131313131313131313131313131313131313131)]))
C       Commit(flags: 0, commitLsn: 0/182A180, endLsn: 0/182A1B0, commitTs: 820310286812228)
B       Begin(finalLsn: 0/182AD48, commitTs: 2025-12-29T08:00:50.879Z, xid: 765)
... 아무키나 누르면 종료
```
