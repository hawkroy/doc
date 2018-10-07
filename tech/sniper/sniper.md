# Sniper Preliminary

## Sniper概述

sniper分为pin-tool和standalone两种模式：

- pin_tool

  整个simulator作为pin-tool存在，利用pin的多线程支持，可以支持multi-thread的benchmark仿真，sniper-2.0版本以前的主要仿真方式

- standalone

  sniper-3.0开始支持mutli-process的benchmark仿真，首先利用pin-tool生成各个benchmark的仿真trace，然后借由named pipe将trace信息传递给独立的simulator程序

这两种方式的仿真核心使用相同的仿真代码，可以认为只有外部的wrapper的仿真环境不同而已

## Sniper目录结构

Sniper

+-----Benchmarks

​	用于保存所有测试用的Applications的测试集

+-----Config

​	sniper仿真器的所有可配置参数的配置文件集合

+-----Include

​	sniper仿真器对外的API头文件，主要是simapi和linux的perf_event定义；支持dynamic使用linux的perf和papi？

+-----Scripts

​	python脚本，用于处理仿真器的执行，结果收集和分析；这里主要是simapi的脚本实现

+-----Sift

​	sniper的序列化格式，用于构建多进程的仿真环境，和standalone模式配合使用

+-----Standalone

​	sniper模拟器的独立运行模式，模拟器本身不依赖于Pin环境，通过Sift和Pin进程进行通信

+-----Pin

​	sniper模拟器的pin模式，模拟器本身内嵌入Pin环境执行

+-----Test

​	sniper模拟器的单元测试环境

+-----Tools

​	用于进行仿真结果分析的python工具集，里面主要包括两类工具，cpi-stack和viz可视化工具

+-----Common

​	sniper模拟器的仿真核心代码

​    +-----Config

​	配置文件解析的实现

​    +-----Fault_injection

​	" 错误注入 "的实现，用于模拟故障产生？？

​    +-----Misc

​	一些库和基础类实现，用于特定和方便的功能实现，杂乱。主要包括仿真时间的模拟(subsecond-time)、lock、log等

​    +-----Network

​	网络模型仿真

​    +-----Sampling

​	采样模型实现，便于通过采样的方式从sniper模拟器中获得实时的仿真结果；配合sim-point使用的模型？？

​    +-----Scheduler

​	OS 调度的抽象实现

​    +-----Scripting

​	sniper模拟器的python脚本集成实现(C++侧)

​    +-----System

​	除了Core，Memory-subsystem和Network外，构建仿真系统必要的一些支持仿真模型

​    +-----Trace_frontend

​	standalone模式下用于解析Sift格式数据的trace前端，代替了Pin的功能

​    +-----Transport

​	目前不知道做什么，NoC模拟？？

​    +-----User

​	同步API，暂时不清楚作用

​    +-----Performance_model

​	各种Core性能仿真模型的设计

​        +-----Branch_predictors

​		各种branch predictor策略的实现

​        +-----Instruction_tracers

​		用于进行指令trace，并分析指令执行规律的trace工具

​        +-----Performance_models

​		性能仿真模型，主要包括core、cache、DRAM等的模型，都不是cycle-accurate模型

​            +-----Core_model

​		用于描述处理器执行组织关系，比如执行单元延时，bypass网络关系等

​            +-----Micro_op

​		sniper的uop定义，用于将x86指令翻译为对应的uop序列

​            +-----Interval_performance_model

​		interval_simulation模型，具体参见论文

​            +-----Rob_performance_model

​		加入rob模拟，更加准确的Core模型

​            +-----One_ipc_model.h/.cc

​		简单的单周期IPC=1的Core模型

​    +-----Core

​	Core结构和功能模型，主要包括Core内部的组织结构，重点是memory-subsystem，还有指令相关功能的仿真，如syscall、thread、topology等；性能仿真由Performance_model指定

​        +-----Memory_subsystem

​		Memory子系统的功能模型

​            +-----Cache

​		基本的Cache功能模型

​            +-----Cheetah

​		Cheetah的Cache仿真模型，具体查看Cheetah的介绍

​            +-----Directory_schemes

​		基于Directory机制的Cache一致性功能实现

​            +-----Dram

​		内部的Dram的接口功能实现

​            +-----Fast_nehalem

​		???

​            +-----parametric_dram_directory_msi

​		某种Cache Coherency的实现方式

​            +-----pr_l1_pr_l2_dram_directory_msi

​		某种Cache Coherency的实现方式

## Sniper类族谱

### 仿真类

- Simulator

  总的仿真对象的集合，包含了所有仿真时需要的仿真对象，是个Singleton的单件模式

- FastForwardPerformanceManager

- MagicServer

- HooksManager

  系统的若干状态通知管理类

- SyscallServer

  系统的系统调用管理类

- SyncServer

  系统的多个Core仿真核的同步管理类

- ThreadManager

- TraceManager

- ClockSkewMinimizationManager

- ClockSkewMinimizationServer

- CoreManager

- StatsManager

- TagsManager

- Transport

- DvfsManager

- FaultinjectionManager

- ThreadStatsManager

- SimThreadManager

- SamplingManager

- RoutineTracer

- MemoryTracker

- InstructionTracer

- Fxsupport

- PthreadEmu

### 多线程类

- 多线程实现环境

  Interface: _Thread

  ​	Function:

  ​		virtual void run() = 0

  ​	Impl:  

| PinThread     | Pin环境下的多线程实现方式           |
| ------------- | ----------------------------------- |
| PthreadThread | Linux Pthread环境下的多线程实现方式 |

- 可以多线程化的类

  Interface: Runnable  

  ​	Function: 

  ​		virtual void run() = 0

  ​	Impl:   

| CoreThread  | 仿真Core某个physical thread的类 |
| ----------- | ------------------------------- |
| SimThread   | 仿真网络中的收发包线程          |
| Monitor     | 用于检查Sift的连接的monitor线程 |
| TraceThread | 用于解析Sift的前端trace模型     |

每个Runnable的实现者都会include一个Thread结构，用于创建可执行的host thread，然后调用Thread的run方法从而执行Runnable的run方法

## Sniper关于时间的模拟

sniper模拟器中的时间模拟使用SubsecondTime进行模拟，不同的时间结构由不同的类代表，若干时间类之间的关系如下所示

- SubsecondTime      / subsecond_time_t(subsecond_time_s) （用于C链接的struct）

  以10-15秒(FS)为基础(当作单位1)，使用uint64_t进行计数，可以认为基础频率为1000GHz，代表了绝对时间信息

- ComponentPeriod

  描述了某个Component的工作频率，本身以m_period(SubsecondTime)为单位，实际代表了在某个Freq下，m_period的绝对时间步进长度。比如，某个Component频率设置为200MHz，则m_period = Subsecond_1S(1000GHz) / 200MHz = 5000，表明这个Component的步进长度为5000，换算为具体频率的时候使用Subsecond_1US/m_period换算为MHz

- SubsecondTimeCycleConverter

  用于使用Cycle数进行描述的时间表示，内部的m_period(ComponentPeriod)表示一个Component的周期频率。所以对于一个以Cycle描述的componet，其经历的绝对时间为Time_Abs = cycle * m_period；将某个绝对时间换算为Cycle时则是 Cycle = Time_Abs / m_period

- ComponentBandwidth

  用于描述Bandwidth固定的Component，这里的Bandwidth固定指的是时间为单位的固定，比如20GB/s。内部通过m_bw_in_bits_per_us(uint64_t)描述bandwidth。如果发送Xbits，则delay_TimeAbs = X * SubsecondTime_1US / m_bw_in_bits_per_us

- ComponentBandwidthPerCycle      

  用于描述使用Cycle表示固定Bandwidth的Component，比如20GB/cycle。内部使用m_bw_in_bits_per_cycle(uint64_t)和m_period(ComponetPeriod *)描述，一个描述固定bandwidth值(Cycle表示);一个表示Cycle的步进长度。如果发送Xbits，则delay_TimeAbs = X * m_period / m_bw_in_bits_per_cycle

- ComponentLatency

  用于描述使用Cycle表示固定Delay的Componet，比如3Cycle。内部使用m_fixed_cycle_latency(uint64_t)和m_period(ComponetPeriod *)描述，一个描述固定delay值(Cycle表示)；一个表示Cycle的步进长度。则对于固定delay = T的Component，则delay_TimeAbs = delay * m_period

- ComponentTime

  表示某个绝对时间delay_TimeAbs或是以Cycle描述的delay_TimeCycle计算到某个Component的绝对时间的表示。内部通过m_time(SubsecondTime)和m_period(ComponetPeriod *)描述，一个描述经历的绝对时间值；一个描述Cycle的步进长度。两者描述的是相同的时间，计数单位不同
