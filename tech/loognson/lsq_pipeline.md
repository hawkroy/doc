# LSQ 流水线分析

------

## 变量说明

- lsq.fetching

  当前entry的data在路上，可能从下一级cache获取

- lsq.ex

  当前entry发生了exception，而无法继续执行；包括

  1. TLB miss，目前实现，永远不会miss
  2. 出现新load先于老store完成的情况
  
- lsq.state

  当前entry的工作状态，是lsq entry的工作状态机

- lsq.op_load/store

  当前entry是ld/st

- lsq.byterdy

  当前entry请求的data是否ready，表明data phase done

- lsq.blk

  当前entry请求的cache block，如果为空，表明miss或是cc-violate

- lsq.req

  当前entry在cache miss的情况下，需要发送到下一层cache的请求
  
- lsq.loadspec

  表明当前的load带有投机性(先于前面的load完成)



- st.replace_delay

  当发生cache block替换时，仿真器延时的时间(这个延时为什么要加入？)

以下3个结构均为refill_packet

- st.lsq_refill

  当前core正在处理的refill请求

- st.lsq_dtagcmp_refill

  当前pipeline上dtagcmp_stage正在处理refill的请求

- st.lsq_dcache_refill

  当前pipeline上dcache_stage正在处理refill的请求

- st.lsq_wtbki

  进行writeback处理的lsq entry，是否算一个pipeline stage

- st.missi

  进行miss处理的lsq entry

- st.dtagcmpi

  dtagcmp_stage上的lsq entry

- st.dcachei

  dcache_stage上的lsq entry

- st.issuei

  addr_stage上的lsq entry

- st.lsq_head

  当前lsq的head指针，表明最老的ld/st的位置

- st.lsq_head1




- cp.refill_set

  当前替换的block所在的set
  
- cp.refill_way

  当前替换的block所在的way

- cp.refill_bitmap

  当前替换的block

- cp.refill_blk		在替换时进行更新(cache_replace)

  当前cache正在进行refill的block，这个block已经是replace过的(即已经push了writeback queue)
  
- cp.replace_paddr

  当前被替换的block的paddr

- cp.replace_status

  当前被替换的block的cache status

- cp.refill_new_status

  替换后，refill时需要填入的新cache status

- cp.last_tagset

  TBD



- refill.valid

  表明当前的refill request是否valid

- refill.replace

  表明当前的refill request需要执行replace；LOONGSON实现的不同之处在于replace不是立即发生的，而是等到refill data返回后，再次上dcache pipeline的时候发生的替换

- refill.intervention

  表明当前的refill request正在进行snoop

- refill.set

  refill request需要访问的set

- refill.paddr

  refill request对应的physical addr

- refill.cnt

- refill.req

  refill相对应的request类型，

- refill.missqid

  本次refill请求由哪个missq entry发起

- refill.ack

  当前refill是一个snoop的时候，表明snoop之后的response结果

- refill.cache_status

  当前refill是一个snoop的时候，表明snoop之后的old cache status

- refill.blk

  保存replace后refill需要填充的block或是被snoop的block

- refill.next

  refill packet的后继节点

------

## Basic Function

- cache_probe

  tag[phy_max : set_idx_max]

  set[set_idx_max-1 : 5]， 每个set 4 way，一共512个set

  blocksize = 32

  cache容量： 512\*4\*32B = 64KB

  bank的划分

  hit condition: same tag + status != I && DIRTY_I

  cc violation condition: hit condition + store_op + status = SHARED

  refill hit condition: 如果当前处理器有refill block，那么如果tag相同，且bank相同，相当于hit。refill_blk是在dtagcmp_stage进行的处理

  hit: hit condition + ! cc violation

- replace_block

  ```c
  struct cache_blk *
  replace_block(struct cache* cp, int bindex, md_addr_t paddr, int req, int inst, int intervention, int new_status)
  // cp: 当前需要进行cache替换的cache
  // bindex: 指定当前需要替换的block，-1为不指定，使用替换算法计算
  // paddr：需要替换进来的paddr
  // req: 当前进行访问的request的request类型
  // inst: 是否为icache
  // intervention: 是否是因为snoop导致的replacement（可能发生吗？）
  // new_status: 需要设置的新的cache status
  ```

  1. 使用随机算法找到一个可替换的block，如果与上一次刚刚refill的是同一个block，那么不选择上次刚刚refill的那个block

  2. 对于因为cc-violation而发生的cache miss，那么直接选择那个因为cc-violation导致的cache block

  3. 进行replace的action替换

     所有的writeback request都比实际HW提前了1T

     1. snoop request
        - 当tag match的情况下，push相关的request到writeback queue中(ELIMINATE, repl->cache_status)
        - 当tag 不match的情况下，push相关的request到writeback queue中(ELIMINATE, INVALID)
     2. replace request
        - 对于MODIFIED block，push相关的request到writeback queue中(WRITEBACK, repl->paddr, repl->cache_status)
        - 对于!INVALID的其他status block，push相关的request到writeback queue中(ELIMINATE, repl->paddr, repl->cache_status)
        - 对于INVALID block，减少writeback queue的number??，并发送到NOC上

  4. 更新被替换的cache block

     1. 非snoop
     2. snoop且tag match

     更新tag/set/status/subblock 信息，仅仅是将当前replacement block标记为invalid

  5. 保存当前被替换的cache block信息到当前cache中，当前replacement已经完成

- cache_miss

  ! next level cache serving + no exception + 

  - load： pipeline visit done + data phase not  done
  - store： pipeline visit done + data phase not done

------

## dcache sequence

1. dcache的pipeline完成，并且已经拿到了数据(针对load)，则开始准备请求writeback regfile

2. 判断dcache miss的lsq entry，有miss的则表明需要push refillq

3. 对于已经commit的store指令，经过仲裁后，判断是否可以写入到dcache；可写，更新为M

   判断条件为：

   store + data phase done + commit + arbitration done + bank not conflict with load

4. dtagcmp的stage处理，两种情况

   1. refill的处理

      - replace

        扫描lsq中所有match的entry

        - store：pipeline visit done + no release，那么直接kill掉当前entry，并标记重做(当作miss处理)

        - speculation load：writeback done + no retire，那么触发exception，并flush pipeline

      - snoop

        - 扫描lsq中所有match的entry
          - store：pipeline visit done + entry no release，则当前entry重做(按照miss处理)
          - load：pipeline visit done + miss (low level已经接收请求，并处理) + data not done，则当前entry重做(按照miss处理) 
          - speculation load：writeback done + no retire，则触发exception，并flush pipeline
        - 在当前stage，对于remote snoop发起的INTERVENTION_SHD/INTERVENTION_EXC/INVALIDATE的请求，将dcache stage的snoop结果直接返回给发起snoop的请求(ack, cache_status)，并更改miss entry的状态为MQ_EXTRDY
        - remote l1 miss或是l2 miss发起的snoop请求
          - 当前missq处于MQ_MODIFY_L1，表明需要snoop l1(由remote l1 miss发起)，根据dcache stage的snoop结果设置ack和l2 block dirty信息
          - 当前missq处于MQ_L2_MISS，表明需要snoop l1(由l2 miss 发起)，根据dcache stage的snoop结果设置ack和l2 block dirty信息

      - refill

        将refill的data直接forward给需要的load/store entry (entry必须match且在当前load data范围内(8B) + 已经完成pipeline visit)，有两个特殊情况需要处理：

        1. store： 如果当前返回的cache status是SHR，那么需要update，依然按照miss处理
        2. 对于本T冲裁需要push missq的request，则kill掉，当T的missq仲裁空拍

   2. 正常的ld/st的处理

      1. 获取指令读取的数据长度，对于MIPS而言，看来最大可读取的是8B

         计算方法：((1<<read_len)-1) << (addr & 0x7)

      2. 如果是load，且无异常，且hit block，那么直接下T writeback regfile (?这里有点问题，如果判断某个load不是跨cacheline边界的)；这样做貌似像是data提前返回给RS

      3. 判断ld/st之间的violate关系

         1. 对于load

            检查data forwarding的条件

            - 扫描所有load之前的store entry(比load老的)，看是否有data的overlap
            - 如果有overlap，当前store指令已经访问完pipeline，且地址有重叠(基于8B-align);可以看出，MIPS应该是只能做8B-align内的任意forwarding，而不能做任意地址的forwarding

            检查memory consistency的条件；按照论文说法，godson2支持SC模型

            - 扫描所有load之前的load entry，如果之前的load还没有完成writeback regfile的回写，那么其为speculation load，后面如果speculation load出现因为replacement导致miss，那么会pipeline flush

            检查load的cache hit/miss条件

            - cache hit，直接标记当前load data phase done；这里是合理的，因为上一步已经检查了store的forwarding条件，所以只要cache hit意味着
              - 有老的store可以forwarding部分data
              - 有部分data可能来自于cache本身
            - cache miss，则可能部分从store forwarding，部分需要refill填充

         2. 对于store

            检查data forwarding的条件

            - 扫描所有该store之后的指令，检查地址overlap
              - 如果match的是load指令(已经访问完pipeline的)，更新overlap的byte area为ready；如果当前的load已经写回(LSQ_WRITEBACK)或是准备写回(wrbki)，那么标记当前load有exception，准备进行exception处理
              - 如果match的是store指令(已经访问完pipeline的)，那么由新的store完成相应的forward，这部分由load上pipeline时进行处理

            检查store的cache hit/miss条件

            - cache hit，标记当前store 的data phase done
            - cache miss，标记访问的data area不ready
         
         3. 对于其他类memory指令，则直接标记为data phase done，目前没有找到类似指令

5. 进行writeback regfile的处理，writeback本身占用1T，但是有可能不是pipe，见1

6. 回收已经release的entry，实际是上个T完成的，延时到本T处理

7. 标记本T已经commit的entry(对于store来说，必须已经完成回写)为下T回收

8. 访问dtlb和dcache，两种情况

   1. refill的处理

     - replace
     
       表明当refill data done后，需要进行replace的block (LOONGSON的replace是一定发生的，采用随机替换)
     
       进行cache block的替换，并设置replace的延时为2T(这里的延时设置，对应到pipeline是怎么个行为？)
     
       被换出的block会写入到writeback queue中(比HW提前1T)
     
     - snoop
     
       表明当前refill 需要进行snoop处理
     
       针对不同的snoop request，对当前cache进行snoop，如果当前cache block为MODIFIED，则ACK_DATA，
     
       修改当前block 的status
     
       并设置replace的延时为2T(这里的延时设置，对应到pipeline是怎么个行为？)
     
     - refill
     
       将refill data填入到cache中
     
       标记bitmap，更改cache status，并表明refill的block已经done(st->refill_blk)，通过这个可以看出，一次只能有一个block处于replace的处理流程中（这一过程的处理在L2中体现，replace & refill是同时进行的，中间不能打断)
     
   2. 正常ld/st的处理
      1. 访问dtlb，如果dtlb miss，表明当前entry有exception，需要后续处理(刷新主流水线)；目前dtlb永远hit，这部分pending
      2. 访问dcache，标记访问的结果(处理cache block访问和cc-violation)，等到dtagcmp_stage进行处理

9. dcache pipeline的仲裁，优先级：refill > new issue > store-back

10. 如果new issue的不能上pipeline，则不能进行新指令issue，标记当前fu为busy