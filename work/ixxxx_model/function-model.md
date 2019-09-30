# Function模型

## Register的定义

```c
// 定义uop ISA层级的寄存器
DEFREG(A, V, Y, B, C)
  /*
   * A: register enum number
   * OTHERS: not used in current model
  */
// 定义uop ISA层级的寄存器，但是是DEFREG的一部分，如ax/al/ah与eax/rax的关系
DEFSUB(A, V, B, C, D, E, F, G, H)
  /*
   * A: register enum number
   */
DEFALIAS(A, B)
  /*
   * A/B：alias register num number
   */
```

## Segment  Flag定义

```c
// 定义x86 manual中GDT/LDT descriptor中的bit定义
DEFSEGFLAG(A, B, C)
  /*
   * A: flag enum number / enum bit number
   * B: bit position number
   * C: 相当于bit-mask
   */
//EX: DEFSEGFLAG(Type, 0, 0xf)
```

具体的bit定义参看function-model.xlsx的seg_flag的sheet

## SimRunner

```c
struct SimRunner 
{
  SimRunnerPrivate simrunner_private;
  UopRegval regs[UOP_REG_MAX];
  map<addr, val> msrs;
  int lastfault;
  PhysAddr pip[2];
  uint8 instbytes[16];
  int inst_length;
  UopRegvalue uop;
  Uop uop;
  uint32 reg_back_ptrs[UOP_REG_MAX + 3];
}；
```

## x86反汇编字符串规则

@表明后续的符号是一个缩写代换符号

%表明后续的符号是一个有明确意义的符号，无需作进一步的代换

代换符号含义

| 代换符号@Symbol | 含义                                                         |
| --------------- | ------------------------------------------------------------ |
| c               | 通过alias.Opcode返回相应的condition code (EFLAGS)            |
| a               | 指令有displacement，类型为long型，通过alias.Disp获得         |
| d               | 指令有displacement，类型为long型，通过alias.Disp获得         |
| g               | x86指令为一系列opcode合集 {指令格式相同，但是使用不同的opcode表示不同指令}，通过alias.Opcode获得<br />包含的opcode合集为：<br />add/or/adc/sbb/and/sub/xor/cmp/rol/ror/rcl/rcr/shl/shr/sar/<br />not/neg/mul/imul/div/idiv/inc/dec/subr/divr/bt/bts/btr/btc |
| i               | 指令有Immed，类型为Quad，通过alias.Immed获得                 |
| j               | 指令有Immed2，类型为Long，通过alias.Immed获得                |
| m               | 指令的操作数为内存操作数，解码modr/m域获得内存操作数结构， 通过alias.Asize获得操作数的寻址长度/lock信息 |
| o               | 表明x86操作的操作数长度，通过alias.OSize获得                 |
| r               | 表明x86指令使用通用寄存器，通过alias.Reg获得具体的寄存器编码 |
| s               | 表明x86指令使用的源操作数寄存器，通过alias.Src获得， ==与r的区别== |
| t               | 表明x86指令使用的目的操作数寄存器，通过alias.Dst获得, ==与r的区别== |
| h               | 与r/s/t配合使用，如@hr等，表明使用寄存器的高8bit             |
| b               | 与r/s/t配合使用，如@hr等，表明使用寄存器的低8bit             |
| w               | 与r/s/t配合使用，如@hr等，表明使用寄存器的低16bit            |

ModR/M域的反汇编规则

- 通过alias.Seg/Base/Index/Scale/Disp获得内存操作数的SIB/seg-selector/Displacment等信息
- 对于RSP/RBP的base寻址来说，如果seg不是SS，那么需要加入段前缀，表明换段
- 对于其他的情况，如果seg不是DS，加入段前缀
- alias.Scale的编码是2幂编码，目前x86支持的scale是1/2/4/8，所以scale分别对应0/1/2/3；当scale==4的时候，代表一种特殊的寻址方式——RIP寻址，此时的displacement由alias.Immed2提供

## x86解码过程描述

### x86解码过程中用到的变量说明

返回值：32bit变量，高16bit表明hit在哪个URom中，低16bit表明URom中的数据下标

| 变量名          | 说明                                                         |
| --------------- | ------------------------------------------------------------ |
| bytes           | 表明当前指令的结束位置，与start的差值为当前指令的长度(包含指令的所有信息) |
| start           | 指令的开始位置                                               |
| opstart         | 指令opcode的开始位置                                         |
| eip             | 指令的有效ip= lip-lip[SEG_BASE]                              |
| longlongaliases | 参见<u><alias寄存器的结构></u>                               |
| longaliases     | 参见<u><alias寄存器的结构></u>                               |
| bytealiases     | 参见<u><alias寄存器的结构></u>                               |
| neednip         | 表明是否需要下一条指令的IP，如果需要，则通过alias.Immed进行传递；这个标志对于需要改变程序流的指令是必须的 |
| saveptr         | 保存解码过程中用到的所有的byte，这些byte                     |
| rex_byte_exist  | 表明当前指令是否包含rex前缀，包含此前缀的指令都是long-mode下的指令 |

### 解码规则

TBD