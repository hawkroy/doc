## Pydgin 工程组织结构 

- pydgin/

  ISS framework, should not modify. 实现了 Fetch-decode-execute的循环，支持 bitwise操作，register file， memory，系统调用的实现，jit标注.....

- parc/, arm/, \<isa\>/

  - machine.py: architecture state的描述
  - instruction.py: 静态指令结构的描述
  - isa.py: 指令编码和语义
  - \<isa\>-sim.py: 顶层执行封装

## 编译安装pydgin-parc-toolchain

1. 安装必须的编译工具

   `apt-get install stow flex bison makeinfo`

2. src/binutils/Makefile.in中关于bison和flex的设置有问题

   作者是想使用bison/flex的yacc/lex语义，使用了bison -d / flex -d，但是输出与实际不符，通过设定环境变量YACC=yacc / LEX=lex来重新定义

3. 修正源码中的texinfo文件

   1. bfd/doc/bfd.texinfo     将@colophon和@cygnus修改为@@colophon和@@cygnus
   2. ld/ld.texinfo  将@colophon和@cygnus修改为@@colophon和@@cygnus
   3. gas/doc/c-mips.texi    sed -i -e 's/@itemx/@@itemx/'
   4. gas/doc/c-tic54x.texi   sed -i -e 's/@itemx/@@itemx/'
   5. gas/doc/c-score.texi    sed -i -e 's/@itemx/@@itemx/'
   6. gas/doc/c-tic54x.texi    @code  => @code{}
   7. gas/doc/c-arc.texi         @bullet => @code{}
   8. gas/doc/c-arm.texi        删除与ARM Floating Point相关内容
   9. gcc/doc/cppopts.texi    sed -i -e 's/@itemx/@@itemx/'
   10. gcc/doc/invoke.texi      sed -i -e 's/@itemx/@@itemx/'
   11. gcc/doc/c-tree.texi      sed -i -e 's/@itemx/@@itemx/'


## Pydygin的仿真结构

- Sim仿真类

  负责初始化一个Simulator，用于指令系统的仿真，新的指令仿真系统继承自Sim

  | 可重载函数                                                  | 描述                                                         |
  | ----------------------------------------------------------- | ------------------------------------------------------------ |
  | \_\_init\_\_(self, arch_name_human, arch_name, jit_enabled) | 定义simulator，jit_enabled表明simulator是否使用jit加速，通常为True |
  | decode(self, bits)                                          | 指令的机器码，用于解析为对应的指令代码和执行函数             |
  | pre_execute(self)                                           | 每条指令执行前的callback函数                                 |
  | post_execute(self)                                          | 每条指令执行后的callback函数                                 |
  | init_state(self, exe_file, exe_name, run_argv, testbin)     | 仿真器准备执行前的初始化，用于加载可执行程序，设置可执行程序的执行环境 |
  | run(self)                                                   | 不用修改，整个framework的核心工作流程。如果必须修改，需要重新定制。目前没有看到修改的必要 |
  | get_entry_point(self)                                       | 不用修改，仿真器开始运行的入口代码部分                       |
  | target(self, driver, args)                                  | 用于pypy的rpython的处理，目前评估不会使用pypy的 JIT加速，不用修改 |

- Machine仿真类

  负责定义\<isa\>的核心执行环境，主要包括memory, register_file和reset_addr

  | 可重载函数                                                   | 描述                                                         |
  | ------------------------------------------------------------ | ------------------------------------------------------------ |
  | \_\_init\_\_(self, memory, register_file, debug, reset_addr) | 定义特定\<isa\>的执行环境，从这个构造函数看来，只支持一个memory和register_file的定义，分别设置为self.mem和self.rf；对于 register file来说，可以设置更多的register file，因为在run的核心代码中并不使用，只在每个具体的指令语义定义中使用 |
  | fetch_pc(self)                                               | 用于获得下一条指令的执行地址pc                               |

- Storage仿真类

  定义指令仿真器中用到的所有存储结构，包括register file和 memory。两个类都可以按照需求进行重载

  - register file

    用于所有可索引的register的 storage，所有的访问都通过\[ \]进行索引，下标来自于register的mapping

    对于使用不同种类和不同bit_width的寄存器，可以定义多个register file

    | 函数                                             | 描述                                                         |
    | ------------------------------------------------ | ------------------------------------------------------------ |
    | \_\_init\_\_(self, constant_zero, num_reg, nbit) | 定义可访问的寄存器文件大小，并指定每个寄存器的bit width；constant_zero表明0号寄存器不可写，为zero寄存器 |
    | \_\_getitem\_\_(self, idx)                       | 访问idx的寄存器                                              |
    | \_\_setitem\_\_(self, idx, value)                | 设置idx对应的寄存器的值                                      |

  - memory

    定义了仿真器使用的memory，按照实现的不同，分为SparseMemory/ByteMemory/WordMemory。其中ByteMemory和WordMemory表明了memory的访问方式，是否支持byte访问；SparseMemory表明memory的实现方式是否为稀疏内存实现方式，稀疏内存采用类似跳表的方式组织内存

    对于x86的系统来说，目前MMIO和IO的实际功能无法描述外(pydgin不支持device)，MSR、memory、当作memory的MMIO和IO都可以模拟 

    | 函数                                      | 描述                                                         |
    | ----------------------------------------- | ------------------------------------------------------------ |
    | Memory(data, size, byte_storage)          | 进行memory的初始化处理，指定整个memory的 address space，data指定整个memory的初始化值为多少，是否使用 byte_storage |
    | read(self, start_addr, num_bytes)         | 数据读操作的接口函数，little-endian，高地址存数据的高位      |
    | iread(self, start_addr, num_bytes)        | 指令读操作的接口函数，little-endian，**<u>从接口说明上看来，pydgin的指令memory部分不支持self-modify-code，需要进一步check</u>** |
    | write(self, start_addr, num_bytes, value) | 数据写操作的接口函数，little-endian                          |

## 新的ISA的添加方法

- bootstrap.py

  

- machine.py

  定义新的machine的仿真类

- \<isa\>-sim.py

  顶层执行封装

- instruction.py

  静态指令结构的描述，不同的指令可以存在复用的field域的定义，所有的field通过定义@property进行返回

- isa.py: 

  指令编码和语义，以及使用的register的mapping定义。其中

  ​	encodings = [ ['指令1'，'bitstring' ], ['指令n', 'bitstring'] ]	定义每条指令的指令编码

  ​	def execute_指令n(s, inst)定义每条指令的执行语义，s为machine中定义的\<isa\>执行环境，inst为解析出来的指令格式 

## x86 ISA的uop function model的结构

