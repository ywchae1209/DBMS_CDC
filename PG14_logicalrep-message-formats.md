https://www.postgresql.org/docs/14/protocol-logicalrep-message-formats.html
### graph by mermaid

--- Update ---
```mermaid
graph TD
  subgraph Update ["Update"]
    subgraph header_0 ["Header"]
      1_1_Byte1__U__["1_1_Byte1('U')"]
      1_2_Int32["1_2_Int32"]
      1_1_Byte1__U__ --> 1_2_Int32
      1_3_Int32["1_3_Int32"]
      1_2_Int32 --> 1_3_Int32
      1_4_Byte1__K__["1_4_Byte1('K')"]
      1_3_Int32 --> 1_4_Byte1__K__
      1_5_Byte1__O__["1_5_Byte1('O')"]
      1_4_Byte1__K__ --> 1_5_Byte1__O__
      1_6_TupleData["1_6_TupleData"]
      1_5_Byte1__O__ --> 1_6_TupleData
      1_7_Byte1__N__["1_7_Byte1('N')"]
      1_6_TupleData --> 1_7_Byte1__N__
      1_8_TupleData["1_8_TupleData"]
      1_7_Byte1__N__ --> 1_8_TupleData
    end
  end

    1_1_Byte1__U__ -.- desc_1_1_Byte1__U__["Identifies the message as an update message."]
    1_2_Int32 -.- desc_1_2_Int32["Xid of the transaction (only present for streamed transactions). This field is available since protocol version 2."]
    1_3_Int32 -.- desc_1_3_Int32["ID of the relation corresponding to the ID in the relation message."]
    1_4_Byte1__K__ -.- desc_1_4_Byte1__K__["Identifies the following TupleData submessage as a key. This field is optional and is only present if the update changed data in any of the column(s) that are part of the REPLICA IDENTITY index."]
    1_5_Byte1__O__ -.- desc_1_5_Byte1__O__["Identifies the following TupleData submessage as an old tuple. This field is optional and is only present if table in which the update happened has REPLICA IDENTITY set to FULL."]
    1_6_TupleData -.- desc_1_6_TupleData["TupleData message part representing the contents of the old tuple or primary key. Only present if the previous 'O' or 'K' part is present."]
    1_7_Byte1__N__ -.- desc_1_7_Byte1__N__["Identifies the following TupleData message as a new tuple."]
    1_8_TupleData -.- desc_1_8_TupleData["TupleData message part representing the contents of a new tuple."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__U__,desc_1_2_Int32,desc_1_3_Int32,desc_1_4_Byte1__K__,desc_1_5_Byte1__O__,desc_1_6_TupleData,desc_1_7_Byte1__N__,desc_1_8_TupleData comment;
```

--- Commit ---
```mermaid
graph TD
  subgraph Commit ["Commit"]
    subgraph header_0 ["Header"]
      1_1_Byte1__C__["1_1_Byte1('C')"]
      1_2_Int8["1_2_Int8"]
      1_1_Byte1__C__ --> 1_2_Int8
      1_3_Int64["1_3_Int64"]
      1_2_Int8 --> 1_3_Int64
      1_4_Int64["1_4_Int64"]
      1_3_Int64 --> 1_4_Int64
      1_5_Int64["1_5_Int64"]
      1_4_Int64 --> 1_5_Int64
    end
  end

    1_1_Byte1__C__ -.- desc_1_1_Byte1__C__["Identifies the message as a commit message."]
    1_2_Int8 -.- desc_1_2_Int8["Flags; currently unused (must be 0)."]
    1_3_Int64 -.- desc_1_3_Int64["The LSN of the commit."]
    1_4_Int64 -.- desc_1_4_Int64["The end LSN of the transaction."]
    1_5_Int64 -.- desc_1_5_Int64["Commit timestamp of the transaction. The value is in number of microseconds since PostgreSQL epoch (2000-01-01)."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__C__,desc_1_2_Int8,desc_1_3_Int64,desc_1_4_Int64,desc_1_5_Int64 comment;
```

--- Relation ---
```mermaid
graph TD
  subgraph Relation ["Relation"]
    subgraph header_0 ["Header"]
      1_1_Byte1__R__["1_1_Byte1('R')"]
      1_2_Int32["1_2_Int32"]
      1_1_Byte1__R__ --> 1_2_Int32
      1_3_Int32["1_3_Int32"]
      1_2_Int32 --> 1_3_Int32
      1_4_String["1_4_String"]
      1_3_Int32 --> 1_4_String
      1_5_String["1_5_String"]
      1_4_String --> 1_5_String
      1_6_Int8["1_6_Int8"]
      1_5_String --> 1_6_Int8
      1_7_Int16["1_7_Int16"]
      1_6_Int8 --> 1_7_Int16
    end
    subgraph repeat_1_1 ["Repeat_1"]
      2_1_Int8["2_1_Int8"]
      2_2_String["2_2_String"]
      2_1_Int8 --> 2_2_String
      2_3_Int32["2_3_Int32"]
      2_2_String --> 2_3_Int32
      2_4_Int32["2_4_Int32"]
      2_3_Int32 --> 2_4_Int32
    end
    header_0 --> repeat_1_1
  end

    1_1_Byte1__R__ -.- desc_1_1_Byte1__R__["Identifies the message as a relation message."]
    1_2_Int32 -.- desc_1_2_Int32["Xid of the transaction (only present for streamed transactions). This field is available since protocol version 2."]
    1_3_Int32 -.- desc_1_3_Int32["ID of the relation."]
    1_4_String -.- desc_1_4_String["Namespace (empty string for pg_catalog)."]
    1_5_String -.- desc_1_5_String["Relation name."]
    1_6_Int8 -.- desc_1_6_Int8["Replica identity setting for the relation (same as relreplident in pg_class)."]
    1_7_Int16 -.- desc_1_7_Int16["Number of columns."]
    2_1_Int8 -.- desc_2_1_Int8["Flags for the column. Currently can be either 0 for no flags or 1 which marks the column as part of the key."]
    2_2_String -.- desc_2_2_String["Name of the column."]
    2_3_Int32 -.- desc_2_3_Int32["ID of the column's data type."]
    2_4_Int32 -.- desc_2_4_Int32["Type modifier of the column (atttypmod)."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__R__,desc_1_2_Int32,desc_1_3_Int32,desc_1_4_String,desc_1_5_String,desc_1_6_Int8,desc_1_7_Int16,desc_2_1_Int8,desc_2_2_String,desc_2_3_Int32,desc_2_4_Int32 comment;
```

--- Delete ---
```mermaid
graph TD
  subgraph Delete ["Delete"]
    subgraph header_0 ["Header"]
      1_1_Byte1__D__["1_1_Byte1('D')"]
      1_2_Int32["1_2_Int32"]
      1_1_Byte1__D__ --> 1_2_Int32
      1_3_Int32["1_3_Int32"]
      1_2_Int32 --> 1_3_Int32
      1_4_Byte1__K__["1_4_Byte1('K')"]
      1_3_Int32 --> 1_4_Byte1__K__
      1_5_Byte1__O__["1_5_Byte1('O')"]
      1_4_Byte1__K__ --> 1_5_Byte1__O__
      1_6_TupleData["1_6_TupleData"]
      1_5_Byte1__O__ --> 1_6_TupleData
    end
  end

    1_1_Byte1__D__ -.- desc_1_1_Byte1__D__["Identifies the message as a delete message."]
    1_2_Int32 -.- desc_1_2_Int32["Xid of the transaction (only present for streamed transactions). This field is available since protocol version 2."]
    1_3_Int32 -.- desc_1_3_Int32["ID of the relation corresponding to the ID in the relation message."]
    1_4_Byte1__K__ -.- desc_1_4_Byte1__K__["Identifies the following TupleData submessage as a key. This field is present if the table in which the delete has happened uses an index as REPLICA IDENTITY."]
    1_5_Byte1__O__ -.- desc_1_5_Byte1__O__["Identifies the following TupleData message as an old tuple. This field is present if the table in which the delete happened has REPLICA IDENTITY set to FULL."]
    1_6_TupleData -.- desc_1_6_TupleData["TupleData message part representing the contents of the old tuple or primary key, depending on the previous field."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__D__,desc_1_2_Int32,desc_1_3_Int32,desc_1_4_Byte1__K__,desc_1_5_Byte1__O__,desc_1_6_TupleData comment;
```

--- Message ---
```mermaid
graph TD
  subgraph Message ["Message"]
    subgraph header_0 ["Header"]
      1_1_Byte1__M__["1_1_Byte1('M')"]
      1_2_Int32["1_2_Int32"]
      1_1_Byte1__M__ --> 1_2_Int32
      1_3_Int8["1_3_Int8"]
      1_2_Int32 --> 1_3_Int8
      1_4_Int64["1_4_Int64"]
      1_3_Int8 --> 1_4_Int64
      1_5_String["1_5_String"]
      1_4_Int64 --> 1_5_String
      1_6_Int32["1_6_Int32"]
      1_5_String --> 1_6_Int32
      1_7_Byten["1_7_Byten"]
      1_6_Int32 --> 1_7_Byten
    end
  end

    1_1_Byte1__M__ -.- desc_1_1_Byte1__M__["Identifies the message as a logical decoding message."]
    1_2_Int32 -.- desc_1_2_Int32["Xid of the transaction (only present for streamed transactions). This field is available since protocol version 2."]
    1_3_Int8 -.- desc_1_3_Int8["Flags; Either 0 for no flags or 1 if the logical decoding message is transactional."]
    1_4_Int64 -.- desc_1_4_Int64["The LSN of the logical decoding message."]
    1_5_String -.- desc_1_5_String["The prefix of the logical decoding message."]
    1_6_Int32 -.- desc_1_6_Int32["Length of the content."]
    1_7_Byten -.- desc_1_7_Byten["The content of the logical decoding message."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__M__,desc_1_2_Int32,desc_1_3_Int8,desc_1_4_Int64,desc_1_5_String,desc_1_6_Int32,desc_1_7_Byten comment;
```

--- Truncate ---
```mermaid
graph TD
  subgraph Truncate ["Truncate"]
    subgraph header_0 ["Header"]
      1_1_Byte1__T__["1_1_Byte1('T')"]
      1_2_Int32["1_2_Int32"]
      1_1_Byte1__T__ --> 1_2_Int32
      1_3_Int32["1_3_Int32"]
      1_2_Int32 --> 1_3_Int32
      1_4_Int8["1_4_Int8"]
      1_3_Int32 --> 1_4_Int8
      1_5_Int32["1_5_Int32"]
      1_4_Int8 --> 1_5_Int32
    end
  end

    1_1_Byte1__T__ -.- desc_1_1_Byte1__T__["Identifies the message as a truncate message."]
    1_2_Int32 -.- desc_1_2_Int32["Xid of the transaction (only present for streamed transactions). This field is available since protocol version 2."]
    1_3_Int32 -.- desc_1_3_Int32["Number of relations"]
    1_4_Int8 -.- desc_1_4_Int8["Option bits for TRUNCATE: 1 for CASCADE, 2 for RESTART IDENTITY"]
    1_5_Int32 -.- desc_1_5_Int32["ID of the relation corresponding to the ID in the relation message. This field is repeated for each relation."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__T__,desc_1_2_Int32,desc_1_3_Int32,desc_1_4_Int8,desc_1_5_Int32 comment;
```

--- Insert ---
```mermaid
graph TD
  subgraph Insert ["Insert"]
    subgraph header_0 ["Header"]
      1_1_Byte1__I__["1_1_Byte1('I')"]
      1_2_Int32["1_2_Int32"]
      1_1_Byte1__I__ --> 1_2_Int32
      1_3_Int32["1_3_Int32"]
      1_2_Int32 --> 1_3_Int32
      1_4_Byte1__N__["1_4_Byte1('N')"]
      1_3_Int32 --> 1_4_Byte1__N__
      1_5_TupleData["1_5_TupleData"]
      1_4_Byte1__N__ --> 1_5_TupleData
    end
  end

    1_1_Byte1__I__ -.- desc_1_1_Byte1__I__["Identifies the message as an insert message."]
    1_2_Int32 -.- desc_1_2_Int32["Xid of the transaction (only present for streamed transactions). This field is available since protocol version 2."]
    1_3_Int32 -.- desc_1_3_Int32["ID of the relation corresponding to the ID in the relation message."]
    1_4_Byte1__N__ -.- desc_1_4_Byte1__N__["Identifies the following TupleData message as a new tuple."]
    1_5_TupleData -.- desc_1_5_TupleData["TupleData message part representing the contents of new tuple."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__I__,desc_1_2_Int32,desc_1_3_Int32,desc_1_4_Byte1__N__,desc_1_5_TupleData comment;
```

--- Type ---
```mermaid
graph TD
  subgraph Type ["Type"]
    subgraph header_0 ["Header"]
      1_1_Byte1__Y__["1_1_Byte1('Y')"]
      1_2_Int32["1_2_Int32"]
      1_1_Byte1__Y__ --> 1_2_Int32
      1_3_Int32["1_3_Int32"]
      1_2_Int32 --> 1_3_Int32
      1_4_String["1_4_String"]
      1_3_Int32 --> 1_4_String
      1_5_String["1_5_String"]
      1_4_String --> 1_5_String
    end
  end

    1_1_Byte1__Y__ -.- desc_1_1_Byte1__Y__["Identifies the message as a type message."]
    1_2_Int32 -.- desc_1_2_Int32["Xid of the transaction (only present for streamed transactions). This field is available since protocol version 2."]
    1_3_Int32 -.- desc_1_3_Int32["ID of the data type."]
    1_4_String -.- desc_1_4_String["Namespace (empty string for pg_catalog)."]
    1_5_String -.- desc_1_5_String["Name of the data type."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__Y__,desc_1_2_Int32,desc_1_3_Int32,desc_1_4_String,desc_1_5_String comment;
```

--- Begin ---
```mermaid
graph TD
  subgraph Begin ["Begin"]
    subgraph header_0 ["Header"]
      1_1_Byte1__B__["1_1_Byte1('B')"]
      1_2_Int64["1_2_Int64"]
      1_1_Byte1__B__ --> 1_2_Int64
      1_3_Int64["1_3_Int64"]
      1_2_Int64 --> 1_3_Int64
      1_4_Int32["1_4_Int32"]
      1_3_Int64 --> 1_4_Int32
    end
  end

    1_1_Byte1__B__ -.- desc_1_1_Byte1__B__["Identifies the message as a begin message."]
    1_2_Int64 -.- desc_1_2_Int64["The final LSN of the transaction."]
    1_3_Int64 -.- desc_1_3_Int64["Commit timestamp of the transaction. The value is in number of microseconds since PostgreSQL epoch (2000-01-01)."]
    1_4_Int32 -.- desc_1_4_Int32["Xid of the transaction."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__B__,desc_1_2_Int64,desc_1_3_Int64,desc_1_4_Int32 comment;
```

--- Origin ---
```mermaid
graph TD
  subgraph Origin ["Origin"]
    subgraph header_0 ["Header"]
      1_1_Byte1__O__["1_1_Byte1('O')"]
      1_2_Int64["1_2_Int64"]
      1_1_Byte1__O__ --> 1_2_Int64
      1_3_String["1_3_String"]
      1_2_Int64 --> 1_3_String
    end
  end

    1_1_Byte1__O__ -.- desc_1_1_Byte1__O__["Identifies the message as an origin message."]
    1_2_Int64 -.- desc_1_2_Int64["The LSN of the commit on the origin server."]
    1_3_String -.- desc_1_3_String["Name of the origin."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__O__,desc_1_2_Int64,desc_1_3_String comment;
```

### yaml
```yamlurl: https://www.postgresql.org/docs/14/protocol-logicalrep-message-formats.html
messages:
- name: Begin
  blocks:
  - type: header
    fields:
    - field: 1_1_Byte1('B')
      description: Identifies the message as a begin message.
    - field: 1_2_Int64
      description: The final LSN of the transaction.
    - field: 1_3_Int64
      description: Commit timestamp of the transaction. The value is in number of
        microseconds since PostgreSQL epoch (2000-01-01).
    - field: 1_4_Int32
      description: Xid of the transaction.
- name: Message
  blocks:
  - type: header
    fields:
    - field: 1_1_Byte1('M')
      description: Identifies the message as a logical decoding message.
    - field: 1_2_Int32
      description: Xid of the transaction (only present for streamed transactions).
        This field is available since protocol version 2.
    - field: 1_3_Int8
      description: Flags; Either 0 for no flags or 1 if the logical decoding message
        is transactional.
    - field: 1_4_Int64
      description: The LSN of the logical decoding message.
    - field: 1_5_String
      description: The prefix of the logical decoding message.
    - field: 1_6_Int32
      description: Length of the content.
    - field: 1_7_Byten
      description: The content of the logical decoding message.
- name: Commit
  blocks:
  - type: header
    fields:
    - field: 1_1_Byte1('C')
      description: Identifies the message as a commit message.
    - field: 1_2_Int8
      description: Flags; currently unused (must be 0).
    - field: 1_3_Int64
      description: The LSN of the commit.
    - field: 1_4_Int64
      description: The end LSN of the transaction.
    - field: 1_5_Int64
      description: Commit timestamp of the transaction. The value is in number of
        microseconds since PostgreSQL epoch (2000-01-01).
- name: Origin
  blocks:
  - type: header
    fields:
    - field: 1_1_Byte1('O')
      description: Identifies the message as an origin message.
    - field: 1_2_Int64
      description: The LSN of the commit on the origin server.
    - field: 1_3_String
      description: Name of the origin.
- name: Relation
  blocks:
  - type: header
    fields:
    - field: 1_1_Byte1('R')
      description: Identifies the message as a relation message.
    - field: 1_2_Int32
      description: Xid of the transaction (only present for streamed transactions).
        This field is available since protocol version 2.
    - field: 1_3_Int32
      description: ID of the relation.
    - field: 1_4_String
      description: Namespace (empty string for pg_catalog).
    - field: 1_5_String
      description: Relation name.
    - field: 1_6_Int8
      description: Replica identity setting for the relation (same as relreplident
        in pg_class).
    - field: 1_7_Int16
      description: Number of columns.
  - type: repeat_1
    fields:
    - field: 2_1_Int8
      description: Flags for the column. Currently can be either 0 for no flags or
        1 which marks the column as part of the key.
    - field: 2_2_String
      description: Name of the column.
    - field: 2_3_Int32
      description: ID of the column's data type.
    - field: 2_4_Int32
      description: Type modifier of the column (atttypmod).
- name: Type
  blocks:
  - type: header
    fields:
    - field: 1_1_Byte1('Y')
      description: Identifies the message as a type message.
    - field: 1_2_Int32
      description: Xid of the transaction (only present for streamed transactions).
        This field is available since protocol version 2.
    - field: 1_3_Int32
      description: ID of the data type.
    - field: 1_4_String
      description: Namespace (empty string for pg_catalog).
    - field: 1_5_String
      description: Name of the data type.
- name: Insert
  blocks:
  - type: header
    fields:
    - field: 1_1_Byte1('I')
      description: Identifies the message as an insert message.
    - field: 1_2_Int32
      description: Xid of the transaction (only present for streamed transactions).
        This field is available since protocol version 2.
    - field: 1_3_Int32
      description: ID of the relation corresponding to the ID in the relation message.
    - field: 1_4_Byte1('N')
      description: Identifies the following TupleData message as a new tuple.
    - field: 1_5_TupleData
      description: TupleData message part representing the contents of new tuple.
- name: Update
  blocks:
  - type: header
    fields:
    - field: 1_1_Byte1('U')
      description: Identifies the message as an update message.
    - field: 1_2_Int32
      description: Xid of the transaction (only present for streamed transactions).
        This field is available since protocol version 2.
    - field: 1_3_Int32
      description: ID of the relation corresponding to the ID in the relation message.
    - field: 1_4_Byte1('K')
      description: Identifies the following TupleData submessage as a key. This field
        is optional and is only present if the update changed data in any of the column(s)
        that are part of the REPLICA IDENTITY index.
    - field: 1_5_Byte1('O')
      description: Identifies the following TupleData submessage as an old tuple.
        This field is optional and is only present if table in which the update happened
        has REPLICA IDENTITY set to FULL.
    - field: 1_6_TupleData
      description: TupleData message part representing the contents of the old tuple
        or primary key. Only present if the previous 'O' or 'K' part is present.
    - field: 1_7_Byte1('N')
      description: Identifies the following TupleData message as a new tuple.
    - field: 1_8_TupleData
      description: TupleData message part representing the contents of a new tuple.
- name: Delete
  blocks:
  - type: header
    fields:
    - field: 1_1_Byte1('D')
      description: Identifies the message as a delete message.
    - field: 1_2_Int32
      description: Xid of the transaction (only present for streamed transactions).
        This field is available since protocol version 2.
    - field: 1_3_Int32
      description: ID of the relation corresponding to the ID in the relation message.
    - field: 1_4_Byte1('K')
      description: Identifies the following TupleData submessage as a key. This field
        is present if the table in which the delete has happened uses an index as
        REPLICA IDENTITY.
    - field: 1_5_Byte1('O')
      description: Identifies the following TupleData message as an old tuple. This
        field is present if the table in which the delete happened has REPLICA IDENTITY
        set to FULL.
    - field: 1_6_TupleData
      description: TupleData message part representing the contents of the old tuple
        or primary key, depending on the previous field.
- name: Truncate
  blocks:
  - type: header
    fields:
    - field: 1_1_Byte1('T')
      description: Identifies the message as a truncate message.
    - field: 1_2_Int32
      description: Xid of the transaction (only present for streamed transactions).
        This field is available since protocol version 2.
    - field: 1_3_Int32
      description: Number of relations
    - field: 1_4_Int8
      description: 'Option bits for TRUNCATE: 1 for CASCADE, 2 for RESTART IDENTITY'
    - field: 1_5_Int32
      description: ID of the relation corresponding to the ID in the relation message.
        This field is repeated for each relation.

```
