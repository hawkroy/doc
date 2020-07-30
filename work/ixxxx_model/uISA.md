[TOC]

## Uop的属性信息

在X86指令进行解码的时候，模拟器中的decoder会给解码后的uop标注不同的属性信息，这些信息完全由decoder和uISA的设计决定。在当前的模拟器中，主要有如下的uop属性

| 属性                 | 作用                                                         |
| -------------------- | ------------------------------------------------------------ |
| BOM                  | X86指令的第一条uop                                           |
| EOM                  | X86指令的最后一条uop                                         |
| FP_BOM               |                                                              |
| FP_EOM               |                                                              |
| MMX_BOM              |                                                              |
| WAITTRAP             |                                                              |
| INTR                 |                                                              |
| INTR_N               |                                                              |
| NOLIT                |                                                              |
| DBLLIT               |                                                              |
| IGNBOM0              |                                                              |
| IGNBOM1              |                                                              |
| WRITEFC1             |                                                              |
| WRITEFSW             |                                                              |
| SAVEFLA              |                                                              |
| WRITEMXCSR           |                                                              |
| FP                   |                                                              |
| NI                   |                                                              |
| TAKEN                | taken hint，表明当前的uop一定会进行跳转，用于MSROM中的跳转指令ujcc, ujmp_onedec |
| BR_FENCE             |                                                              |
| MS_NO_PRED           |                                                              |
| SETSCORE             |                                                              |
| READSCORE            |                                                              |
| ENDMSCHUNK           |                                                              |
| IMMNONE              |                                                              |
| IMM16                |                                                              |
| INS_XLAT_TEMPLATE    | trigger MSROM的标志，当decode遇到这个标志，需要将decode逻辑切换到MSROM的解码，同时在流水线中插入一些xlat_template的uop |
| END_OF_LINEINE       |                                                              |
| ESP_FOLD             | esp_folding的uop，当前uop从stack engine读取offset counter，具体参考fusion.md |
| FUSE_OPT_1-21        | micro-fusion的类型，具体参考fusion.md                        |
| TMP_LAM              |                                                              |
| TMPPP                |                                                              |
| RMW_LAM              | rmw类型的uop，decode为1条uop，但是在allocate阶段之后需要split，具体 参考fusion.md |
| NOSCORE_FLAGRENAMING |                                                              |
| BEGIN_OF_FRAME       |                                                              |
| EOF                  |                                                              |
| INF                  |                                                              |
| S2PAIR               |                                                              |
| BOTOP                |                                                              |
| EOTOP                |                                                              |
| QUESTIONABLE         |                                                              |

