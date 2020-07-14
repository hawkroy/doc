# CNX Core Module Design Spec Template

TODO: 需要明确下Module的规模达到多少需要写Module Spec

## Brief Description

简要描述模块的功能

## Feature Lists

罗列当前模块支持的一些功能特性，比如：

- 支持Pipe访问
- 使用多大的存储空间，SRAM组成还是DFF
- 使用何种算法

## Port Lists

罗列模块的所有输入、输出信号的类型、位宽、含义

## Data Storage

罗列Module中使用的SRAM， DFF Array

### 1. Structure Info

描述每种storage的基本结构信息，大小、关联性、bank等

### 2. Data Structure

存储的数据结构，bit的含义

### 3. Index Function

storage索引信息的生成、由哪些信号生成storage的索引信息和相关的index function

### 4. Timing Info

描述每种存储结构的读、写时序，reset时序等

## Block Diagram

模块内部子模块间的逻辑关系，简要描述子模块完成的功能

## Workflow

描述该模块实现的各种功能的流程图。模块可能完成了不同的功能，以BPU为例，有预测的流程；更新的流程等。这里着重功能的描述

## Timing Sequence Diagram

按照实际的时序划分pipeline stage，并标注不同stage中完成的功能在哪个block实现；如果模块完成了不同的功能(如workflow中描述)，则按照不同功能分段标注pipeline stage

## 子模块Spec

重复上述过程

------

# CNX Core Module Design Draft

上述的Design Spec是我们最终需要形成的文档，但是在看RTL代码的过程中，无法一次性形成上述Spec需要的所有信息，需要不停地看代码抽象，最终获得对于RTL的代码的抽象认识。这个过程中，需要我们以某种Draft的文档形式进行讨论总结，形成Design Spec文档。Draft文档主要用于建立如何看RTL，进行抽象的流程。

## 1. 模块的基本信息

属于哪个模块的子模块，可能完成的功能（基于经验判断）

## 1. Port Lists

罗列模块的所有输入、输出信号的类型、位宽、含义

## 2. Data Storage

罗列Module中使用的SRAM， DFF Array

### 1. Structure Info

描述每种storage的基本结构信息，大小、关联性、bank等

### 2. Data Structure

存储的数据结构，bit的含义

### 3. Index Function

storage索引信息的生成、由哪些信号生成storage的索引信息和相关的index function

## 3. 初步的pipeline stage划分，按照可能的pipeline stage将信号分组

大体划分pipeline stage的时序关系，归类不同的信号含义(Excel形式?)

## 4. 逐步总结control逻辑功能，修正3中的信号分组和pipeline stage划分