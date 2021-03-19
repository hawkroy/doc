[TOC]

# uISA

## uISA编程环境



## uOP属性列表

| 属性名称             | 属性含义                                                     | Timing use?   |
| -------------------- | ------------------------------------------------------------ | ------------- |
| BOM                  | 表示uOP为x86指令的开始                                       | Yes           |
| EOM                  | 表示uOP为x86指令的结束                                       | Yes           |
| FP_BOM               | 表示为x87类型的uOP，只标记在x86的BOM上，用于判断后续的x87指令在fsw.ES=1的时候，是否触发\#MF(math fault)的#NM中断；以下x87指令不会设置该标志：<br />fninit / fnclex / fnstsw / fnstcw / fnstenv / fnsave | No            |
| FP_EOM               | 表示为x87类型的uOP，只标记在x86的EOM上，用于表示当前执行的非控制类x87指令，用于更新FIP/FCS， FOP等寄存器，无论当前x87指令是否引入unmask excep；从手册看，FOP的更新应该对应存在unmask excep的指令，这里的时间对应"fopcode compatibility mode"(sdm.Vol1 8.1.9.1) | No            |
| MMX_BOM              | 表示指令为MMX的指令，只标记在x86的BOM上，用于表示更新处理器内部的rename寄存器，表示ST0-ST7都不是有效的x87浮点数值(MMX只用于整形数据的处理) | No            |
| WAITTRAP             | 表示为x87指令的wait类指令，在模型中用于判断cr0.TS \| cr0.MP是否set，需要触发"device not available"的#NM中断 | No            |
| INTR                 | 表示当前标记的uOP也可以被中断打断，而不需要是x86的指令边界   | Yes，间接使用 |
| INTR_N               |                                                              |               |
| NOLIT                |                                                              |               |
| DBLLIT               |                                                              |               |
| IGNBOM0              | 从代码来看，该标志只用于sti指令，当该标志置位，且irq_delay[0] = 0，则当前core如果要产生中断需要等待两个EOM标志后，才可以触发中断(irq_delay[0] == 0)；意味着sti指令执行后有一条x86指令无法触发中断；这符合sdm.Vo2 "STI指令的描述"(==IF flag set, core begin respoding to external, maskable interrupts after the next instruction is executed==) | No            |
| IGNBOM1              | 类似于IGNBOM0，这个标志针对SS段的处理                        | No            |
| WRITEFC1             | 表明x87指令执行后会影响fsw.FC1；sdm.Vol1 8.1.3.3中描述大多数x87指令会影响FC1(代表指令是否会出现#IS(数据栈的上、下溢))，FC320表示condition code。在当前实现中，FC1 / FC320有实体寄存器对应；sdm中FC3210属于fsw的一部分 | Yes           |
| WRITEFSW             | 表明x87指令执行后会更新fsw。在当前实现中，fsw被分成了3部分，TOP用于进行数据堆栈的rename部分(RENAME)，FC3210用于条件码和状态(FC320 / FC1)，其他部分用于异常表示和8087的兼容busy位(FSW)；带有此标志的uOP表示需要更新FSW部分，mask为0x80fff | Yes           |
| SAVEFLA              | 表明x87指令存在memory操作数，用于当指令执行后，将memory操作数的ds/dp存入FDS/FDP寄存器 | No            |
| WRITEMXCSR           | 带有此标志的uOP表明会更新MXCSR寄存器。在当前实现中，MXCSR寄存器没有进行split。该寄存器用于XMM类指令的处理 | Yes           |
| FP                   | 表示为x87类型的uOP，只标记在x86的BOM的uOP上<br />用于根据cr0.TS/EM，检查Core是否触发"device-not-available" \#NM中断 | No            |
| NI                   | 对于SSE类型的指令带有该标志，只标记在x86的BOM的uOP上<br />用于根据cr0.TS(1)/EM(0)的情况下，Core触发"device-not-available"#NM中断 | No            |
| TAKEN                | 带有该标志的uOP表示"branch hint"，通知bpu该uOP大概率会跳转，直接预测为taken | Yes, only     |
| BR_FENCE             | 未使用                                                       | N.A.          |
| MS_NO_PRED           | 未使用；==从名字猜测表明MSROM中的branch uOP不需要进行预测==  | N.A.          |
| SETSCORE             |                                                              |               |
| READSCORE            |                                                              |               |
| ENDMSCHUNK           | 未使用；==从名字猜测表明MSROM解码到该标志后，后续的uOP需要等待下一个cycle才能继续解码== | N.A.          |
| IMMNONE              | 未使用；==从uOP序列来看，主要用于ESP压栈后的ESP新值计算，表明当前uOP没有立即数域== | N.A.          |
| IMM16                | 未使用；==主要用于lea /fxchg指令，但是没有看懂为什么用在这些地方== | No            |
| INS_XLAT_TEMPLATE    | 带有此标志的uOP表示需要trigger MSROM，trigger MSROM的同时会产生一条"nop" uOP传递到后端进行处理，详见"frontend.md"部分 | Yes, only     |
| END_OF_LINEINE       | 未使用；==只在fnstsw指令中发现了此标志的使用，但是原因未知，猜测可能和sdm.Vol2 fstsw/fnstsw中的"IA32 Architecture Compatibility描述有关"== | No            |
| ESP_FOLD             | 带有此标志的uOP表示会进行esp的offset patch处理，与stack engine相关，详见"frontend.md"部分 | Yes, only     |
| FUSE_OPT_1-21        | 用于表示uOP之间的micro-fusion关系，详见"frontend.md"部分     | Yes, only     |
| TMP_LAM              | 未使用                                                       | N.A.          |
| TMPPP                | 未使用                                                       | N.A.          |
| RMW_LAM              | 对于使能了"laminate_rmw"的Core，如果uOP包含该标志，则表示当前uOP与前面的uOP之间需要在解码后拆开为多个uOP；该标志仅仅是一个decode是否如何看待uOP间micro-fusion关系的标志 | Yes, only     |
| NOSCORE_FLAGRENAMING |                                                              |               |
| BEGIN_OF_FRAME       | 未使用                                                       | N.A.          |
| EOF                  | 用于时序模型中的stat统计的标志；没有发现实际意义             | Yes, only     |
| INF                  | 未使用                                                       | N.A.          |
| S2PAIR               | 模型中有功能代码，但是uOP序列中没有使用。从功能看，带有此标志的uOP，第2个操作数(src(1))，如果是SSE类型的寄存器，则XMM0->XMM1, XMM1->XMM0，两两组队根据当前src(1)值进行寄存器名字交换 | No            |
| BOTOP                | 未看到功能实现，在uOP序列中大量使用，目前无法猜测出具体功能  | No            |
| EOTOP                | 未看到功能实现，在uOP序列中大量使用，目前无法猜测出具体功能  | No            |
| QUESTIONABLE         | 未使用                                                       | N.A.          |

## uOP列表