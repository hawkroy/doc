# L2流水线分析

------

## 变量说明

- missq.state

  当前entry的状态

- missq.set

  entry refill完毕后，填入的L1 cache的set

- missq.paddr

  L1 cache miss的paddr

- missq.inst

  是否是icache的miss

- missq.memcnt

  魔数，代表什么时候memory data done

- missq.cachecnt

  魔数，代表什么时候cache data done

- missq.cpuid

  当前entry中请求的cpuid，可能是本地，也可能是来自于remote

- missq.req

  当前entry中需要发送的请求类型，来自于L1 cache的lsq的req请求

- missq.ack

  发送的请求是否有收到ack信息

- missq.cache_status

  当前entry需要返回到L2 的cache status或是L1的cache status，这个在cache状态设置不确定的情况下使用，比如MESI的情况

- missq.qid

  当前entry在missq中的编号

- missq.data_directory

  当前L2 cache block缓存的需要snoop的dcache列表

- missq.inst_directory

  当前L2 cache block缓存的需要snoop的icache列表

- missq.data_intervention_set

  已经发送了intervention req的dcache

- missq.inst_intervention_sent

  已经发送了intervention req的icache

- missq.data_ack_received

  已经收到的intervention ack的dcache，对于snoop到的dcache为I的情况，则

- missq.inst_ack_received

  已经收到的intervention ack的icache

- missq.intervention_type

  missq需要发送的intervention的request type

- missq.L2_changed_blk

  当前访问的L2 cache block需要被修改状态的，通常需要做snoop或是refill

- missq.L2_replace_blk

  当L2 cache出现miss时，需要替换的cache block

- missq.L2_replace_paddr

  当L2 cache出现miss时，被替换的paddr

- missq.L2_replace_way

  当L2 cache出现miss时，被替换的way

- missq.wait_for_reissue

- missq.wait_for_response

  0： 表示不需要response

  1： 表示L1和L2之间的coherence不sync，正在更新directory

  2：表明当前response已经返回，需要重新reprobe L2

- missq.invn_match_i1

- missq.invn_match_i2

- missq.read_local_l2

  在进行本地L2 cache read，表明正在访问L2 cache

- missq.modify_l1

  remote snoop需要snoop l1并改变L1状态

- missq.wtbkq_checked

  在进行L1 snoop的时候，需要check的addr是否与writeback queue有conflict

- missq.conflict_checked

  在进行L1 snoop的时候，需要check的addr是否与miss queue有conflict

- missq.memread_sent

  是否对memory controller发出了读请求

- missq.memwrite_sent

  是否对memory controller发出了写请求，实际已经代表写出去了

- missq.scache_refilled

  对于L2 miss的情况，是否在进行L2 block refill的处理

- missq.lsconten

  l1 load/store contention



- st.lasti[MQ_STATE_NUM]

  表示上一个T在pipeline stage的是missq 中的哪个entry；通常，当设置lasti之后，lasti指名的missq entry已经transmit到下一个状态



- cp.refill_set

  当前 l2cache refill的是哪个set，此时refill已经完成

- cp.refill_way

  当前 l2cache refill的是哪个way，此时refill已经完成

- cp.refill_blk

  当前 l2cache refill的是哪个cache block，此时refill已经完成

- cp.refill_bitmap

  当前 l2cache refill的block哪部分data ready，对于l2cache来说，这个为0，全部都是ready

------

## basic function

- scache_probe

  

------

## missq sequence

实际上，这个部分并不属于L2 Cache的pipeline的一部分，在Loongson设计中，这部分算作是Core Bus Interface的一部分，Core Bus Interface用于连接Un-Core部分，包括NoC和共享Cache。在目前的Loongsom设计中，L2 Cache算作是Un-Core部分，并且是Slice化的。这部分的Design并不是pipeline的设计风格，而是偏向FSM的状态机控制。介于此，这里采用表格驱动的方式来描述各个状态转换条件

## wtbkq sequence

每个T都会检查writeback queue，已决定当T是否可以进行writeback

- arbitrate wrbkq entry，获得一个需要writeback的entry；仲裁的算法采用带有spin功能的round-robin算法

- 获得当前writeback queue的empty slot个数

- 获得当前要进行writeback的entry的paddr，并决定请求发送给哪个Core

  - 发送到local

    - missq.state == MQ_READ_L2 + same cache block
      
      - 处理同一个cpu的missq和wrbkq请求之间的forwarding情况(当L1 cache 和L2 cache coherent不sync导致，L1 miss, L2 hit，因为L1 writeback then readback)
    
        当missq read L2读取l2 cache block发现not coherent(L2 directory有，但是依然l1 miss)，那么missq req需要等待writeback回写(stable状态)(writeback req可能正在NoC上路由)，设置wait_for_response=1
    
        对于wrbkq entry来说，如果当前missq正在去读l2 cache，那么等待；如果有missq 已经在等待writeback 回写，那么直接将结果回写入l2 cache，并release wrbkq，更新credit。 ***实现本身与HW可能有出入***
      
      - 串行处理wrbkq 和 missq对于l2 cache的写入/读出处理
      
      当目前有missq req在read local l2 cache的时候(持续4T) +  读取的cache set与writeback的cache set相同(可能read miss，replace整好是writeback的req)，或者l2 cache port已经被占用(当T的missq read和wrbkq write)，那么writeback不能issue, issue_wrbk = 0
      
  - (missq.state == MQ_MODIFY_L1 || missq.state == MQ_L2_MISS) + same cache block
  
    相当于是missq已经snoop了l1 cache，两种可能情况：
  
    - writeback由snoop本身引起
  
      这种情况，在loongson设计中只有l1i会出现，更新inst_ack_received变量，l1d不走writeback queue通道，而是直接修改missq entry上的信息
  
      - writeback由l1 自发产生(replace)
  
        更新missq的directory entry
  
      release wrbkq，更新credit
  
    issue_wrbk
  
    - 经过L2_write_delay写入到 l2cache
    - release writebak queue entry
    - 更新WRBK channel credit
    - 标记占用scache port
  
  - 发送到remote
  
    - 当前的writeback由被remote的snoop请求触发
  
      将当前的wrbk entry的信息直接forwarding给missq entry，并标记missq entry MQ_EXTRDY；release wrbkq entry，更新credit
  
    - 当前的writeback请求由当前的L1自身导致，且home不在local
  
      通过NoC的WTBK channel发送req，并release entry，更新credit