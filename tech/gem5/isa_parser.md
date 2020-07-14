# ISA Parser程序分析

Gem5仿真模拟器支持多种ISA指令集，为了更好的适应多种ISA指令集的解码处理，Gem5设计了以`.isa`结尾的DSL，用于完成各种ISA指令的解码和语义定义。这里，首先需要搞清楚，这个DSL需要完成的功能都包括哪些，这里罗列如下：

1. 完成machine code到虚拟仿真指令的decode

   需要清晰定义machine code的二进制数据有哪些指令解码域，如opcode, src(0)，src(1)，dst，immed等；同时需要清楚这些域的组合如何映射到对应的仿真指令上；如依据opcode的值定义分别生成不同的虚拟仿真指令

2. 完成每条虚拟仿真指令的语义定义

   在定义虚拟仿真指令的语义的时候，要完成的主要有如下的工作：

   - 操作哪些寄存器，寄存器的类型是什么
   - 执行哪些具体的操作
   - 操作数的来源，在哪个core的execution environment上执行指令语义

**Gem5设计的DSL是type-aware的，这个意思是说在设计不同的虚拟仿真指令的语义的时候，语义本身已经明确定义了对应操作数的数据类型，所以在后续的操作过程中，所有的语义操作都是基于ADT(abstract-data-type)的类型进行操作。这个是Gem5 DSL最大的一个特征；同时，Gem5的语法设计上借助了大量的python语法，借助于python的`eval`语句可以实现高阶的反射特性，这一点是我们自己设计DSL的时候可以借鉴的地方**

### ISA的语法元素

- 直接输出到C++文件的部分

  `output <Field> {{ code literal }}`，其中Filed只能是header, decoder, exec之一，分别对应到C++文件的header part, decoder part和exec part ；code literal为c++代码，直接写入到生成的c++文件中

- 直接转换为python的parser可执行的部分

  `let {{ code literal }}`，其中的code literal为合法的python语句，parser解析的时候，对于其中的code literal代码可以通过`eval`进行执行，**这需要parser部分进行较好的context的管理**。这种设计给ISA Parser带来了极大的灵活性，很多无法通过形式化语法定义的东西都可以借助于这类"插件编程"的方式进行解决，比如对于x86 ISA来说，所有的uop的定义都是采用let的方式进行定义的，这样可以将ISA Parser进行最大的功能扩展

- decode语义定义部分

  decode的定义采用了类-C的switch-case的语法结构，用于描述machine code翻译为不同虚拟仿真指令的过程。不同的指令，可能在指令语义和执行上有不同的差别，这部分通过format前缀和code literal进行进一步的定义说明。

  ```c
  // form-1
  decode BITFIELD {
      CASE_val : {format_name}::inst_name({{code literal}}, other_param);
      ....
      default : {format_name}::inst_name({{code literal}}, other_param);
  }
  
  // form-2
  // default will impose on all statements in decode block
  decode BITFIELD default : {format_name}::inst_name({{code literal}}, other_param) {
      CASE_val : {format_name}::inst_name({{code_literal}}, other_param);
      ....
  }
  
  // form-3
  decode BITFIELD {
      format <format_name> {
      	CASE_val : inst_name({{code_literal}}, other_param);
      	CASE_val : inst_name({{code_literal}}, other_param);
          CASE_val : {format_name2}::inst_name({{code_literal}}, other_param)
          format <format_name3> {
              CASE_val : inst_name({{code_literal}}, other_param);
              CASE_val : inst_name({{code_literal}}, other_param);
          }
      }
  }
  ```

  decode block中用到的BITFIELD来自于另外一个语法结构

  `def bitfield BITFIELD c-part-variable`，这种定义在转换的时候，会将BITFIELD的部分替换为c-part-variable的语义变量，c-part-variable来自于machine code的一部分

- 指令语义定义部分

  语义部分定义主要包括如下几个方面：

  - format的定义，format定义了每类不同的指令在最终c++中的组织模板格式；format基于若干的template来实现，主要包括：

    1. header_output： 对应指令的声明
    2. decoder_output： 指令中除execute()函数外的所有类方法定义
    3. exec_output： 指令execute语义的定义
    4. decode_block：指令在decode sequence中的使用，即按照decode block中的逻辑定义后每个case在c++中执行的语句，通常为new virtual_inst_type(params)

  - template的定义，每个template定义了一类可以处理的模板结构，在生成对应的c++文件的时候，通过`InstObjectParams`结构进行正则匹配替换，目前主要包含如下几类keyword用于在c++生成时进行模板替换

    - class_name：指令类的名字
    - op_class：指令所属的操作类别
    - mnemonic：指令名字
    - base_class：指令类的基类
    - flags：指令执行后执行的flag设置代码
    - op_wb：dst或是dst_memory的写入代码
    - op_rd：src或是src_memory的读取代码
    - op_decl：临时操作变量的定义，比如从src中进行值的读取
    - code：指令实际语义的含义

    模板中最为关键的时exec_output的定义，其通常的形式为

    ```c
    xxxxx::execute(....) {
        Fault fault = NoFault;
        
        %(op_decl)s;
        %(op_rd)s;
        
        %(code)s;
        
        if (fault == NoFault)
        {
            %(op_wb)s;
        }
        return fault;
    }
    ```

    对于每一条待翻译的指令，通过format都会生成一个InstObjParam的结构体用于进行模板代换时使用，这个InstObjParam是在解析decode block中的指令语句时build的，其主要有两个参数code, *flag；code为decode block中该条指令的语义描述（对于x86不存在），flag代表了该条指令的一系列的flag，比如是否是浮点操作、向量操作等

  - operand的定义

    如上所述，Gem5中每个operand都是带有类型的，这样便于Gem5在生成代码的时候使用不同类型操作数的函数进行处理。operand的定义如下：

    ```c
    def operands {{
            'SrcReg1':       foldInt('src1', 'foldOBit', 1),
            'SSrcReg1':      intReg('src1', 1),
            'SrcReg2':       foldInt('src2', 'foldOBit', 2),
            'SSrcReg2':      intReg('src2', 1),
            'Index':         foldInt('index', 'foldABit', 3),
            'Base':          foldInt('base', 'foldABit', 4),
            'DestReg':       foldInt('dest', 'foldOBit', 5),
            'SDestReg':      intReg('dest', 5),
            'Data':          foldInt('data', 'foldOBit', 6),
            'DataLow':       foldInt('dataLow', 'foldOBit', 6),
            'DataHi':        foldInt('dataHi', 'foldOBit', 6),
            'ProdLow':       impIntReg(0, 7),
    }};
    ```

    左边为语义中使用的operand名字，右边为对应的类型定义，右边的类型是一个函数，展开后变为python的tuple格式：

    ```c
    (RegType, default_sizetype, BITFIELD, flags, priority)
    // RegType: 指明寄存器的类型，比如是整型、浮点、控制类，这个被利用在python的parser中，parser中的类型为'RegType'+Operand，如IntReg在parser中重定义为IntRegOperand
    // default_sizetype: 寄存器的默认数据类型大小，这个是C++中对应的类型
    // BITFIELD: machine code中对应的位域
    // flags: C++中使用的flag，用于表明当前operand在C++中的若干类型信息，比如是否是整型、浮点、控制类寄存器等
    // priority: 表明当前的一个优先级，目前还不太明白作用
    ```

    由此可见，对于Gem5来说，所有的操作数在编译生成代码的过程中都是已知类型的，并且在编译期就是确定的，所以在编译的过程中，不同类型的操作数的操作可以基于不同的模板来进行处理。operand在parse中处理完成后，用于生成exec_output的template中的`op_decl`, `op_rd`, `op_wr`的模板生成。

### 关于x86中的x86->uop格式的转换

原始的Gem5的parser目标只是用来完成machine code的语义解析，这种对于RISC类型的ISA已经足够了，每条RISC类型的machine code分解为特定域后，对应的语义是明确的。然而，这种处理在面对CISC的x86 ISA的时候是不能完全描述的，原因是x86的machine code经过翻译后会分解为一系列的uop sequence，每个uop对应RISC中的一条指令。如此，parser要完成的工作就从只需要进行解码后的语义分析转变为两个部分：1、解码为uop序列；2、基于每个uop定义语义信息。

为此，Gem5在原来parser的基础上进行了扩展，充分利用了parser可以嵌入执行python代码（let block）的能力。扩展的方式如下：

1. 原始的parser功能依然保留，但是对于x86来说大部分的format都不是通过format / template的方式定义，而是更多采取内嵌python脚本的方式生成format / template，外围的format / template仅仅是一个占位的作用，解码后的x86指令被称为macroop，每个macroop不具备exec的能力，在每个macroop的constructor中会构造该macroop对应的microop sequence；microop sequence与macroop的对应的关系通过*.py的指令描述文件进行描述
2. 语义描述部分完全由microop的语义描述替代，而所有的microop的语义和结构信息全部在\*.isa的文件中通过`let block`进行描述，有多少种microop就会在\*.isa中定义多少个microop的类。同时，*.py中描述的每一类macroop的翻译过程用到的每条microop sequence又有自己单独的语法结构用来描述operand，其operand与Parser中实际的operand的定义描述也是通过`let block`的形式以python脚本的方式进行映射的。

整个x86解码和语义翻译过程可以通过下图进行表示：

![x86 parse flow](F:\document\note_github\project\gem5\dia\isa_parser_x86_flow.png)

通过观察Gem5中的x86->uop的.py文件，可以确定如下几个信息：

1. Gem5定义的uop ISA是一个两操作数的RISC ISA，通过ld/st来进行memory operand的操作。对于ld/st来说，其memory operand的操作数通过`seg, sib(riprel), disp`或是`seg [1, t0, t1(immed)]`  两种形式进行表示
2. 所有的uop(microop)的语义全部在isa文件中以python的形式固化好，是hard-cod的。**Gem5依然是一个ISA级别的解析和语义产生工具，而不是一个Uop级别的语义产生工具**