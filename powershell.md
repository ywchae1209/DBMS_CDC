powershell command

* count scala source line(all)

```powershell
Get-ChildItem -Recurse -Filter *.scala | 
    Get-Content | 
    Measure-Object -Line
```

* count scala source line(per file)
```powershell
Get-ChildItem -Recurse -Filter *.scala | 
    ForEach-Object {
        $lines = (Get-Content $_.FullName | Measure-Object -Line).Lines
        [PSCustomObject]@{File=$_.FullName; Lines=$lines}
    }
```

* sample out
```out
File                                                    Lines
----                                                    -----
io\flux\SpecTest.scala                                   214
io\flux\base\package.scala                                73
io\flux\base\ProtoBaseCodec.scala                         57
io\flux\env\StateFile.scala                               52
io\flux\env\StatsRuntime.scala                            36
io\flux\postgres\DecodeTask.scala                        100
io\flux\postgres\PgMeta.scala                             89
io\flux\postgres\PgMetaState.scala                        73
io\flux\postgres\PgMetaZipper.scala                       93
io\flux\postgres\PgOutputCodecs.scala                    559
io\flux\postgres\PgOutputFields.scala                    304
io\flux\postgres\PgOutputMsg.scala                       207
io\flux\postgres\ProtoPgCodec.scala                      657
io\flux\postgres\TaskResult.scala                         60
io\flux\service\ServiceProtobufCodec.scala                 7
io\flux\structural\PgPump.scala                          134
io\flux\structural\PgPumpConf.scala                      108
io\flux\structural\PgPumpContext.scala                    87
io\flux\structural\PgQue.scala                            15
io\flux\structural\PumpController.scala                   79
io\flux\structural\ReadController.scala                   87
io\flux\structural\TopicManager.scala                     71
io\flux\structural\applier\TxSpooler.scala                31
io\flux\structural\topicQue\CQDispatcher.scala            95
io\flux\structural\topicQue\CQQue.scala                   41
io\flux\structural\topicQue\CQReader.scala                35
io\flux\structural\topicQue\CQWriter.scala                25
io\flux\structural\topicQue\traits\PersisitQue.scala      87
io\flux\structural\topicQue\traits\PersistReader.scala    20
io\flux\structural\topicQue\traits\PersistWriter.scala    17
io\flux\utils\mutableState.scala                         106
io\flux\utils\pekkoAux.scala                              63
io\flux\utils\Util.scala                                  76

File                                                                     Lines
----                                                                     -----
io\flux\postgres\proto3\v0\PostgresProto.scala                          168
io\flux\postgres\proto3\v0\P_Begin.scala                                177
io\flux\postgres\proto3\v0\P_BeginPrepare.scala                         234
io\flux\postgres\proto3\v0\P_ColumnValue.scala                          213
io\flux\postgres\proto3\v0\P_Commit.scala                               204
io\flux\postgres\proto3\v0\P_CommitPrepared.scala                       264
io\flux\postgres\proto3\v0\P_Delete.scala                               147
io\flux\postgres\proto3\v0\P_ErrMsg.scala                               114
io\flux\postgres\proto3\v0\P_Ignore.scala                               146
io\flux\postgres\proto3\v0\P_Insert.scala                               147
io\flux\postgres\proto3\v0\P_LogicalMsg.scala                           207
io\flux\postgres\proto3\v0\P_Mode.scala                                  69
io\flux\postgres\proto3\v0\P_Origin.scala                               144
io\flux\postgres\proto3\v0\P_PgOutputMsg.scala                          926
io\flux\postgres\proto3\v0\P_Prepare.scala                              264
io\flux\postgres\proto3\v0\P_Relation.scala                             241
io\flux\postgres\proto3\v0\P_RelationColumn.scala                       204
io\flux\postgres\proto3\v0\P_ReplicaIdentity.scala                       68
io\flux\postgres\proto3\v0\P_RollbackPrepared.scala                     294
io\flux\postgres\proto3\v0\P_StreamAbort.scala                          177
io\flux\postgres\proto3\v0\P_StreamAbortSub.scala                       144
io\flux\postgres\proto3\v0\P_StreamCommit.scala                         234
io\flux\postgres\proto3\v0\P_StreamPrepare.scala                        264
io\flux\postgres\proto3\v0\P_StreamStart.scala                          144
io\flux\postgres\proto3\v0\P_StreamStopMsg.scala                         77
io\flux\postgres\proto3\v0\P_TaskResult.scala                           252
io\flux\postgres\proto3\v0\P_Truncate.scala                             157
io\flux\postgres\proto3\v0\P_TupleData.scala                            117
io\flux\postgres\proto3\v0\P_TypeMsg.scala                              174
io\flux\postgres\proto3\v0\P_Update.scala                               175
io\flux\postgres\proto3\v0\P_X_Delete.scala                             147
io\flux\postgres\proto3\v0\P_X_Insert.scala                             147
io\flux\postgres\proto3\v0\P_X_LogicalMsg.scala                         147
io\flux\postgres\proto3\v0\P_X_Relation.scala                           147
io\flux\postgres\proto3\v0\P_X_Truncate.scala                           147
io\flux\postgres\proto3\v0\P_X_TypeMsg.scala                            147
io\flux\structural\service\proto3\v0\DataServiceClient.scala             97
io\flux\structural\service\proto3\v0\DataServiceHandler.scala           118
io\flux\structural\service\proto3\v0\DataServicePowerApi.scala           15
io\flux\structural\service\proto3\v0\DataServicePowerApiHandler.scala   119
io\flux\structural\service\proto3\v0\P_Block.scala                      177
io\flux\structural\service\proto3\v0\P_BlockOffset.scala                147
io\flux\structural\service\proto3\v0\P_BlockOffsets.scala               117
io\flux\structural\service\proto3\v0\P_GetStreamRequest.scala           180
io\flux\structural\service\proto3\v0\P_StartPumpRequest.scala           148
io\flux\structural\service\proto3\v0\P_StartPumpResponse.scala          169
io\flux\structural\service\proto3\v0\ServiceProto.scala                  46


```
