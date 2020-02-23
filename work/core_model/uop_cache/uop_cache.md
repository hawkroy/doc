# Micro operation Cache Presentation

------

## Trace Cache

### 背景

简化的Core执行模型——生产者(Fetch、Decoder)[Frontend] / 消费者模型(Rename, Execution, Retire)[Backend]，中间通过一个Instruction Buffer用于两者交互的queue。

![core_model](D:\doc\note\work\core_model\uop_cache\dia\core_model.jpeg)

对于Core的性能评估中最为重要的指标：IPC，实际就可以认为是Core的带宽(Throughput)，说明整体的Core的优化方向也是在带宽方向进行优化，较深的流水线深度实际上加大了每一条指令的延时(Latency)。

按照简化模型，Dispatch/Issue的宽度决定了Backend的带宽上限，且Backend提供的实际带宽主要有如下几个因素决定：

- Instruction Window (ROB / LSQ)，决定了可以发现的ILP的可能性
- Function Unit，提供了可以同时并行的指令数上限
- Physical register file，与Instruction Window配合，使得尽量多的指令可以进入到Instruction Window中

目前Core的设计趋势是加大上述的因素资源，尽量增大Backend的带宽上限

Frontend为了匹配Backend带宽，需要考虑影响Frontend带宽的影响因素，主要原因有：

- Instruction Cache hit rate

- branch predictor accuracy

   上述两个因素已经被大量研究过了，也有一些不错的算法和实现完成进行性能的提高，上述两个主要影响的是latency，对于IPC的影响是间接的

   在增大backend的带宽后(>4)，出现的frontend的影响因素

- branch throughput

   原始的BPU一次只进行一个basic block的预测，经过workload的profiling，发现对于一个basic block来说通常只包含5-6条指令，这大大限制了per cycle的fetch带宽

- noncontiguous instruction alignment

   

- fetch unit latency

