# Sniper Internal

## Sniper基础

sniper分为pin-tool和standalone两种模式：

- pin_tool

  整个simulator作为pin-tool存在，利用pin的多线程支持，可以支持multi-thread的benchmark仿真，sniper-2.0版本以前的主要仿真方式

- standalone

  sniper-3.0开始支持mutli-process的benchmark仿真，首先利用pin-tool生成各个benchmark的仿真trace，然后借由named pipe将trace信息传递给独立的simulator程序

这两种方式的仿真核心使用相同的仿真代码，可以认为只有外部的wrapper的仿真环境不同而已

## Sniper目录结构

- common

  仿真核心代码，所有simulator object均在此文件夹中实现

  - core

    CPU结构仿真代码，主要包括Core内部的组织结构，重点是memory-subsystem；core主要包含指令相关功能的仿真，如syscall、thread、topology等

  - fault-injection

    进行错误注入的仿真block

  - misc

    辅助功能模块，主要包括仿真时间的模拟(subsecond-time)、lock、log等

  - network

    network controller和网络延时的模拟

  - performance-model

    性能仿真模型，主要包括core、cache、DRAM等的模型，都不是cycle-accurate模型。对于core来说，主要包括如下几个模型: 1-ipc，interval, rob-based, rob_smt-based

  - sampling

    采样模型，应该是配合sim-point使用的模型

  - scheduler

    模拟OS的仿真调度策略模型

  - scripting

    simapi的若干C++侧的实现

  - system

    顶层仿真模型，用于将各个部分组织起来

  - trace-frontend

    standalone模式下用于解析sift格式的simulator前端

  - transport

    暂时不知道?

  - user

    同步API，具体用于什么?

- config

  sniper的配置文件目录，里面提供了不同Intel微架构的配置选项，修改配置即可以针对不同的问题进行仿真

- include

  主要包含了针对Linux的perf和papi的引用，支持dynamic使用linux的perf和papi？

- pin

  PIN的wrapper环境，允许sniper以pin-tool的形式运行

- scripts

  sniper支持运行时使用python进行运行时配置，这里主要提供的是python的simapi的脚本

- sift

  sniper生成的trace格式，sniper-3.0之后使用sift格式进行standalone仿真。同时，trace文件也可以用于后续的离线仿真

- standalone

  standalone的wrapper环境，允许sniper以独立的APP形式运行，在此种模式下，必须配合sift的trace格式+前端pin-tool的tracer才能进行仿真

- tools

  仿真结束后，用于进行仿真数据分析的python脚本

## SIFT格式分析



## Sniper 仿真类分析