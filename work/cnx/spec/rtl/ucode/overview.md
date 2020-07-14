rom_inst_defs.txt	定义了ucode中的汇编最终的二进制格式，这个格式会体现在最终的编译文件master.lst (这个类似于编译后生成的list文件，表示了rom中的 二进制组织结构和反汇编)中

master_inst.txt		定义了所有ucode汇编的指令opcode和基本的汇编format，就是汇编语法的组织结构

rom_params.txt	  定义了汇编语法层面一些助记符于二进制格式中某些域的值间的对应关系，比如hw reg, reg ctrl field, size field等

汇编的基本步骤

1. 使用C的预处理器将所有的宏展开，汇编器仅仅处理内建的名字和数值
2. opname(来自于master_inst.txt)定义了unit/op1/op2，直接copy到二进制格式的对应位置
3. 每条ucode的指令类型值信息(包括operand等)直接copy到二进制格式的对应位置

```asm
xADD.S32 EAX, ECX, ECX
;step1: preprocessor, xADD.0 g16, g17, g17
;step2: xADD -> 0010, 010010,00- (bits) per master_inst.txt
;step3: g16 -> 16, etc, reg_ctrl field of ROM inst generated, etc, all rom bits filled in
```

asm format

```txt
template INST_MNENIC ::= <FIELD, TYPE>* [RCTRL] DOT_T1_SIZE DOT_R;
```

