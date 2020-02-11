# or1k core with myhdl in python

## overview

使用orpsocv2作为参考工程，使用python中的myhdl进行工程重构，目的研究是否可以混合不同抽象级别的代码，如何在已有的RTL中进行代码抽象，形成新的model。为后续构建Core的model提供经验参考

------

## or1200 Core Features

- 3类指令集

  - ORBIS32/64, basic instruction, 32bit aligned on 32bit boundary, operating 32bit or 64bit data
  - ORVDX64, dsp/vector extend instruction, 32bit aligned on 32bit boundary, operating 8/16/32/64 bit data
  - ORFPX32/64, float point instruction, 32bit aligned on 32bit boundary, operating 32bit or 64bit data, 32bit/single precision, 64bit/double-precision

- 32bit or 64bit linear address with implemented physical address

- inst format: 2 src(or 1 src/1 immed) + 1 dst

- memory address mode:

  R<sub>base</sub>+16bit signed immed(load/store) / 26bit signed immed(branch)

- 32或是16个通用寄存器大小(可以shadow)

- 可选delay slot (CPUCFG.nd bit specific)

- 支持cache和mmu

- 默认使用big-endian，HW可以实现little-endian(SR.lee bit specific)

- 支持精确异常/中断，异常/中断处理必须按照指令流顺序处理；使用fast-context switch模式(通过shadow register方式)支持快速exception处理，exception可以嵌套