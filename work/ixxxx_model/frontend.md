[TOC]

## Frontend特性

- 每T最多解码出4条uop(setting_fe_width)
- 每次Fetch读取16B(mtf_fetch_bandwidth)的数据
- 每次Decode可以处理16B(mtf_decode_bandwidth)的数据
- 每次Decode可以处理的对齐的chunk数据(mtf_decode_aligned_chunks)
- 支持fetch_fusion(ild之后完成)
- branch predict预测
- macro-fusion / micro-fusion

## Front-end的仿真流程结构

![frontend](dia/frontend_stru.jpeg)

frontend主要完成如下功能：

1. 当backend需要flush frontend的时候，frontend进行reset，并根据backend的要求进行flush和reset。双方交互的接口为q_beuflush[tid] SIMQ。该SIMQ中存储着所有mispredict或是需要flush frontend的uop。frontend flush前端所有的queue，并根据该uop的信息reset前端的fetch状态

2. 检查每个phythread的stall reason是否已经unblock (stall reason会在后续详述)

3. 对于SMT系统，需要根据每个phythread的状态，按照一定的冲裁算法进行fetch thread的选择；每T不一定只有一个phythread会执行fetch动作

4. frontend_fetch模拟真正的fetch动作

   - MTF fetch

     1. 读取ITLB，进行VA->PA的地址翻译；如果ITLB miss，进入PMH进行进一步处理
     2. 读取ICACHE，如果ICACHE miss，则进行L2 Cache的读取；返回的response放入q_icu_fill SIMQ中；该SIMQ表示已经fetch的ICACHE miss line data已经返回
     3. push stream-prefetch，进行指令的prefetch预测处理
     4. 进行BPU预测处理，如果预测不成功，建立投机堆栈

   - MS fetch

     X86指令需要trigger MSROM，执行MSROM的fetch动作，进行MSROM的时序模拟

   - PERFECT fetch

     完美fetch模拟，在这种情况下，不会出现itlb/icache miss以及bpu mis-predict情况，直接可以进行后续解码

5. 当fetch没有出现stall的时候，将模拟取指得到的fetch line指令送入q_fe_fetch[tid] SIMQ中，该SIMQ代表进行了ILD处理，等待进行decode的X86指令

6. 根据每个phythread状态选择decode thread，进行decode动作

7. frontend_uop模拟真正的decode动作

   - 进行指令翻译，通过XLAT的直接翻译
   - 需要进入MSROM进行取值翻译的过程——重置fetch为MS_FETCH
   - micro-fusion/macro-fusion/unlimination的处理 （fusing相关， decode-fusion)

8. 解码后的指令放入q_fe_uops[tid] SIMQ，该SIMQ代表已经进行解码的uop序列，准备进行后续的处理

## Frontend的Pipeline结构

Core的frontend是一种In-Order的pipeline结构，如下图所示：

![frontend_pipe](dia/frontend_tim.jpeg)

Frontend的Pipeline结构由3段流水线构成：

- fetch&decode pipeline
  - fetch / bpu

    根据当前phythread的lip值进行lip_next的预测(bpu)，并访问il1 cache system进行指令读取，每次读取setting_mtf_fetch_bandwidth的指令data，目前设计为16B；如果il1 cache system出现miss，那么等待il1 cache miss done后进行后续处理 (icache fetch stall)

  - ild

    完成指令的切分处理，并将切分后的x86指令存入x86 inst QUEUE中；在ild过程中，frontend存在stall的情况 (ild stall)；视配置情况，进行fusing的处理 (默认在decode处理)

  - decode

    对x86指令进行解码，翻译为对应的uop序列；对于译码较多的X86指令，则需要trigger MSROM进行进一步译码；在译码过程中完成macro-fusion / micro-fusion。在decode过程中，存在stall情况 (decode stall)

- bpu update pipeline

  当branch指令retire或是complete后，需要进行bpu表结构的更新，用于以后branch指令的预测

- frontend flush pipeline

  当branch指令在执行后，发现出现了mispredict的情况，此时需要重置frontend，从正确的位置重新抓取指令执行

## Frontend Pipeline在模拟器中实现

- Fetch & Decode pipeline

  基于frontend是In-Order的pipeline结构的特点，模拟器在实现frontend的时序结构的时候进行了抽象，将frontend抽象为2个功能模块，并根据pipeline stage和pipeline buffer设置SIMQ的大小，按照fetch-alloc-latency设置SIMQ的延迟信息，用于仿真frontend的latency

  ![fetch-decode-sim](dia/fetch-decode_sim.jpeg)

  模拟器中function与pipeline结构的function的对应关系如下：

  - frontend-fetch：实现Fetch、BPU、ILD、Decode (包括MSROM和XLAT解码)的功能
  - frontend-uop：实现micor-fusion、macro-fusion功能

  SIMQ用于实现pipeline的延迟和Buffer结构：

  - q_fe_fetch[tid] SIMQ

    用于模拟读取且切分的X86指令，实际存储使用uop格式；

    - size：设计为frontend pipeline stage和内部所有buffer结构的总大小，表明可以有多少的X86指令可以in-flight，包含3个部分：

      max_uops_per_fetch  --- 一次fetch可以解码出来的uop个数， setting_fetch_width (4) *  XLAT_UOP_NUM (4)

      max_uops_per_line  --- 一次可解码的最大uop个数，**setting_fe_width * max_uops_per_fused**，代码中这个设计应该有问题，实际应该就是setting_fe_width (4)

      max_uops_per_fused --- 一个uop最大可fused的uop个数；最大为5，实际与设置的fusing规则相关
      - 路上buffer

        pipeline stage的buffer, T= (fetch_to_alloc_latency (16) / fe_clock (2) + 1) * MAX(max_uops_per_fetch, max_uops_per_line) * max_uops_per_fused

        表明经过T这么长时间后，fetch可以进入的uop个数 (这个个数也代表了可以fetch的X86指令数)

      - fetch data buffer

        setting_fe_fetch_buffers (0) * MAX (max_uops_per_fetch, max_uops_per_line) * max_uops_per_fused

        表明读取到Fetch buffer中的Data包含的最大个数的uop个数 (这个个数也代表了最大可能的X86指令数)

      - x86 inst queue

        setting_fe_iq_size (18) * MAX(MAX_XLAT_UOPS, setting_fe_mswidth)

        表明X86 inst queue中可以解码出来的最大 uop个数，每个entry代表一条X86指令

    - latency：设置为fetch-alloc的pipeline latency—— **setting_fetch_to_alloc_latency(16) - setting_fe_clock(2) - setting_alloc_clock(2)**；这个设计应该有点问题，就是setting_fetch_to_alloc_latency 

  - q_fe_uops[tid] SIMQ

    用于模拟X86解码后的uop输出，和micro-fusion、macro-fusion处理后的uop

    - size：

      依据不同的管理方式

      - uops：设计为uop_queue的size (uq_size = setting_fe_uq_size/nthread)，uq_size * max_uops_per_fused
      - chunk：按照一次最大解码的X86指令个数管理，即uq_size一个entry表示一次最大的X86解码个数，uq_size * MAX(max_uops_per_mtf, max_uops_per_chunk) * max_uops_per_fused

    - latency：0，在这个queue中不体现任何时序信息

- bpu update pipeline

  对于bpu update的pipeline的仿真，模拟器的实现相对比较简单

  ![bpu-update](dia/bpu-update_sim.jpeg)

  SIMQ 用于实现pipeline的延时结构：

  - q_bp_update[tid] SIMQ

    用于保存执行完成的branch uop，并根据branch uop的branch info进行bpu表结构的更新

    - size：为实际ROB大小(因为每个进入ROB的指令都可能是branch uop)，setting_max_rob_size (按照thread进行均分)
    - latency：在模拟器中，这个延迟设置为一个固定的延迟，为setting_update_bp_latency (14)

- beuflush pipeline

  对于beuflush的pipeline的仿真，模拟器的实现相对比较简单，且是在frontend的cycle函数的一开始进行调用

  ![beuflush](dia/beuflush_sim.jpeg)

  SIMQ 用于实现pipeline的延时结构：

  - q_beuflush[tid] SIMQ

    用于保存执行完成且mis-predict的branch uop，frontend使用这个SIMQ来获取需要重置的IP，并根据新的IP信息进行指令读取

    - size：设置为latency + 1，说明这个SIMQ本身是一个用于延时实现的Queue
    - latency：latency在模拟器中为固定延迟 setting_bpmiss_latency (30) - setting_fetch_to_alloc_latency (16) - setting_alloc_to_exec_latency (8) [- setting_dispatch_latency (6)]；这里的设置反映了一条branch uop最理想情况下通过core pipeline发现branch mis-predict需要flush frontend的时间

## frontend的status管理——基于phythread

在SMT中，frontend需要基于phythread进行管理，而backend不需要。对于每一个phythread，其frontend的status切换如下：



## phythread的仲裁机制



## 功能实现

### Fetch

### Instruction-Lenght-Decode (ild)

### Decode

#### Fusion

#### MSROM Fetch

### Flush