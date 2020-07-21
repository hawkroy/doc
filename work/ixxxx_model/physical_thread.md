[TOC]

## Physical Thread

Ixxxx模型中支持SMT模式，在已知的Ixxxx的实际产品中，一个物理核(Physical Core)最多支持2个逻辑核(Logical Thread)。在当前的仿真器中，为了支持HT，需要创建如下的数据结构：

- Mapping

  用于描述register的映射关系，本质属于RAT中的内容，放在这里的原因在于Ixxxx的仿真器是Function-first的仿真方式，所以投机路径上的register映射关系需要记录

- Context

  用于描述投机路径上的Physical Thread的执行路径。这些执行路径中的内容可以认为是frontend需要一直传递到backend的信息；如果backend判断当前处于错误的投机路径，那么需要将这些执行路径上的信息回滚到已知的状态。**<u>同时，意味着这些信息在实际硬件中是需要按照每个thread单独维护的</u>**

- phythread

  代表SMT中的logical thread，即硬件中的线程概念。每个phythread有自己独立的寄存器、映射表、以及其他信息；在SMT的实现中，只有frontend的硬件资源对于phythread是感知的，而backend对于phythread没有感知

- LogicalThread

  目前，不知道是否对应OS中的软件调度线程——如果是，那么这部分是模拟器的特殊实现，实际硬件不包括；如果不是，那么意味着Ixxxx的硬件中实现了部分软件调度使用的线程信息。从实现代码来看，LogicalThread代表运行于硬件上的软件线程的状态信息。LogicalThread与phythread之间的对应和调度关系，需要结合更多的代码来查看

### Mapping (RAT register mapping table)

Mapping结构反映了模拟器中的Rename机制和结构。模拟器中的Rename采用机制，其具体结构和算法如下：



### Context (phythread execution environment)

### phythread (SMT's HT)

### LogicalThread (OS thread)