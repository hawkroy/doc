[TOC]

## Fusion的分类

处理器中有如下两种fusion机制，对于哪些情况下可以fusion，这个完全由处理器的Uop的指令格式决定。对于模拟器中的uop指令格式在uISA中进行描述

### micro-fusion

多个uop占用同相同的物理资源，这样在decode/rat/rob/retire阶段都占用减少的资源，到了backend执行时会被split成多个uop，送到多个执行单元进行执行；这部分的处理是编码在uop的属性中，为静态信息，哪些micro-fusion会被开启，由模拟器中的 fused_opt的开关决定。目前模拟器中支持如下种类的micro-fusion功能

| fusing bit (start from 0)  [enable by config] | description                                                  |
| --------------------------------------------- | ------------------------------------------------------------ |
| FUSE_OPT_1    [1]                             | esp_folding push<br />push：<br />        zero = ista ss: 0(rsp + (-1)\*op_size)  BOM, ESP_FOLD<br />        zero = std op_src<br />        rsp = lea ss: 0(rsp + (-1)\*op_size)   EOM, FUSE_OPT_1 |
| FUSE_OPT_2    [1]                             | esp_folding pop<br />pop：<br />        dst  = load ss: 0(rsp + 0)    BOM, ESP_FOLD<br />        rsp = lea ss: 0(rsp  + 1\*op_size)   EOM, FUSE_OPT_2 |
| FUSE_OPT_13  [0]                              | 当前model中没有使用                                          |
| FUSE_OPT_8    [1]                             | store_address和store_data间融合，针对int类型store-data<br />mov %reg, (mem)：<br />       zero = ista seg: disp (base + index*scale)   BOM<br />       zero = std src_reg   EOM, FUSE_OPT_8 |
| FUSE_OPT_14  [1]                              | 类似于FUSE_OPT_8，但是针对于float类型store-data<br />movss %reg, (mem)：<br />       zero = ista seg: disp (base + index*scale)   BOM<br />       zero = fstd src_reg, 0  EOM, FUSE_OPT_14 |
| FUSE_OPT_3    [0]                             | fp<--->int的转换                                             |
| FUSE_OPT_4    [1]                             | laminate_rmw (load-op-store)feature, load-op part            |
| FUSE_OPT_5    [1]                             |                                                              |
| FUSE_OPT_6    [1]                             |                                                              |
| FUSE_OPT_17  [1]                              |                                                              |
| FUSE_OPT_20  [1]                              | tickel execute, 与ret指令相关                                |
| FUSE_OPT_19  [1]                              |                                                              |
| FUSE_OPT_11  [1]                              | 于FUSE_OPT_14的使用位置基本相同，都是ista / fstd  FUSE_FLAG的序列，没有看出更细微的差别 |
| FUSE_OPT_12  [1]                              | laminate_rmw (load-op_store)feature, load-op / store-imm     |
| FUSE_OPT_15  [1]                              | tickel execute, 与ret指令相关；与FUSE_OPT_16配合使用         |
| FUSE_OPT_16 [1]                               | return indirect，与ret指令相关                               |
| FUSE_OPT_7    [0]                             |                                                              |
| FUSE_OPT_9    [0]                             |                                                              |
| FUSE_OPT_18  [0]                              | call direct，与call指令相关                                  |
| FUSE_OPT_10  [0]                              |                                                              |
| FUSE_OPT_21  [0]                              |                                                              |

#### ESP Folding

#### Laminate RMW

### macro-fusion

多个X86指令在解码时被翻译为一条uop，这样在decode/rat/rob/retire阶段都占用较少的资源，即使到了backend执行时，也不会被重新split成多个uop；这部分的处理是在模拟器的decode阶段完成，其模拟器中只支持2条X86指令之间的macro-fusion。目前模拟器中支持如下种类的macro-fusion功能

## Fusion在模拟器中的实现

