[TOC]

## Allocate特性

- 每周期支持最多setting_width(4)个uop的renaming和allocate的处理
- memory renaming和register renaming机制
- partial eFLAG和 partial register的处理
- 支持branch的checkpoint的机制(setting_br_checkpoint || setting_periodic_checkpoint)
- 多种分配rob/load buffer/store buffer的算法支持(static-partition / dynamic sharing)
- src operand的rob read port的分配算法
- 支持lock predictor

## Allocate的仿真流程结构

![alloc_stru](dia/alloc_stru.jpeg)

对于allocate stage来说，主要的工作在于完成backend各种硬件资源的分配，根据数据流分析的依据，将指令按照true-dependency的方式进行依赖分析，这里的分析主要包括两个部分：寄存器依赖和内存依赖(memory renaming)。同时，完成指令集架构上的特定优化策略。而在硬件时序上，allocate stage并不复杂，只完成了两件主要的事情：

- 读取idq中的解码后的Uop
- 完成uop的资源分配和重命名工作

在模拟器中，allocate阶段主要实现了如下的功能：

- 各种backend资源的分配，包括(ROB/branch color/LB/SB)
- lock指令的预测——针对lock load
- 寄存器的重命名——包括src / dst operands
- partial eflag / register的处理——对于X86指令集，某些通用寄存器允许局部读、写
- 内存重命名——memory renaming——包括load / std, load / sta之间的依赖关系处理
- 针对X86指令集语义的特定优化

针对于SMT的处理器来说，allocate stage是一个共享的stage，所以有大部分的功能集中于如何选择一个phythread用于当前周期执行allocate的逻辑。

所以，对于allocate stage主要集中于功能实现，和 SMT的线程切换、管理功能，而不重点分析时序逻辑结构

## Allocate的Pipeline结构

TBD

## Allocate Pipeline在模拟器中实现

对于allocate stage来说，功能的实现大于对于性能和时序的实现，这里主要总结下模拟器中使用的一些SIMQ，这些SIMQ的大小和latency反映了allocate stage内部的一些时序情况

| SIMQ name                             | Size                         | Latency          | Type | Location   |
| ------------------------------------- | ---------------------------- | ---------------- | ---- | ---------- |
| q_bigflush_reclaim[tid]               | bigflush_latency+1           | bigflush_latency | Uop  | ROB->alloc |
| q_alloc_to_idq[tid]<br />\<not used\> | setting_width(4)             |                  | N.A  | N.A        |
| q_inserted_uops[tid]                  | UOP_NSRC(10)*2               | 0                | Uop  | internal   |
| q_alloc_block[tid]                    | setting_width\*max_fused_uop | 0                | Uop  | internal   |
| q_wake_thread[tid]                    | 1                            | wake_latency     | bool | internal   |
| q_rsrc_avail_to_idq_tsel[tid]         | rsrc_idq_size                | rsrc_idq_latency | bool | internal   |
| q_uops_stalled_in_alloc[tid]          | alstall_size                 | alstall_latency  | bool | internal   |

**<u>表中size / latency的说明</u>**

- bigflush_latency = setting_bpmiss_latency(30) - setting_fetch_to_alloc_latency(16) - setting_alloc_to_exec_latency(8) = 6
- wake_latency = setting_alloc_rsrc_avail_compute_latency(2) - setting_alloc_clock(2) = 0
- rsrc_idq_size = (setting_alloc_rsrc_avail_to_idq_tsel_latency(4) / setting_alloc_clock(2))+2 = 4
- rsrc_idq_latency = setting_alloc_rsrc_avail_to_idq_tsel_latency = 4
- alstall_size = (setting_allocate_tsel_alstall_latency(4) / setting_alloc_clock(2))+2 = 4
- alstall_latency = setting_allocate_tsel_alstall_latency = 4

## Allocate的SMT管理

allocate stage对于SMT的系统来说是一个共享的stage，这个stage的phythread管理主要完成两方面的工作

- idq_read的phythread选择

  每周期选择一个phythread用于读取最多setting_width(4)个uop到allocate stage进行处理

- allocate的phythread选择

  每周期选择一个phythread用于进行最多setting_width(4)个uop的allocate

### idq_read的phythread处理

#### Phythread的仲裁策略

在选择哪个phythread可以读取IDQ时，模拟器使用的策略如下：

- phythread priority parking机制 (local_thread_priority)

  当allocate遇到阻塞idq read的情况时，将下一次首先进行仲裁的phythread设置为阻塞idq read的phythread；否则按照下面的round-robin策略，将当前T选择出来的phythread+1作为下一次的仲裁开始phythread

- round-robin仲裁策略

  从local_thread_priority的phythread开始轮询所有的phythread，直到找到一个符合条件的phythread

  - 对于setting_periodic_checkpoint(1)的配置，查看当前T是否处于checkpoint的recovery周期，如果是，当前T直接选择对应的phythread
  - 当前phythread的IDQ有新的uop解码出来
  - 当前phythread的allocate stage存在没有allocate的uop——已经从IDQ读取，但是没有allocate完成
  - 当前phythread有可分配的resource——从allocate stage传递过来的信号

  当上面的条件无法同时满足时，则按如下策略选择phythread(条件只要满足其中之一即可)

  - 当前phythread的IDQ有新的uop解码出来
  - 当前phythread的allocate stage存在没有allocate的uop——已经从IDQ读取，但是没有allocate完成

  当找到一个符合条件的phythread后，更新phythread priority parking机制

#### idq_read的工作条件

当模拟器冲裁出一个idq_read的phythread后，要检查当前phythread的条件是否可以继续进行idq_read

- idq_read_active——当前是否处于allocate的mis-predict的flush周期内

  当backend发现一个branch出现预测错误的时候，会立刻flush frontend和allocate stage。但是flush信号从backend传递到frontend需要一定的delay，所以在这个过程中，allocate不能读取IDQ中的uop——属于obsolete的uop

- 之前读入allocate阶段的uop已经全部被allocate(can_read_new_uops)

- 读入到allocate阶段的uop(num_uops_in_alloc_block) == setting_width(4)，这里不计算fused uop——实际就是会占用ROB单元的uop

- 如果fxch_alloc_restriction(1)，那么fxch指令必须当前allocate的最后一条uop

- 如果当前周期内含有branch uop，那么处理的branch uop个数必须小于setting_restrict_branch_alloc_per_clock(0)

### allocate的phythread处理

原则上，allocate阶段的phythread会选择上一个周期进行IDQ读取的phythread，但是，有两个特殊的情况需要考虑：

- 当前的allocate stage被某一个phythread独占——这个独占主要在于allocate对于某些特殊情况可能要动态插入uop，这时会stall idq_read
- 某个phythread的硬件资源(rsrc_avail)刚刚可用(上一个T不可用，本T可用)

## Allocate Stage分配的资源

在Allocate阶段会进行所有backend的硬件资源分配，这些资源主要包括

| resource                                                 | size                                                         |
| -------------------------------------------------------- | ------------------------------------------------------------ |
| store buffer                                             | setting_rob_partitioned(1)<br />       setting_num_sb(32) / num_active_phythreads<br />else<br />       setting_num_sb(32) |
| load buffer                                              | setting_rob_partitioned(1)<br />       setting_num_lb(48) / num_active_phythreads<br />else<br />       setting_num_lb(48) |
| rob                                                      | setting_rob_partitioned(1)<br />       setting_rob_size(128) / num_active_phythreads<br />else<br />       setting_rob_size(128) |
| rob_block                                                | rob_size / setting_width(4)                                  |
| uaq(uop allocation queue)                                | 由pb文件指定其大小，在目前的配置中为24                       |
| uoptags                                                  | setting_uoptags_partitioned(0)<br />       setting_rob_partitioned(1)<br />             setting_num_uoptags(setting_rob_size(128)-nthread(2)*23) / num_active_phythreads<br />       else<br />             setting_num_uoptags<br />else<br />       phytid[0] = setting_num_uoptags<br />       phytid[!0] = 0 |
| br_checkpoints<br />\<setting_br_checkpoint\>            | setting_br_checkpoint(0) / nthread                           |
| periodic_checkpoint<br />\<setting_periodic_checkpoint\> | setting_periodic_checkpoint(4) / nthread                     |

- **<u>ROB</u>**
- **<u>load buffer</u>**
- **<u>store buffer</u>**
- **<u>uoptags</u>**
- **<u>branch color</u>**

## uISA的Uop指令格式



## 功能实现

**<u>IDQ read</u>**

当allocate阶段冲裁出来一个phythread进行IDQ的读取后，这个stage完成的功能相对非常简单

- 从IDQ中读出uop，将其放入到allocate stage的q_alloc_block中，这个SIMQ可以认为是内部的一个stage buffer
- 如果当前配置setting_allocate_tsel_perfect(1)仲裁策略，那么当前T轮询所有符合idq_read条件的phythread

**<u>Resource Update</u>**

更新allocate阶段所有处于stall状态的phythread(alloc_sleep_reason)，如果等待的硬件资源已经可用，那么将对应的phythread唤醒，将其加入到后续的allocate stage的arbitration。并将当前的phythread的资源可用状态(rsrc_avail)通知到idq_read阶段。这里，存在3中可能情况：

- phythread有资源可用，直接跳过

- phythread因为某种资源而stall

  统计当前T等待的唤醒资源状态，如果资源已经变得可用，那么加入唤醒逻辑，并表示当前phythread处于唤醒过程中；如果依然不可用，那么继续等待

  - ROB

    - ! setting_rob_shared(0), static_partition,   当前phythread对应的logic thread处于active，且有rob空间

    - setting_rob_limits(0), 静态设置每个phythread的rob大小， 同上

    - setting_rob_dynamic(0), 根据IPC情况动态分配，同上

    - setting_rob_shared(1), shared，不区分彼此容量

      这种情况下是否可分配的方法：

      每个phythread有一个reserved的rob容量(min_rob_alloc)，统计其他phythread的rob使用情况——只统计超过min_rob_alloc的部分，如果shared的总量 - 超出的总量 - (min_rob_alloc*phythread-1)还有容量，那么可以分配

  - LB

    - ! setting_rob_shared(0), static_partition,   当前phythread对应的logic thread处于active，且有lb空间

    - shared，不区分彼此容量

      这种情况下是否可分配的方法：

      每个phythread有一个reserved的lb容量(min_rob_alloc)，统计其他phythread的lb使用情况——只统计超过min_rob_alloc的部分，如果shared的总量 - 超出的总量 - (min_rob_alloc*phythread-1)还有容量，那么可以分配

  - SB

    - ! setting_rob_shared(0), static_partition,   当前phythread对应的logic thread处于active，且有sb空间

    - shared，不区分彼此容量

      这种情况下是否可分配的方法：

      每个phythread有一个reserved的sb容量(min_rob_alloc)，统计其他phythread的sb使用情况——只统计超过min_rob_alloc的部分，如果shared的总量 - 超出的总量 - (min_rob_alloc*phythread-1)还有容量，那么可以分配

  - RS

    - ! block_alloc_rs(0), allocate rs in block mode——同一个T的需要allocate的uop都可以进入RS(这里只考虑当前allocate有多少个uop，而不考虑是否是否占用RS entry)，只考虑单个uop的情况
    - block_alloc_rs_only_remaining(0), 只考虑当前剩下的未分配的allocate uop(num_uops_in_alloc_block)
    - block_alloc_rs(1), 考虑当T需要allocate的所有uop(orig_num_uops_in_alloc_block)

    判断的方法：

    - ! setting_rob_shared(0)

      - setting_sched_thread_partition(0), scheduler shared， 根据当前的scheduler arbitration策略(setting_schedule_policy{POLICY_THREAD_UNAWARE})，对于，thread_unware，则直接查找所有的scheduler的占用情况，查看是否还有容量可以分配
      - ! setting_sched_thread_partition, static partition, TBD，在看scheduler的时候需要注意以下

    - shared，不区分彼此容量

      每个phythread有reserved的rs容量(min_rob_alloc)，统计reserved的容量和当前已经动态分配的容量，如果还有容量，可以分配

  - IDQ_EMPTY

    当前有新的uop在IDQ中

  - SCOREBOARD

    MSROM中的标有SETSCORE的uop已经从pipeline中retire了

  - BR_COLOR(br_checkpoint)

    - 当前T没有处于mis-predict的flush周期中
    - 如果处于mis-predict的flush周期中，需要查看下一个branch_color是否valid

    mis-predict的flush周期只影响当前T的allocate，意味着flush周期为1T

- phythread等待的资源已经可用，正在处于唤醒的过程中

  - 唤醒逻辑已经唤醒了phythread(q_wake_thread ready)，那么清除stall状态
  - 唤醒逻辑还没有ready，那么继续等待

**<u>Allocate-stall Check</u>**

allocate stall的检查，这里主要是检查allocate stage对于各种backend的硬件资源的分配是否可以满足当前T中需要进行allocate的uops的个数。具体的操作如下：

- 对于使能block stall check的情况

  - block_alloc_sb(1) | block_alloc_lb(1) | block_alloc_uaq(1)

    判断当前周期的空余entries个数是否<setting_width(4)

  - block_alloc_rs(0) && setting_bypass_uaq(1)

    判断当前rs中的空余entries个数是否<当前T需要allocate的uop个数(block_alloc_rs_only_remaining)，将状态直接置为BLOCK_RS

- 对于每一个uop而言(包括fused uop)

  - 当前的allocate stage没有stall且处于active状态
  - 处理的uop个数<=setting_width(4)，这种case不会发生？
  - 不存在block stall的情况(基于上面介绍的block stall check)或是某个uop终止了allocate stage(last_uop_ended_allocation_block)
  - 当前thread没有可用的uop用于allocate且没有需要动态插入的uop，那么结束本次allocate，对于perfect的策略，那么设置当前phythread为IDQ_EMPTY
  - 对于! dead_at_rat的uop
    - 对于MSROM中的uop，如果带有READSCORE的属性，且串行的uop还没有retire，那么需要stall，设置为STALL_SCOREBOARD，当前phythread进入sleep状态
    - 设置enable_fp_partial_flag_stall(0)，如果当前uop是merge_fc320_fc1，且fc1和fc320的writer(至少有1个valid)不同，设置stall_fp_partial_flag
    - 设置allocate_restrict_num_fcw_writes(1)，如果当前的dst_reg == FCW0，且fcw_writer_num == num_fcw_writes_allowed(7)，设置stall_fcw
    - 设置allocate_block_rob(0)，对于按照block方式(rob_size/setting_width)进行分配的ROB，如果当前rob entry不够，设置为stall_brob，当前phythread进入sleep状态
    - 对于非micro/macro-fusion的指令，rob的空闲数量 < mrn_mov_generated的动态uop + rob_req(1)的uop个数，设置为stall_rob，当前phythread进入sleep状态
    - 

**<u>Lock Predict</u>**

**<u>Resource Allocation</u>**

**<u>Memory Dependency Check</u>**

**<u>Memory Renaming</u>**

**<u>Register Renaming</u>**

**<u>branch handling</u>**



