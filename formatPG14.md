## https://www.postgresql.org/docs/14/protocol-message-formats.html


--- Terminate (F) ---
```mermaid
graph TD
  subgraph Terminate__F_ ["Terminate (F)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__X__["1_1_Byte1('X')"]
      1_2_Int32_4_["1_2_Int32(4)"]
      1_1_Byte1__X__ --> 1_2_Int32_4_
    end
  end

    1_1_Byte1__X__ -.- desc_1_1_Byte1__X__["Identifies the message as a termination."]
    1_2_Int32_4_ -.- desc_1_2_Int32_4_["Length of message contents in bytes, including self."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__X__,desc_1_2_Int32_4_ comment;
```

--- Execute (F) ---
```mermaid
graph TD
  subgraph Execute__F_ ["Execute (F)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__E__["1_1_Byte1('E')"]
      1_2_Int32["1_2_Int32"]
      1_1_Byte1__E__ --> 1_2_Int32
      1_3_String["1_3_String"]
      1_2_Int32 --> 1_3_String
      1_4_Int32["1_4_Int32"]
      1_3_String --> 1_4_Int32
    end
  end

    1_1_Byte1__E__ -.- desc_1_1_Byte1__E__["Identifies the message as an Execute command."]
    1_2_Int32 -.- desc_1_2_Int32["Length of message contents in bytes, including self."]
    1_3_String -.- desc_1_3_String["The name of the portal to execute (an empty string selects the unnamed portal)."]
    1_4_Int32 -.- desc_1_4_Int32["Maximum number of rows to return, if portal contains a query that returns rows (ignored otherwise). Zero denotes “no limit”."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__E__,desc_1_2_Int32,desc_1_3_String,desc_1_4_Int32 comment;
```

--- RowDescription (B) ---
```mermaid
graph TD
  subgraph RowDescription__B_ ["RowDescription (B)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__T__["1_1_Byte1('T')"]
      1_2_Int32["1_2_Int32"]
      1_1_Byte1__T__ --> 1_2_Int32
      1_3_Int16["1_3_Int16"]
      1_2_Int32 --> 1_3_Int16
    end
    subgraph repeat_1_1 ["Repeat_1"]
      2_1_String["2_1_String"]
      2_2_Int32["2_2_Int32"]
      2_1_String --> 2_2_Int32
      2_3_Int16["2_3_Int16"]
      2_2_Int32 --> 2_3_Int16
      2_4_Int32["2_4_Int32"]
      2_3_Int16 --> 2_4_Int32
      2_5_Int16["2_5_Int16"]
      2_4_Int32 --> 2_5_Int16
      2_6_Int32["2_6_Int32"]
      2_5_Int16 --> 2_6_Int32
      2_7_Int16["2_7_Int16"]
      2_6_Int32 --> 2_7_Int16
    end
    header_0 --> repeat_1_1
  end

    1_1_Byte1__T__ -.- desc_1_1_Byte1__T__["Identifies the message as a row description."]
    1_2_Int32 -.- desc_1_2_Int32["Length of message contents in bytes, including self."]
    1_3_Int16 -.- desc_1_3_Int16["Specifies the number of fields in a row (can be zero)."]
    2_1_String -.- desc_2_1_String["The field name."]
    2_2_Int32 -.- desc_2_2_Int32["If the field can be identified as a column of a specific table, the object ID of the table; otherwise zero."]
    2_3_Int16 -.- desc_2_3_Int16["If the field can be identified as a column of a specific table, the attribute number of the column; otherwise zero."]
    2_4_Int32 -.- desc_2_4_Int32["The object ID of the field's data type."]
    2_5_Int16 -.- desc_2_5_Int16["The data type size (see pg_type.typlen). Note that negative values denote variable-width types."]
    2_6_Int32 -.- desc_2_6_Int32["The type modifier (see pg_attribute.atttypmod). The meaning of the modifier is type-specific."]
    2_7_Int16 -.- desc_2_7_Int16["The format code being used for the field. Currently will be zero (text) or one (binary). In a RowDescription returned from the statement variant of Describe, the format code is not yet known and will always be zero."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__T__,desc_1_2_Int32,desc_1_3_Int16,desc_2_1_String,desc_2_2_Int32,desc_2_3_Int16,desc_2_4_Int32,desc_2_5_Int16,desc_2_6_Int32,desc_2_7_Int16 comment;
```

--- NotificationResponse (B) ---
```mermaid
graph TD
  subgraph NotificationResponse__B_ ["NotificationResponse (B)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__A__["1_1_Byte1('A')"]
      1_2_Int32["1_2_Int32"]
      1_1_Byte1__A__ --> 1_2_Int32
      1_3_Int32["1_3_Int32"]
      1_2_Int32 --> 1_3_Int32
      1_4_String["1_4_String"]
      1_3_Int32 --> 1_4_String
      1_5_String["1_5_String"]
      1_4_String --> 1_5_String
    end
  end

    1_1_Byte1__A__ -.- desc_1_1_Byte1__A__["Identifies the message as a notification response."]
    1_2_Int32 -.- desc_1_2_Int32["Length of message contents in bytes, including self."]
    1_3_Int32 -.- desc_1_3_Int32["The process ID of the notifying backend process."]
    1_4_String -.- desc_1_4_String["The name of the channel that the notify has been raised on."]
    1_5_String -.- desc_1_5_String["The “payload” string passed from the notifying process."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__A__,desc_1_2_Int32,desc_1_3_Int32,desc_1_4_String,desc_1_5_String comment;
```

--- FunctionCallResponse (B) ---
```mermaid
graph TD
  subgraph FunctionCallResponse__B_ ["FunctionCallResponse (B)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__V__["1_1_Byte1('V')"]
      1_2_Int32["1_2_Int32"]
      1_1_Byte1__V__ --> 1_2_Int32
      1_3_Int32["1_3_Int32"]
      1_2_Int32 --> 1_3_Int32
      1_4_Byten["1_4_Byten"]
      1_3_Int32 --> 1_4_Byten
    end
  end

    1_1_Byte1__V__ -.- desc_1_1_Byte1__V__["Identifies the message as a function call result."]
    1_2_Int32 -.- desc_1_2_Int32["Length of message contents in bytes, including self."]
    1_3_Int32 -.- desc_1_3_Int32["The length of the function result value, in bytes (this count does not include itself). Can be zero. As a special case, -1 indicates a NULL function result. No value bytes follow in the NULL case."]
    1_4_Byten -.- desc_1_4_Byten["The value of the function result, in the format indicated by the associated format code. n is the above length."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__V__,desc_1_2_Int32,desc_1_3_Int32,desc_1_4_Byten comment;
```

--- ParseComplete (B) ---
```mermaid
graph TD
  subgraph ParseComplete__B_ ["ParseComplete (B)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__1__["1_1_Byte1('1')"]
      1_2_Int32_4_["1_2_Int32(4)"]
      1_1_Byte1__1__ --> 1_2_Int32_4_
    end
  end

    1_1_Byte1__1__ -.- desc_1_1_Byte1__1__["Identifies the message as a Parse-complete indicator."]
    1_2_Int32_4_ -.- desc_1_2_Int32_4_["Length of message contents in bytes, including self."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__1__,desc_1_2_Int32_4_ comment;
```

--- GSSResponse (F) ---
```mermaid
graph TD
  subgraph GSSResponse__F_ ["GSSResponse (F)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__p__["1_1_Byte1('p')"]
      1_2_Int32["1_2_Int32"]
      1_1_Byte1__p__ --> 1_2_Int32
      1_3_Byten["1_3_Byten"]
      1_2_Int32 --> 1_3_Byten
    end
  end

    1_1_Byte1__p__ -.- desc_1_1_Byte1__p__["Identifies the message as a GSSAPI or SSPI response. Note that this is also used for SASL and password response messages. The exact message type can be deduced from the context."]
    1_2_Int32 -.- desc_1_2_Int32["Length of message contents in bytes, including self."]
    1_3_Byten -.- desc_1_3_Byten["GSSAPI/SSPI specific message data."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__p__,desc_1_2_Int32,desc_1_3_Byten comment;
```

--- BackendKeyData (B) ---
```mermaid
graph TD
  subgraph BackendKeyData__B_ ["BackendKeyData (B)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__K__["1_1_Byte1('K')"]
      1_2_Int32_12_["1_2_Int32(12)"]
      1_1_Byte1__K__ --> 1_2_Int32_12_
      1_3_Int32["1_3_Int32"]
      1_2_Int32_12_ --> 1_3_Int32
      1_4_Int32["1_4_Int32"]
      1_3_Int32 --> 1_4_Int32
    end
  end

    1_1_Byte1__K__ -.- desc_1_1_Byte1__K__["Identifies the message as cancellation key data. The frontend must save these values if it wishes to be able to issue CancelRequest messages later."]
    1_2_Int32_12_ -.- desc_1_2_Int32_12_["Length of message contents in bytes, including self."]
    1_3_Int32 -.- desc_1_3_Int32["The process ID of this backend."]
    1_4_Int32 -.- desc_1_4_Int32["The secret key of this backend."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__K__,desc_1_2_Int32_12_,desc_1_3_Int32,desc_1_4_Int32 comment;
```

--- EmptyQueryResponse (B) ---
```mermaid
graph TD
  subgraph EmptyQueryResponse__B_ ["EmptyQueryResponse (B)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__I__["1_1_Byte1('I')"]
      1_2_Int32_4_["1_2_Int32(4)"]
      1_1_Byte1__I__ --> 1_2_Int32_4_
    end
  end

    1_1_Byte1__I__ -.- desc_1_1_Byte1__I__["Identifies the message as a response to an empty query string. (This substitutes for CommandComplete.)"]
    1_2_Int32_4_ -.- desc_1_2_Int32_4_["Length of message contents in bytes, including self."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__I__,desc_1_2_Int32_4_ comment;
```

--- CommandComplete (B) ---
```mermaid
graph TD
  subgraph CommandComplete__B_ ["CommandComplete (B)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__C__["1_1_Byte1('C')"]
      1_2_Int32["1_2_Int32"]
      1_1_Byte1__C__ --> 1_2_Int32
      1_3_String["1_3_String"]
      1_2_Int32 --> 1_3_String
    end
  end

    1_1_Byte1__C__ -.- desc_1_1_Byte1__C__["Identifies the message as a command-completed response."]
    1_2_Int32 -.- desc_1_2_Int32["Length of message contents in bytes, including self."]
    1_3_String -.- desc_1_3_String["The command tag. This is usually a single word that identifies which SQL command was completed. For an INSERT command, the tag is INSERT oid rows, where rows is the number of rows inserted. oid used to be the object ID of the inserted row if rows was 1 and the target table had OIDs, but OIDs system columns are not supported anymore; therefore oid is always 0. For a DELETE command, the tag is DELETE rows where rows is the number of rows deleted. For an UPDATE command, the tag is UPDATE rows where rows is the number of rows updated. For a SELECT or CREATE TABLE AS command, the tag is SELECT rows where rows is the number of rows retrieved. For a MOVE command, the tag is MOVE rows where rows is the number of rows the cursor's position has been changed by. For a FETCH command, the tag is FETCH rows where rows is the number of rows that have been retrieved from the cursor. For a COPY command, the tag is COPY rows where rows is the number of rows copied. (Note: the row count appears only in PostgreSQL 8.2 and later.)"]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__C__,desc_1_2_Int32,desc_1_3_String comment;
```

--- BindComplete (B) ---
```mermaid
graph TD
  subgraph BindComplete__B_ ["BindComplete (B)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__2__["1_1_Byte1('2')"]
      1_2_Int32_4_["1_2_Int32(4)"]
      1_1_Byte1__2__ --> 1_2_Int32_4_
    end
  end

    1_1_Byte1__2__ -.- desc_1_1_Byte1__2__["Identifies the message as a Bind-complete indicator."]
    1_2_Int32_4_ -.- desc_1_2_Int32_4_["Length of message contents in bytes, including self."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__2__,desc_1_2_Int32_4_ comment;
```

--- Bind (F) ---
```mermaid
graph TD
  subgraph Bind__F_ ["Bind (F)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__B__["1_1_Byte1('B')"]
      1_2_Int32["1_2_Int32"]
      1_1_Byte1__B__ --> 1_2_Int32
      1_3_String["1_3_String"]
      1_2_Int32 --> 1_3_String
      1_4_String["1_4_String"]
      1_3_String --> 1_4_String
      1_5_Int16["1_5_Int16"]
      1_4_String --> 1_5_Int16
      1_6_Int16_C_["1_6_Int16[C]"]
      1_5_Int16 --> 1_6_Int16_C_
      1_7_Int16["1_7_Int16"]
      1_6_Int16_C_ --> 1_7_Int16
    end
    subgraph repeat_1_1 ["Repeat_1"]
      2_1_Int32["2_1_Int32"]
      2_2_Byten["2_2_Byten"]
      2_1_Int32 --> 2_2_Byten
    end
    header_0 --> repeat_1_1
    subgraph repeat_2_2 ["Repeat_2"]
      3_1_Int16["3_1_Int16"]
      3_2_Int16_R_["3_2_Int16[R]"]
      3_1_Int16 --> 3_2_Int16_R_
    end
    repeat_1_1 --> repeat_2_2
  end

    1_1_Byte1__B__ -.- desc_1_1_Byte1__B__["Identifies the message as a Bind command."]
    1_2_Int32 -.- desc_1_2_Int32["Length of message contents in bytes, including self."]
    1_3_String -.- desc_1_3_String["The name of the destination portal (an empty string selects the unnamed portal)."]
    1_4_String -.- desc_1_4_String["The name of the source prepared statement (an empty string selects the unnamed prepared statement)."]
    1_5_Int16 -.- desc_1_5_Int16["The number of parameter format codes that follow (denoted C below). This can be zero to indicate that there are no parameters or that the parameters all use the default format (text); or one, in which case the specified format code is applied to all parameters; or it can equal the actual number of parameters."]
    1_6_Int16_C_ -.- desc_1_6_Int16_C_["The parameter format codes. Each must presently be zero (text) or one (binary)."]
    1_7_Int16 -.- desc_1_7_Int16["The number of parameter values that follow (possibly zero). This must match the number of parameters needed by the query."]
    2_1_Int32 -.- desc_2_1_Int32["The length of the parameter value, in bytes (this count does not include itself). Can be zero. As a special case, -1 indicates a NULL parameter value. No value bytes follow in the NULL case."]
    2_2_Byten -.- desc_2_2_Byten["The value of the parameter, in the format indicated by the associated format code. n is the above length."]
    3_1_Int16 -.- desc_3_1_Int16["The number of result-column format codes that follow (denoted R below). This can be zero to indicate that there are no result columns or that the result columns should all use the default format (text); or one, in which case the specified format code is applied to all result columns (if any); or it can equal the actual number of result columns of the query."]
    3_2_Int16_R_ -.- desc_3_2_Int16_R_["The result-column format codes. Each must presently be zero (text) or one (binary)."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__B__,desc_1_2_Int32,desc_1_3_String,desc_1_4_String,desc_1_5_Int16,desc_1_6_Int16_C_,desc_1_7_Int16,desc_2_1_Int32,desc_2_2_Byten,desc_3_1_Int16,desc_3_2_Int16_R_ comment;
```

--- Parse (F) ---
```mermaid
graph TD
  subgraph Parse__F_ ["Parse (F)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__P__["1_1_Byte1('P')"]
      1_2_Int32["1_2_Int32"]
      1_1_Byte1__P__ --> 1_2_Int32
      1_3_String["1_3_String"]
      1_2_Int32 --> 1_3_String
      1_4_String["1_4_String"]
      1_3_String --> 1_4_String
      1_5_Int16["1_5_Int16"]
      1_4_String --> 1_5_Int16
    end
    subgraph repeat_1_1 ["Repeat_1"]
      2_1_Int32["2_1_Int32"]
    end
    header_0 --> repeat_1_1
  end

    1_1_Byte1__P__ -.- desc_1_1_Byte1__P__["Identifies the message as a Parse command."]
    1_2_Int32 -.- desc_1_2_Int32["Length of message contents in bytes, including self."]
    1_3_String -.- desc_1_3_String["The name of the destination prepared statement (an empty string selects the unnamed prepared statement)."]
    1_4_String -.- desc_1_4_String["The query string to be parsed."]
    1_5_Int16 -.- desc_1_5_Int16["The number of parameter data types specified (can be zero). Note that this is not an indication of the number of parameters that might appear in the query string, only the number that the frontend wants to prespecify types for."]
    2_1_Int32 -.- desc_2_1_Int32["Specifies the object ID of the parameter data type. Placing a zero here is equivalent to leaving the type unspecified."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__P__,desc_1_2_Int32,desc_1_3_String,desc_1_4_String,desc_1_5_Int16,desc_2_1_Int32 comment;
```

--- SSLRequest (F) ---
```mermaid
graph TD
  subgraph SSLRequest__F_ ["SSLRequest (F)"]
    subgraph header_0 ["Header"]
      1_1_Int32_8_["1_1_Int32(8)"]
      1_2_Int32_80877103_["1_2_Int32(80877103)"]
      1_1_Int32_8_ --> 1_2_Int32_80877103_
    end
  end

    1_1_Int32_8_ -.- desc_1_1_Int32_8_["Length of message contents in bytes, including self."]
    1_2_Int32_80877103_ -.- desc_1_2_Int32_80877103_["The SSL request code. The value is chosen to contain 1234 in the most significant 16 bits, and 5679 in the least significant 16 bits. (To avoid confusion, this code must not be the same as any protocol version number.)"]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Int32_8_,desc_1_2_Int32_80877103_ comment;
```

--- AuthenticationSASLFinal (B) ---
```mermaid
graph TD
  subgraph AuthenticationSASLFinal__B_ ["AuthenticationSASLFinal (B)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__R__["1_1_Byte1('R')"]
      1_2_Int32["1_2_Int32"]
      1_1_Byte1__R__ --> 1_2_Int32
      1_3_Int32_12_["1_3_Int32(12)"]
      1_2_Int32 --> 1_3_Int32_12_
      1_4_Byten["1_4_Byten"]
      1_3_Int32_12_ --> 1_4_Byten
    end
  end

    1_1_Byte1__R__ -.- desc_1_1_Byte1__R__["Identifies the message as an authentication request."]
    1_2_Int32 -.- desc_1_2_Int32["Length of message contents in bytes, including self."]
    1_3_Int32_12_ -.- desc_1_3_Int32_12_["Specifies that SASL authentication has completed."]
    1_4_Byten -.- desc_1_4_Byten["SASL outcome additional data, specific to the SASL mechanism being used."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__R__,desc_1_2_Int32,desc_1_3_Int32_12_,desc_1_4_Byten comment;
```

--- PortalSuspended (B) ---
```mermaid
graph TD
  subgraph PortalSuspended__B_ ["PortalSuspended (B)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__s__["1_1_Byte1('s')"]
      1_2_Int32_4_["1_2_Int32(4)"]
      1_1_Byte1__s__ --> 1_2_Int32_4_
    end
  end

    1_1_Byte1__s__ -.- desc_1_1_Byte1__s__["Identifies the message as a portal-suspended indicator. Note this only appears if an Execute message's row-count limit was reached."]
    1_2_Int32_4_ -.- desc_1_2_Int32_4_["Length of message contents in bytes, including self."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__s__,desc_1_2_Int32_4_ comment;
```

--- CopyOutResponse (B) ---
```mermaid
graph TD
  subgraph CopyOutResponse__B_ ["CopyOutResponse (B)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__H__["1_1_Byte1('H')"]
      1_2_Int32["1_2_Int32"]
      1_1_Byte1__H__ --> 1_2_Int32
      1_3_Int8["1_3_Int8"]
      1_2_Int32 --> 1_3_Int8
      1_4_Int16["1_4_Int16"]
      1_3_Int8 --> 1_4_Int16
      1_5_Int16_N_["1_5_Int16[N]"]
      1_4_Int16 --> 1_5_Int16_N_
    end
  end

    1_1_Byte1__H__ -.- desc_1_1_Byte1__H__["Identifies the message as a Start Copy Out response. This message will be followed by copy-out data."]
    1_2_Int32 -.- desc_1_2_Int32["Length of message contents in bytes, including self."]
    1_3_Int8 -.- desc_1_3_Int8["0 indicates the overall COPY format is textual (rows separated by newlines, columns separated by separator characters, etc). 1 indicates the overall copy format is binary (similar to DataRow format). See COPY for more information."]
    1_4_Int16 -.- desc_1_4_Int16["The number of columns in the data to be copied (denoted N below)."]
    1_5_Int16_N_ -.- desc_1_5_Int16_N_["The format codes to be used for each column. Each must presently be zero (text) or one (binary). All must be zero if the overall copy format is textual."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__H__,desc_1_2_Int32,desc_1_3_Int8,desc_1_4_Int16,desc_1_5_Int16_N_ comment;
```

--- ParameterDescription (B) ---
```mermaid
graph TD
  subgraph ParameterDescription__B_ ["ParameterDescription (B)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__t__["1_1_Byte1('t')"]
      1_2_Int32["1_2_Int32"]
      1_1_Byte1__t__ --> 1_2_Int32
      1_3_Int16["1_3_Int16"]
      1_2_Int32 --> 1_3_Int16
    end
    subgraph repeat_1_1 ["Repeat_1"]
      2_1_Int32["2_1_Int32"]
    end
    header_0 --> repeat_1_1
  end

    1_1_Byte1__t__ -.- desc_1_1_Byte1__t__["Identifies the message as a parameter description."]
    1_2_Int32 -.- desc_1_2_Int32["Length of message contents in bytes, including self."]
    1_3_Int16 -.- desc_1_3_Int16["The number of parameters used by the statement (can be zero)."]
    2_1_Int32 -.- desc_2_1_Int32["Specifies the object ID of the parameter data type."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__t__,desc_1_2_Int32,desc_1_3_Int16,desc_2_1_Int32 comment;
```

--- FunctionCall (F) ---
```mermaid
graph TD
  subgraph FunctionCall__F_ ["FunctionCall (F)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__F__["1_1_Byte1('F')"]
      1_2_Int32["1_2_Int32"]
      1_1_Byte1__F__ --> 1_2_Int32
      1_3_Int32["1_3_Int32"]
      1_2_Int32 --> 1_3_Int32
      1_4_Int16["1_4_Int16"]
      1_3_Int32 --> 1_4_Int16
      1_5_Int16_C_["1_5_Int16[C]"]
      1_4_Int16 --> 1_5_Int16_C_
      1_6_Int16["1_6_Int16"]
      1_5_Int16_C_ --> 1_6_Int16
    end
    subgraph repeat_1_1 ["Repeat_1"]
      2_1_Int32["2_1_Int32"]
      2_2_Byten["2_2_Byten"]
      2_1_Int32 --> 2_2_Byten
    end
    header_0 --> repeat_1_1
    subgraph repeat_2_2 ["Repeat_2"]
      3_1_Int16["3_1_Int16"]
    end
    repeat_1_1 --> repeat_2_2
  end

    1_1_Byte1__F__ -.- desc_1_1_Byte1__F__["Identifies the message as a function call."]
    1_2_Int32 -.- desc_1_2_Int32["Length of message contents in bytes, including self."]
    1_3_Int32 -.- desc_1_3_Int32["Specifies the object ID of the function to call."]
    1_4_Int16 -.- desc_1_4_Int16["The number of argument format codes that follow (denoted C below). This can be zero to indicate that there are no arguments or that the arguments all use the default format (text); or one, in which case the specified format code is applied to all arguments; or it can equal the actual number of arguments."]
    1_5_Int16_C_ -.- desc_1_5_Int16_C_["The argument format codes. Each must presently be zero (text) or one (binary)."]
    1_6_Int16 -.- desc_1_6_Int16["Specifies the number of arguments being supplied to the function."]
    2_1_Int32 -.- desc_2_1_Int32["The length of the argument value, in bytes (this count does not include itself). Can be zero. As a special case, -1 indicates a NULL argument value. No value bytes follow in the NULL case."]
    2_2_Byten -.- desc_2_2_Byten["The value of the argument, in the format indicated by the associated format code. n is the above length."]
    3_1_Int16 -.- desc_3_1_Int16["The format code for the function result. Must presently be zero (text) or one (binary)."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__F__,desc_1_2_Int32,desc_1_3_Int32,desc_1_4_Int16,desc_1_5_Int16_C_,desc_1_6_Int16,desc_2_1_Int32,desc_2_2_Byten,desc_3_1_Int16 comment;
```

--- Sync (F) ---
```mermaid
graph TD
  subgraph Sync__F_ ["Sync (F)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__S__["1_1_Byte1('S')"]
      1_2_Int32_4_["1_2_Int32(4)"]
      1_1_Byte1__S__ --> 1_2_Int32_4_
    end
  end

    1_1_Byte1__S__ -.- desc_1_1_Byte1__S__["Identifies the message as a Sync command."]
    1_2_Int32_4_ -.- desc_1_2_Int32_4_["Length of message contents in bytes, including self."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__S__,desc_1_2_Int32_4_ comment;
```

--- AuthenticationSCMCredential (B) ---
```mermaid
graph TD
  subgraph AuthenticationSCMCredential__B_ ["AuthenticationSCMCredential (B)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__R__["1_1_Byte1('R')"]
      1_2_Int32_8_["1_2_Int32(8)"]
      1_1_Byte1__R__ --> 1_2_Int32_8_
      1_3_Int32_6_["1_3_Int32(6)"]
      1_2_Int32_8_ --> 1_3_Int32_6_
    end
  end

    1_1_Byte1__R__ -.- desc_1_1_Byte1__R__["Identifies the message as an authentication request."]
    1_2_Int32_8_ -.- desc_1_2_Int32_8_["Length of message contents in bytes, including self."]
    1_3_Int32_6_ -.- desc_1_3_Int32_6_["Specifies that an SCM credentials message is required."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__R__,desc_1_2_Int32_8_,desc_1_3_Int32_6_ comment;
```

--- CopyData (F & B) ---
```mermaid
graph TD
  subgraph CopyData__F___B_ ["CopyData (F & B)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__d__["1_1_Byte1('d')"]
      1_2_Int32["1_2_Int32"]
      1_1_Byte1__d__ --> 1_2_Int32
      1_3_Byten["1_3_Byten"]
      1_2_Int32 --> 1_3_Byten
    end
  end

    1_1_Byte1__d__ -.- desc_1_1_Byte1__d__["Identifies the message as COPY data."]
    1_2_Int32 -.- desc_1_2_Int32["Length of message contents in bytes, including self."]
    1_3_Byten -.- desc_1_3_Byten["Data that forms part of a COPY data stream. Messages sent from the backend will always correspond to single data rows, but messages sent by frontends might divide the data stream arbitrarily."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__d__,desc_1_2_Int32,desc_1_3_Byten comment;
```

--- ReadyForQuery (B) ---
```mermaid
graph TD
  subgraph ReadyForQuery__B_ ["ReadyForQuery (B)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__Z__["1_1_Byte1('Z')"]
      1_2_Int32_5_["1_2_Int32(5)"]
      1_1_Byte1__Z__ --> 1_2_Int32_5_
      1_3_Byte1["1_3_Byte1"]
      1_2_Int32_5_ --> 1_3_Byte1
    end
  end

    1_1_Byte1__Z__ -.- desc_1_1_Byte1__Z__["Identifies the message type. ReadyForQuery is sent whenever the backend is ready for a new query cycle."]
    1_2_Int32_5_ -.- desc_1_2_Int32_5_["Length of message contents in bytes, including self."]
    1_3_Byte1 -.- desc_1_3_Byte1["Current backend transaction status indicator. Possible values are 'I' if idle (not in a transaction block); 'T' if in a transaction block; or 'E' if in a failed transaction block (queries will be rejected until block is ended)."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__Z__,desc_1_2_Int32_5_,desc_1_3_Byte1 comment;
```

--- NoticeResponse (B) ---
```mermaid
graph TD
  subgraph NoticeResponse__B_ ["NoticeResponse (B)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__N__["1_1_Byte1('N')"]
      1_2_Int32["1_2_Int32"]
      1_1_Byte1__N__ --> 1_2_Int32
    end
    subgraph repeat_1_1 ["Repeat_1"]
      2_1_Byte1["2_1_Byte1"]
      2_2_String["2_2_String"]
      2_1_Byte1 --> 2_2_String
    end
    header_0 --> repeat_1_1
  end

    1_1_Byte1__N__ -.- desc_1_1_Byte1__N__["Identifies the message as a notice."]
    1_2_Int32 -.- desc_1_2_Int32["Length of message contents in bytes, including self."]
    2_1_Byte1 -.- desc_2_1_Byte1["A code identifying the field type; if zero, this is the message terminator and no string follows. The presently defined field types are listed in Section 53.8. Since more field types might be added in future, frontends should silently ignore fields of unrecognized type."]
    2_2_String -.- desc_2_2_String["The field value."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__N__,desc_1_2_Int32,desc_2_1_Byte1,desc_2_2_String comment;
```

--- CopyInResponse (B) ---
```mermaid
graph TD
  subgraph CopyInResponse__B_ ["CopyInResponse (B)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__G__["1_1_Byte1('G')"]
      1_2_Int32["1_2_Int32"]
      1_1_Byte1__G__ --> 1_2_Int32
      1_3_Int8["1_3_Int8"]
      1_2_Int32 --> 1_3_Int8
      1_4_Int16["1_4_Int16"]
      1_3_Int8 --> 1_4_Int16
      1_5_Int16_N_["1_5_Int16[N]"]
      1_4_Int16 --> 1_5_Int16_N_
    end
  end

    1_1_Byte1__G__ -.- desc_1_1_Byte1__G__["Identifies the message as a Start Copy In response. The frontend must now send copy-in data (if not prepared to do so, send a CopyFail message)."]
    1_2_Int32 -.- desc_1_2_Int32["Length of message contents in bytes, including self."]
    1_3_Int8 -.- desc_1_3_Int8["0 indicates the overall COPY format is textual (rows separated by newlines, columns separated by separator characters, etc). 1 indicates the overall copy format is binary (similar to DataRow format). See COPY for more information."]
    1_4_Int16 -.- desc_1_4_Int16["The number of columns in the data to be copied (denoted N below)."]
    1_5_Int16_N_ -.- desc_1_5_Int16_N_["The format codes to be used for each column. Each must presently be zero (text) or one (binary). All must be zero if the overall copy format is textual."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__G__,desc_1_2_Int32,desc_1_3_Int8,desc_1_4_Int16,desc_1_5_Int16_N_ comment;
```

--- PasswordMessage (F) ---
```mermaid
graph TD
  subgraph PasswordMessage__F_ ["PasswordMessage (F)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__p__["1_1_Byte1('p')"]
      1_2_Int32["1_2_Int32"]
      1_1_Byte1__p__ --> 1_2_Int32
      1_3_String["1_3_String"]
      1_2_Int32 --> 1_3_String
    end
  end

    1_1_Byte1__p__ -.- desc_1_1_Byte1__p__["Identifies the message as a password response. Note that this is also used for GSSAPI, SSPI and SASL response messages. The exact message type can be deduced from the context."]
    1_2_Int32 -.- desc_1_2_Int32["Length of message contents in bytes, including self."]
    1_3_String -.- desc_1_3_String["The password (encrypted, if requested)."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__p__,desc_1_2_Int32,desc_1_3_String comment;
```

--- CancelRequest (F) ---
```mermaid
graph TD
  subgraph CancelRequest__F_ ["CancelRequest (F)"]
    subgraph header_0 ["Header"]
      1_1_Int32_16_["1_1_Int32(16)"]
      1_2_Int32_80877102_["1_2_Int32(80877102)"]
      1_1_Int32_16_ --> 1_2_Int32_80877102_
      1_3_Int32["1_3_Int32"]
      1_2_Int32_80877102_ --> 1_3_Int32
      1_4_Int32["1_4_Int32"]
      1_3_Int32 --> 1_4_Int32
    end
  end

    1_1_Int32_16_ -.- desc_1_1_Int32_16_["Length of message contents in bytes, including self."]
    1_2_Int32_80877102_ -.- desc_1_2_Int32_80877102_["The cancel request code. The value is chosen to contain 1234 in the most significant 16 bits, and 5678 in the least significant 16 bits. (To avoid confusion, this code must not be the same as any protocol version number.)"]
    1_3_Int32 -.- desc_1_3_Int32["The process ID of the target backend."]
    1_4_Int32 -.- desc_1_4_Int32["The secret key for the target backend."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Int32_16_,desc_1_2_Int32_80877102_,desc_1_3_Int32,desc_1_4_Int32 comment;
```

--- CopyDone (F & B) ---
```mermaid
graph TD
  subgraph CopyDone__F___B_ ["CopyDone (F & B)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__c__["1_1_Byte1('c')"]
      1_2_Int32_4_["1_2_Int32(4)"]
      1_1_Byte1__c__ --> 1_2_Int32_4_
    end
  end

    1_1_Byte1__c__ -.- desc_1_1_Byte1__c__["Identifies the message as a COPY-complete indicator."]
    1_2_Int32_4_ -.- desc_1_2_Int32_4_["Length of message contents in bytes, including self."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__c__,desc_1_2_Int32_4_ comment;
```

--- Query (F) ---
```mermaid
graph TD
  subgraph Query__F_ ["Query (F)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__Q__["1_1_Byte1('Q')"]
      1_2_Int32["1_2_Int32"]
      1_1_Byte1__Q__ --> 1_2_Int32
      1_3_String["1_3_String"]
      1_2_Int32 --> 1_3_String
    end
  end

    1_1_Byte1__Q__ -.- desc_1_1_Byte1__Q__["Identifies the message as a simple query."]
    1_2_Int32 -.- desc_1_2_Int32["Length of message contents in bytes, including self."]
    1_3_String -.- desc_1_3_String["The query string itself."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__Q__,desc_1_2_Int32,desc_1_3_String comment;
```

--- DataRow (B) ---
```mermaid
graph TD
  subgraph DataRow__B_ ["DataRow (B)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__D__["1_1_Byte1('D')"]
      1_2_Int32["1_2_Int32"]
      1_1_Byte1__D__ --> 1_2_Int32
      1_3_Int16["1_3_Int16"]
      1_2_Int32 --> 1_3_Int16
    end
    subgraph repeat_1_1 ["Repeat_1"]
      2_1_Int32["2_1_Int32"]
      2_2_Byten["2_2_Byten"]
      2_1_Int32 --> 2_2_Byten
    end
    header_0 --> repeat_1_1
  end

    1_1_Byte1__D__ -.- desc_1_1_Byte1__D__["Identifies the message as a data row."]
    1_2_Int32 -.- desc_1_2_Int32["Length of message contents in bytes, including self."]
    1_3_Int16 -.- desc_1_3_Int16["The number of column values that follow (possibly zero)."]
    2_1_Int32 -.- desc_2_1_Int32["The length of the column value, in bytes (this count does not include itself). Can be zero. As a special case, -1 indicates a NULL column value. No value bytes follow in the NULL case."]
    2_2_Byten -.- desc_2_2_Byten["The value of the column, in the format indicated by the associated format code. n is the above length."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__D__,desc_1_2_Int32,desc_1_3_Int16,desc_2_1_Int32,desc_2_2_Byten comment;
```

--- StartupMessage (F) ---
```mermaid
graph TD
  subgraph StartupMessage__F_ ["StartupMessage (F)"]
    subgraph header_0 ["Header"]
      1_1_Int32["1_1_Int32"]
      1_2_Int32_196608_["1_2_Int32(196608)"]
      1_1_Int32 --> 1_2_Int32_196608_
    end
    subgraph repeat_1_1 ["Repeat_1"]
      2_1_String["2_1_String"]
      2_2_user["2_2_user"]
      2_1_String --> 2_2_user
      2_3_database["2_3_database"]
      2_2_user --> 2_3_database
      2_4_options["2_4_options"]
      2_3_database --> 2_4_options
      2_5_replication["2_5_replication"]
      2_4_options --> 2_5_replication
      2_6_String["2_6_String"]
      2_5_replication --> 2_6_String
    end
    header_0 --> repeat_1_1
    subgraph repeat_2_2 ["Repeat_2"]
      3_1_user["3_1_user"]
      3_2_database["3_2_database"]
      3_1_user --> 3_2_database
      3_3_options["3_3_options"]
      3_2_database --> 3_3_options
      3_4_replication["3_4_replication"]
      3_3_options --> 3_4_replication
    end
    repeat_1_1 --> repeat_2_2
  end

    1_1_Int32 -.- desc_1_1_Int32["Length of message contents in bytes, including self."]
    1_2_Int32_196608_ -.- desc_1_2_Int32_196608_["The protocol version number. The most significant 16 bits are the major version number (3 for the protocol described here). The least significant 16 bits are the minor version number (0 for the protocol described here)."]
    2_1_String -.- desc_2_1_String["The parameter name. Currently recognized names are: user The database user name to connect as. Required; there is no default. database The database to connect to. Defaults to the user name. options Command-line arguments for the backend. (This is deprecated in favor of setting individual run-time parameters.) Spaces within this string are considered to separate arguments, unless escaped with a backslash (\); write \\ to represent a literal backslash. replication Used to connect in streaming replication mode, where a small set of replication commands can be issued instead of SQL statements. Value can be true, false, or database, and the default is false. See Section 53.4 for details. In addition to the above, other parameters may be listed. Parameter names beginning with _pq_. are reserved for use as protocol extensions, while others are treated as run-time parameters to be set at backend start time. Such settings will be applied during backend start (after parsing the command-line arguments if any) and will act as session defaults."]
    2_2_user -.- desc_2_2_user["The database user name to connect as. Required; there is no default."]
    2_3_database -.- desc_2_3_database["The database to connect to. Defaults to the user name."]
    2_4_options -.- desc_2_4_options["Command-line arguments for the backend. (This is deprecated in favor of setting individual run-time parameters.) Spaces within this string are considered to separate arguments, unless escaped with a backslash (\); write \\ to represent a literal backslash."]
    2_5_replication -.- desc_2_5_replication["Used to connect in streaming replication mode, where a small set of replication commands can be issued instead of SQL statements. Value can be true, false, or database, and the default is false. See Section 53.4 for details."]
    2_6_String -.- desc_2_6_String["The parameter value."]
    3_1_user -.- desc_3_1_user["The database user name to connect as. Required; there is no default."]
    3_2_database -.- desc_3_2_database["The database to connect to. Defaults to the user name."]
    3_3_options -.- desc_3_3_options["Command-line arguments for the backend. (This is deprecated in favor of setting individual run-time parameters.) Spaces within this string are considered to separate arguments, unless escaped with a backslash (\); write \\ to represent a literal backslash."]
    3_4_replication -.- desc_3_4_replication["Used to connect in streaming replication mode, where a small set of replication commands can be issued instead of SQL statements. Value can be true, false, or database, and the default is false. See Section 53.4 for details."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Int32,desc_1_2_Int32_196608_,desc_2_1_String,desc_2_2_user,desc_2_3_database,desc_2_4_options,desc_2_5_replication,desc_2_6_String,desc_3_1_user,desc_3_2_database,desc_3_3_options,desc_3_4_replication comment;
```

--- SASLResponse (F) ---
```mermaid
graph TD
  subgraph SASLResponse__F_ ["SASLResponse (F)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__p__["1_1_Byte1('p')"]
      1_2_Int32["1_2_Int32"]
      1_1_Byte1__p__ --> 1_2_Int32
      1_3_Byten["1_3_Byten"]
      1_2_Int32 --> 1_3_Byten
    end
  end

    1_1_Byte1__p__ -.- desc_1_1_Byte1__p__["Identifies the message as a SASL response. Note that this is also used for GSSAPI, SSPI and password response messages. The exact message type can be deduced from the context."]
    1_2_Int32 -.- desc_1_2_Int32["Length of message contents in bytes, including self."]
    1_3_Byten -.- desc_1_3_Byten["SASL mechanism specific message data."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__p__,desc_1_2_Int32,desc_1_3_Byten comment;
```

--- AuthenticationGSSContinue (B) ---
```mermaid
graph TD
  subgraph AuthenticationGSSContinue__B_ ["AuthenticationGSSContinue (B)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__R__["1_1_Byte1('R')"]
      1_2_Int32["1_2_Int32"]
      1_1_Byte1__R__ --> 1_2_Int32
      1_3_Int32_8_["1_3_Int32(8)"]
      1_2_Int32 --> 1_3_Int32_8_
      1_4_Byten["1_4_Byten"]
      1_3_Int32_8_ --> 1_4_Byten
    end
  end

    1_1_Byte1__R__ -.- desc_1_1_Byte1__R__["Identifies the message as an authentication request."]
    1_2_Int32 -.- desc_1_2_Int32["Length of message contents in bytes, including self."]
    1_3_Int32_8_ -.- desc_1_3_Int32_8_["Specifies that this message contains GSSAPI or SSPI data."]
    1_4_Byten -.- desc_1_4_Byten["GSSAPI or SSPI authentication data."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__R__,desc_1_2_Int32,desc_1_3_Int32_8_,desc_1_4_Byten comment;
```

--- ErrorResponse (B) ---
```mermaid
graph TD
  subgraph ErrorResponse__B_ ["ErrorResponse (B)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__E__["1_1_Byte1('E')"]
      1_2_Int32["1_2_Int32"]
      1_1_Byte1__E__ --> 1_2_Int32
    end
    subgraph repeat_1_1 ["Repeat_1"]
      2_1_Byte1["2_1_Byte1"]
      2_2_String["2_2_String"]
      2_1_Byte1 --> 2_2_String
    end
    header_0 --> repeat_1_1
  end

    1_1_Byte1__E__ -.- desc_1_1_Byte1__E__["Identifies the message as an error."]
    1_2_Int32 -.- desc_1_2_Int32["Length of message contents in bytes, including self."]
    2_1_Byte1 -.- desc_2_1_Byte1["A code identifying the field type; if zero, this is the message terminator and no string follows. The presently defined field types are listed in Section 53.8. Since more field types might be added in future, frontends should silently ignore fields of unrecognized type."]
    2_2_String -.- desc_2_2_String["The field value."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__E__,desc_1_2_Int32,desc_2_1_Byte1,desc_2_2_String comment;
```

--- CloseComplete (B) ---
```mermaid
graph TD
  subgraph CloseComplete__B_ ["CloseComplete (B)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__3__["1_1_Byte1('3')"]
      1_2_Int32_4_["1_2_Int32(4)"]
      1_1_Byte1__3__ --> 1_2_Int32_4_
    end
  end

    1_1_Byte1__3__ -.- desc_1_1_Byte1__3__["Identifies the message as a Close-complete indicator."]
    1_2_Int32_4_ -.- desc_1_2_Int32_4_["Length of message contents in bytes, including self."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__3__,desc_1_2_Int32_4_ comment;
```

--- AuthenticationSSPI (B) ---
```mermaid
graph TD
  subgraph AuthenticationSSPI__B_ ["AuthenticationSSPI (B)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__R__["1_1_Byte1('R')"]
      1_2_Int32_8_["1_2_Int32(8)"]
      1_1_Byte1__R__ --> 1_2_Int32_8_
      1_3_Int32_9_["1_3_Int32(9)"]
      1_2_Int32_8_ --> 1_3_Int32_9_
    end
  end

    1_1_Byte1__R__ -.- desc_1_1_Byte1__R__["Identifies the message as an authentication request."]
    1_2_Int32_8_ -.- desc_1_2_Int32_8_["Length of message contents in bytes, including self."]
    1_3_Int32_9_ -.- desc_1_3_Int32_9_["Specifies that SSPI authentication is required."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__R__,desc_1_2_Int32_8_,desc_1_3_Int32_9_ comment;
```

--- CopyBothResponse (B) ---
```mermaid
graph TD
  subgraph CopyBothResponse__B_ ["CopyBothResponse (B)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__W__["1_1_Byte1('W')"]
      1_2_Int32["1_2_Int32"]
      1_1_Byte1__W__ --> 1_2_Int32
      1_3_Int8["1_3_Int8"]
      1_2_Int32 --> 1_3_Int8
      1_4_Int16["1_4_Int16"]
      1_3_Int8 --> 1_4_Int16
      1_5_Int16_N_["1_5_Int16[N]"]
      1_4_Int16 --> 1_5_Int16_N_
    end
  end

    1_1_Byte1__W__ -.- desc_1_1_Byte1__W__["Identifies the message as a Start Copy Both response. This message is used only for Streaming Replication."]
    1_2_Int32 -.- desc_1_2_Int32["Length of message contents in bytes, including self."]
    1_3_Int8 -.- desc_1_3_Int8["0 indicates the overall COPY format is textual (rows separated by newlines, columns separated by separator characters, etc). 1 indicates the overall copy format is binary (similar to DataRow format). See COPY for more information."]
    1_4_Int16 -.- desc_1_4_Int16["The number of columns in the data to be copied (denoted N below)."]
    1_5_Int16_N_ -.- desc_1_5_Int16_N_["The format codes to be used for each column. Each must presently be zero (text) or one (binary). All must be zero if the overall copy format is textual."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__W__,desc_1_2_Int32,desc_1_3_Int8,desc_1_4_Int16,desc_1_5_Int16_N_ comment;
```

--- AuthenticationOk (B) ---
```mermaid
graph TD
  subgraph AuthenticationOk__B_ ["AuthenticationOk (B)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__R__["1_1_Byte1('R')"]
      1_2_Int32_8_["1_2_Int32(8)"]
      1_1_Byte1__R__ --> 1_2_Int32_8_
      1_3_Int32_0_["1_3_Int32(0)"]
      1_2_Int32_8_ --> 1_3_Int32_0_
    end
  end

    1_1_Byte1__R__ -.- desc_1_1_Byte1__R__["Identifies the message as an authentication request."]
    1_2_Int32_8_ -.- desc_1_2_Int32_8_["Length of message contents in bytes, including self."]
    1_3_Int32_0_ -.- desc_1_3_Int32_0_["Specifies that the authentication was successful."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__R__,desc_1_2_Int32_8_,desc_1_3_Int32_0_ comment;
```

--- AuthenticationMD5Password (B) ---
```mermaid
graph TD
  subgraph AuthenticationMD5Password__B_ ["AuthenticationMD5Password (B)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__R__["1_1_Byte1('R')"]
      1_2_Int32_12_["1_2_Int32(12)"]
      1_1_Byte1__R__ --> 1_2_Int32_12_
      1_3_Int32_5_["1_3_Int32(5)"]
      1_2_Int32_12_ --> 1_3_Int32_5_
      1_4_Byte4["1_4_Byte4"]
      1_3_Int32_5_ --> 1_4_Byte4
    end
  end

    1_1_Byte1__R__ -.- desc_1_1_Byte1__R__["Identifies the message as an authentication request."]
    1_2_Int32_12_ -.- desc_1_2_Int32_12_["Length of message contents in bytes, including self."]
    1_3_Int32_5_ -.- desc_1_3_Int32_5_["Specifies that an MD5-encrypted password is required."]
    1_4_Byte4 -.- desc_1_4_Byte4["The salt to use when encrypting the password."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__R__,desc_1_2_Int32_12_,desc_1_3_Int32_5_,desc_1_4_Byte4 comment;
```

--- NegotiateProtocolVersion (B) ---
```mermaid
graph TD
  subgraph NegotiateProtocolVersion__B_ ["NegotiateProtocolVersion (B)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__v__["1_1_Byte1('v')"]
      1_2_Int32["1_2_Int32"]
      1_1_Byte1__v__ --> 1_2_Int32
      1_3_Int32["1_3_Int32"]
      1_2_Int32 --> 1_3_Int32
      1_4_Int32["1_4_Int32"]
      1_3_Int32 --> 1_4_Int32
    end
    subgraph repeat_1_1 ["Repeat_1"]
      2_1_String["2_1_String"]
    end
    header_0 --> repeat_1_1
  end

    1_1_Byte1__v__ -.- desc_1_1_Byte1__v__["Identifies the message as a protocol version negotiation message."]
    1_2_Int32 -.- desc_1_2_Int32["Length of message contents in bytes, including self."]
    1_3_Int32 -.- desc_1_3_Int32["Newest minor protocol version supported by the server for the major protocol version requested by the client."]
    1_4_Int32 -.- desc_1_4_Int32["Number of protocol options not recognized by the server."]
    2_1_String -.- desc_2_1_String["The option name."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__v__,desc_1_2_Int32,desc_1_3_Int32,desc_1_4_Int32,desc_2_1_String comment;
```

--- ParameterStatus (B) ---
```mermaid
graph TD
  subgraph ParameterStatus__B_ ["ParameterStatus (B)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__S__["1_1_Byte1('S')"]
      1_2_Int32["1_2_Int32"]
      1_1_Byte1__S__ --> 1_2_Int32
      1_3_String["1_3_String"]
      1_2_Int32 --> 1_3_String
      1_4_String["1_4_String"]
      1_3_String --> 1_4_String
    end
  end

    1_1_Byte1__S__ -.- desc_1_1_Byte1__S__["Identifies the message as a run-time parameter status report."]
    1_2_Int32 -.- desc_1_2_Int32["Length of message contents in bytes, including self."]
    1_3_String -.- desc_1_3_String["The name of the run-time parameter being reported."]
    1_4_String -.- desc_1_4_String["The current value of the parameter."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__S__,desc_1_2_Int32,desc_1_3_String,desc_1_4_String comment;
```

--- AuthenticationSASLContinue (B) ---
```mermaid
graph TD
  subgraph AuthenticationSASLContinue__B_ ["AuthenticationSASLContinue (B)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__R__["1_1_Byte1('R')"]
      1_2_Int32["1_2_Int32"]
      1_1_Byte1__R__ --> 1_2_Int32
      1_3_Int32_11_["1_3_Int32(11)"]
      1_2_Int32 --> 1_3_Int32_11_
      1_4_Byten["1_4_Byten"]
      1_3_Int32_11_ --> 1_4_Byten
    end
  end

    1_1_Byte1__R__ -.- desc_1_1_Byte1__R__["Identifies the message as an authentication request."]
    1_2_Int32 -.- desc_1_2_Int32["Length of message contents in bytes, including self."]
    1_3_Int32_11_ -.- desc_1_3_Int32_11_["Specifies that this message contains a SASL challenge."]
    1_4_Byten -.- desc_1_4_Byten["SASL data, specific to the SASL mechanism being used."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__R__,desc_1_2_Int32,desc_1_3_Int32_11_,desc_1_4_Byten comment;
```

--- AuthenticationSASL (B) ---
```mermaid
graph TD
  subgraph AuthenticationSASL__B_ ["AuthenticationSASL (B)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__R__["1_1_Byte1('R')"]
      1_2_Int32["1_2_Int32"]
      1_1_Byte1__R__ --> 1_2_Int32
      1_3_Int32_10_["1_3_Int32(10)"]
      1_2_Int32 --> 1_3_Int32_10_
    end
    subgraph repeat_1_1 ["Repeat_1"]
      2_1_String["2_1_String"]
    end
    header_0 --> repeat_1_1
  end

    1_1_Byte1__R__ -.- desc_1_1_Byte1__R__["Identifies the message as an authentication request."]
    1_2_Int32 -.- desc_1_2_Int32["Length of message contents in bytes, including self."]
    1_3_Int32_10_ -.- desc_1_3_Int32_10_["Specifies that SASL authentication is required."]
    2_1_String -.- desc_2_1_String["Name of a SASL authentication mechanism."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__R__,desc_1_2_Int32,desc_1_3_Int32_10_,desc_2_1_String comment;
```

--- Flush (F) ---
```mermaid
graph TD
  subgraph Flush__F_ ["Flush (F)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__H__["1_1_Byte1('H')"]
      1_2_Int32_4_["1_2_Int32(4)"]
      1_1_Byte1__H__ --> 1_2_Int32_4_
    end
  end

    1_1_Byte1__H__ -.- desc_1_1_Byte1__H__["Identifies the message as a Flush command."]
    1_2_Int32_4_ -.- desc_1_2_Int32_4_["Length of message contents in bytes, including self."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__H__,desc_1_2_Int32_4_ comment;
```

--- CopyFail (F) ---
```mermaid
graph TD
  subgraph CopyFail__F_ ["CopyFail (F)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__f__["1_1_Byte1('f')"]
      1_2_Int32["1_2_Int32"]
      1_1_Byte1__f__ --> 1_2_Int32
      1_3_String["1_3_String"]
      1_2_Int32 --> 1_3_String
    end
  end

    1_1_Byte1__f__ -.- desc_1_1_Byte1__f__["Identifies the message as a COPY-failure indicator."]
    1_2_Int32 -.- desc_1_2_Int32["Length of message contents in bytes, including self."]
    1_3_String -.- desc_1_3_String["An error message to report as the cause of failure."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__f__,desc_1_2_Int32,desc_1_3_String comment;
```

--- Describe (F) ---
```mermaid
graph TD
  subgraph Describe__F_ ["Describe (F)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__D__["1_1_Byte1('D')"]
      1_2_Int32["1_2_Int32"]
      1_1_Byte1__D__ --> 1_2_Int32
      1_3_Byte1["1_3_Byte1"]
      1_2_Int32 --> 1_3_Byte1
      1_4_String["1_4_String"]
      1_3_Byte1 --> 1_4_String
    end
  end

    1_1_Byte1__D__ -.- desc_1_1_Byte1__D__["Identifies the message as a Describe command."]
    1_2_Int32 -.- desc_1_2_Int32["Length of message contents in bytes, including self."]
    1_3_Byte1 -.- desc_1_3_Byte1["'S' to describe a prepared statement; or 'P' to describe a portal."]
    1_4_String -.- desc_1_4_String["The name of the prepared statement or portal to describe (an empty string selects the unnamed prepared statement or portal)."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__D__,desc_1_2_Int32,desc_1_3_Byte1,desc_1_4_String comment;
```

--- AuthenticationGSS (B) ---
```mermaid
graph TD
  subgraph AuthenticationGSS__B_ ["AuthenticationGSS (B)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__R__["1_1_Byte1('R')"]
      1_2_Int32_8_["1_2_Int32(8)"]
      1_1_Byte1__R__ --> 1_2_Int32_8_
      1_3_Int32_7_["1_3_Int32(7)"]
      1_2_Int32_8_ --> 1_3_Int32_7_
    end
  end

    1_1_Byte1__R__ -.- desc_1_1_Byte1__R__["Identifies the message as an authentication request."]
    1_2_Int32_8_ -.- desc_1_2_Int32_8_["Length of message contents in bytes, including self."]
    1_3_Int32_7_ -.- desc_1_3_Int32_7_["Specifies that GSSAPI authentication is required."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__R__,desc_1_2_Int32_8_,desc_1_3_Int32_7_ comment;
```

--- SASLInitialResponse (F) ---
```mermaid
graph TD
  subgraph SASLInitialResponse__F_ ["SASLInitialResponse (F)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__p__["1_1_Byte1('p')"]
      1_2_Int32["1_2_Int32"]
      1_1_Byte1__p__ --> 1_2_Int32
      1_3_String["1_3_String"]
      1_2_Int32 --> 1_3_String
      1_4_Int32["1_4_Int32"]
      1_3_String --> 1_4_Int32
      1_5_Byten["1_5_Byten"]
      1_4_Int32 --> 1_5_Byten
    end
  end

    1_1_Byte1__p__ -.- desc_1_1_Byte1__p__["Identifies the message as an initial SASL response. Note that this is also used for GSSAPI, SSPI and password response messages. The exact message type is deduced from the context."]
    1_2_Int32 -.- desc_1_2_Int32["Length of message contents in bytes, including self."]
    1_3_String -.- desc_1_3_String["Name of the SASL authentication mechanism that the client selected."]
    1_4_Int32 -.- desc_1_4_Int32["Length of SASL mechanism specific Initial Client Response that follows, or -1 if there is no Initial Response."]
    1_5_Byten -.- desc_1_5_Byten["SASL mechanism specific Initial Response."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__p__,desc_1_2_Int32,desc_1_3_String,desc_1_4_Int32,desc_1_5_Byten comment;
```

--- GSSENCRequest (F) ---
```mermaid
graph TD
  subgraph GSSENCRequest__F_ ["GSSENCRequest (F)"]
    subgraph header_0 ["Header"]
      1_1_Int32_8_["1_1_Int32(8)"]
      1_2_Int32_80877104_["1_2_Int32(80877104)"]
      1_1_Int32_8_ --> 1_2_Int32_80877104_
    end
  end

    1_1_Int32_8_ -.- desc_1_1_Int32_8_["Length of message contents in bytes, including self."]
    1_2_Int32_80877104_ -.- desc_1_2_Int32_80877104_["The GSSAPI Encryption request code. The value is chosen to contain 1234 in the most significant 16 bits, and 5680 in the least significant 16 bits. (To avoid confusion, this code must not be the same as any protocol version number.)"]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Int32_8_,desc_1_2_Int32_80877104_ comment;
```

--- AuthenticationKerberosV5 (B) ---
```mermaid
graph TD
  subgraph AuthenticationKerberosV5__B_ ["AuthenticationKerberosV5 (B)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__R__["1_1_Byte1('R')"]
      1_2_Int32_8_["1_2_Int32(8)"]
      1_1_Byte1__R__ --> 1_2_Int32_8_
      1_3_Int32_2_["1_3_Int32(2)"]
      1_2_Int32_8_ --> 1_3_Int32_2_
    end
  end

    1_1_Byte1__R__ -.- desc_1_1_Byte1__R__["Identifies the message as an authentication request."]
    1_2_Int32_8_ -.- desc_1_2_Int32_8_["Length of message contents in bytes, including self."]
    1_3_Int32_2_ -.- desc_1_3_Int32_2_["Specifies that Kerberos V5 authentication is required."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__R__,desc_1_2_Int32_8_,desc_1_3_Int32_2_ comment;
```

--- NoData (B) ---
```mermaid
graph TD
  subgraph NoData__B_ ["NoData (B)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__n__["1_1_Byte1('n')"]
      1_2_Int32_4_["1_2_Int32(4)"]
      1_1_Byte1__n__ --> 1_2_Int32_4_
    end
  end

    1_1_Byte1__n__ -.- desc_1_1_Byte1__n__["Identifies the message as a no-data indicator."]
    1_2_Int32_4_ -.- desc_1_2_Int32_4_["Length of message contents in bytes, including self."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__n__,desc_1_2_Int32_4_ comment;
```

--- AuthenticationCleartextPassword (B) ---
```mermaid
graph TD
  subgraph AuthenticationCleartextPassword__B_ ["AuthenticationCleartextPassword (B)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__R__["1_1_Byte1('R')"]
      1_2_Int32_8_["1_2_Int32(8)"]
      1_1_Byte1__R__ --> 1_2_Int32_8_
      1_3_Int32_3_["1_3_Int32(3)"]
      1_2_Int32_8_ --> 1_3_Int32_3_
    end
  end

    1_1_Byte1__R__ -.- desc_1_1_Byte1__R__["Identifies the message as an authentication request."]
    1_2_Int32_8_ -.- desc_1_2_Int32_8_["Length of message contents in bytes, including self."]
    1_3_Int32_3_ -.- desc_1_3_Int32_3_["Specifies that a clear-text password is required."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__R__,desc_1_2_Int32_8_,desc_1_3_Int32_3_ comment;
```

--- Close (F) ---
```mermaid
graph TD
  subgraph Close__F_ ["Close (F)"]
    subgraph header_0 ["Header"]
      1_1_Byte1__C__["1_1_Byte1('C')"]
      1_2_Int32["1_2_Int32"]
      1_1_Byte1__C__ --> 1_2_Int32
      1_3_Byte1["1_3_Byte1"]
      1_2_Int32 --> 1_3_Byte1
      1_4_String["1_4_String"]
      1_3_Byte1 --> 1_4_String
    end
  end

    1_1_Byte1__C__ -.- desc_1_1_Byte1__C__["Identifies the message as a Close command."]
    1_2_Int32 -.- desc_1_2_Int32["Length of message contents in bytes, including self."]
    1_3_Byte1 -.- desc_1_3_Byte1["'S' to close a prepared statement; or 'P' to close a portal."]
    1_4_String -.- desc_1_4_String["The name of the prepared statement or portal to close (an empty string selects the unnamed prepared statement or portal)."]
  classDef comment fill:#f0f0f0,stroke:#666,stroke-dasharray: 5 5;
  class desc_1_1_Byte1__C__,desc_1_2_Int32,desc_1_3_Byte1,desc_1_4_String comment;
```
