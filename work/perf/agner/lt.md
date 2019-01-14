# LT(latency-throughput)测试分析

### 汇编中使用的宏变量

- instruct

  被测试latency或是throughput的单一x86指令

- regsize

  测试时使用的寄存器宽度，8/16/32/64/128/256/512，default=32，legacy code使用65表示mmx register

- regtype

  使用的寄存器类型，对于regsize<=64，默认使用r；regsize>=128，默认使用v

  | 符号 | 说明                        |
  | ---- | --------------------------- |
  | r    | 通用寄存器 general register |
  | h    | ax/bx/cx/dx的高8bit         |
  | v    | 128bit以上的vector register |
  | m    | mmx register                |
  | k    | mask register               |

- numop

  非立即数操作数个数，0-2个，对于regsize >=256，则可以是3个

- numimm

  立即数操作数个数，0-2个

- immvalue

  第一个立即数操作数的值

- tmode

  | 标签 | 说明                                                         |
  | :--- | ------------------------------------------------------------ |
  | L    | latency测试                                                  |
  | T    | throughput测试                                               |
  | M    | 针对source operand是memory的指令throughput测试               |
  | MR   | 针对destination是memory的指令throughput测试，形如mov [m], r类型，有2个operands |
  | LMR  | 针对MR的形式进行latency的测试，可以有1 or 2个operands        |
  | M3   | 针对第3个source operand是memory的指令throughput测试，指令本身有4个operands |
  | L2   | 从第2个operand进行latency测试，指令本身有4个operands         |
  | L4   | 从第4个operand进行latency测试，指令本身有4个operands         |
  | A    | 进行指令测试前，首先clear eax，针对0个operands               |
  | D    | 进行指令测试前，clear eax & edx，针对0个operands             |

- blockp

  在测量throughput的时候，在一些特定的执行port或是流水线pipe上插入一些占位指令

- cpubrand

  当前测试机的vendor ID

- elementsize

  testdata定义时使用的数据bit位宽，针对向量指令，只有在指定了regval0或是regval1的时候才有效

- regval0

  第1个源寄存器的初值

- regval1

  第2个源寄存器的初值

