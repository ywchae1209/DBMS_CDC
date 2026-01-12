

```sql
SELECT oid, typname, typlen, typcategory FROM pg_type;
```

* not-fully-listed (over 600 types)

| oid  |                typname                 | typlen | typcategory|
|------|----------------------------------------|--------|------------|
|   16 | bool                                   |      1 | B          |
|   17 | bytea                                  |     -1 | U          |
|   18 | char                                   |      1 | Z          |
|   19 | name                                   |     64 | S          |
|   20 | int8                                   |      8 | N          |
|   21 | int2                                   |      2 | N          |
|   22 | int2vector                             |     -1 | A          |
|   23 | int4                                   |      4 | N          |
|   24 | regproc                                |      4 | N          |
|   25 | text                                   |     -1 | S          |
|   26 | oid                                    |      4 | N          |
|   27 | tid                                    |      6 | U          |
|   28 | xid                                    |      4 | U          |
|   29 | cid                                    |      4 | U          |
|   30 | oidvector                              |     -1 | A          |
|   71 | pg_type                                |     -1 | C          |
|   75 | pg_attribute                           |     -1 | C          |
|   81 | pg_proc                                |     -1 | C          |
|   83 | pg_class                               |     -1 | C          |
|  114 | json                                   |     -1 | U          |
|  142 | xml                                    |     -1 | U          |
|  194 | pg_node_tree                           |     -1 | Z          |
| 3361 | pg_ndistinct                           |     -1 | Z          |
| 3402 | pg_dependencies                        |     -1 | Z          |
| 5017 | pg_mcv_list                            |     -1 | Z          |
|   32 | pg_ddl_command                         |      8 | P          |
| 5069 | xid8                                   |      8 | U          |
