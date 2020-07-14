[TOC]

# IPC Tool Design Spec

## 目标和使用范围

本工具是一个程序特征分析的工具，试图解决如下几方面的问题：

- 程序运行与特定机器架构(architecture profile)上的**粗略**的性能评估结果
- 特定程序的自身固有的并行特性，目前主要指Instruction-Level Parallelism (ILP)
- 评估不同机器架构运行相同的程序时的性能差异
- 当在特定机器架构上的性能结果与程序固有的并行特性有差距的时候，指出可能的架构改进策略和方向

后续可能的使用范围？

- 用于找出程序是否存在线程化的可能，如果存在，如何划分线程
- 用于分析程序的是否可向量化，针对可向量化部分提供改进建议

基于工具的使用范围，工具不根据特定的机器架构特点进行实现 (比如特定的微架构设计)，只包含所有机器架构的通用参数。所以，该工具不适用：

- 不能用于分析特定架构的精确性能

## 软件结构

工具采用on-line trace的方式对特定程序的运行时分析，基于Sniper Simulator进行二次开发。整个工具分为2个部分：

- trace部分，用于分析程序的指令静态和动态信息，工具中使用Pin
- IPC 分析部分，按照trace的指令信息，形成Dynamic Dependency Graph (DDG)，并在此基础上结合特定的机器架构分析程序性能，获得性能参数结果 (Performance Profile)
- 链接部分：使用Linux PIPE机制

![software architecture](D:\doc\note\work\ipctool\dia\sw_arch.jpeg)

## 限制

- 不针对特定x86架构实现的micro operation设计，采用通用micro operation，在生成的micro operation数量、之间的关系等方面不准确
- 无法精确反映需要trigger MSROM的x86指令；这部分指令在本工具中被当作特定latency的barrier指令；指令的latency通过测试后进行拟合
- Function unit的占用模拟可能存在与实际情况占用顺序不一致的情况
- 后端的资源的占用情况无法精确反映 (后端资源都以micro operation为分配单位)；在function unit的执行上工具与实际硬件间会存在差异
- load / store对于Cache 系统的影响与实际系统中的存在顺序上的不一致
- 无法模拟load / store中的资源竞争的情况 (比如对于fill queue的竞争)
- 工具本身有consistency问题，无法保证因果关系正确
- 无法精确模拟store的行为 (不能模拟出RFO和Store data的过程)

## 支持的架构特征 (Architecture Profile)

工具支持如下的机器架构特征

- 前端
  - 分支预测器
  - fetch带宽
  - decode penalty (对于x86只要是prefix处理)
  - I-Cache的架构参数 - size / associativity / access latency
- 后端
  - register renaming & memory aliasing detect
  - ROB / RS / LSU / register file size
  - issue width / commit width
  - Function units mapping & capabilities
  - Instruction type / latency  / throughput
  - load - store forwarding
  - memory hierarchy (functional {Cache Structure / MSI coherence protocol} with simple latency)
- 支持多核结构 (UMA / NUMA)
  - 简单的ring / mesh network建模
- DRAM
  - 简单的throughput / latency配置

## 性能参数结果 (Performance Profile)

- Instruction-Per-Cycle (IPC) / IPC-Stack
- .....

## 实现

这里只描述IPC tool部分的实现。算法通过逐一扫描执行过的动态指令流信息，解码成特定的micro operation序列，生成指令间依赖关系，从而计算出每条指令“最早”能schedule的时间点，在根据当前schedule的时间点，计算因为architecture profile引入的额外的latency，从而达到模拟out-of-order执行的效果。

### 指令解码

将当前指令按照如下micro operation进行解码

1. load micro operation 
2. execute micro operation
3. store micro operation

当指令中不包含上述某一类micro operation的语义时，则相应的micro operation空缺，这3类micro operation之间依次依赖，即 store依赖execute, execute依赖load。由此，一条指令最多翻译为3条micro operation。

#### serialization指令处理

工具无法精确的获得每条指令在不同平台下的decode结果 (micro operation format对于我们是未知的)。对于trigger MSROM的指令来说，其decode的micro operation数量和执行latency取决于运行时环境，所以这部分工具无法精确模拟。一个相对简单的方式：将这些指令当成serialization指令，即这条指令在core中是串行执行的；在执行之前，rob为空，当执行结束之后，后续的指令可以进入rob。在实际的机器中，也存在真正的serialization指令，比如cpuid指令。所以，这里我们将这些情况统一处理

定义一个特殊的uop： serialization_uop，通过这个uop来处理所有需要串行处理的情况

### 算法流程

工具依照典型的Core结构，在进行算法设计时，依然采用in-order frontend, out-of-order backend的设计，但是在backend仿真中，因为不是以cycle为主进行仿真，所以backend中的resource constrain仿真有限制，会出现与实现硬件执行时序不一致的情况

仿真过程对于Core流水线的抽象如下

![pipeline-abstract](D:\doc\note\work\ipctool\dia\pipeline_abstract.jpeg)

每个被scan的指令会分别记录T_fetch, T_alloc, T_sched, T_complete, T_commit的时间，在指令一次模拟经过不同的处理路径时，使用特定的时间记录计算下一个节点的时间值

如果有更多的architecture profile的feature被加入，那么可以为指令加入更多的时间标记节点，引入更多的流水级控制函数

### main-flow

```c
map<uint64_t, uint64_t> reg_livemap;		// <reg-id, cycle> mapping for register rename
map<AddrRange, uint64_t> mem_livemap;		// <{addr, size}, cycle> mapping for memory aliasing
uint64_t core_fetch_cycle = 0;			    // instruction T_fetch
typedef struct ROB_Entry
{
    uint64_t t_leave_rs;				    // instruction leave RS time
    uint64_t t_commit;					    // instruction commit time
};
ROB_Entry rob[ROB_ENTRIES];

void ipc_analyze(DynamicInstruction *inst)
{
    // simulate fetch & decode part
    <uint64_t t_alloc, bool mispred> = frontend(inst);
    
    // get micro operations
    // also special handle for "serialization" instruction
    // 		which instruction is serialization is defined by ourselves
    // 		now, some trigger MSROM instructions will treat as special serialization instruction
    uops = decode(inst);
    
    // alloc & backend stage
    for (uop in uops) {
		uint64_t t_sched = alloc(uop, t_alloc);		// also include retire
		// get "early" schedule cycle according to dynamic dependency
		for (src in uop.srcs)
			if reg_livemap.find(src)
             	t_sched = max(t_sched, reg_livemap[src]);
		for (src_addr in uop.source_addresses)
			if mem_livemap.find(src_addr)
			   t_sched = max(t_sched, mem_livemap[src_addr]);
         // add backend resource constrains to schedule cycle
         t_sched = schedule(uop, t_sched);			    // for resource contention
         uint64_t t_complete = t_sched + get_latency(uop);   // also simulate mem system
         for (dst in uop.dsts)
			reg_livemap[dst] = t_complete;
         for (dst_addr in uop.dst_addresses)
			mem_livemap[dst_addr] = t_complete;
         // for branch mis-prediction uop, sync with frontend
		handle_branch(uop, t_complete);
         commit(uop, t_complete);		// calc instruction commit cycle
    }
}
```

### frontend

用于模拟frontend in-order latency，目前考虑I-Cache访问

```c
tuple<uint64_t, bool> frontend(DynamicInstruction *inst)
{
    static uint64_t last_fetch_line_pc;  // remember core last fetch line address
    uint64_t t_alloc = 0;
    if (last_fetch_line_pc == inst->pc & fetch_mask)
        return core_fetch_cycle;
    MemRef ref = getIcacheDelay(inst->pc);
    if (ref.hit) {
        // hit case, pipe visit
        t_alloc = core_fetch_cycle + ref.latency;
        core_fetch_cycle++;
    }
    else {
    	// miss case
        t_alloc = core_fetch_cycle + ref.latency;
        core_fetch_cycle = t_alloc;
    }
    bool mispred = BP_sim(inst);
    return tuple(t_alloc, mispred);
}
```

### alloc

用于分析architecture profile中提供的各种资源是否可以满足指令的需求，如果不满足，则计算latency；同时，也完成当前cycle的retire工作。

```c
uint64_t alloc(Uop *uop, uint64_t alloc_cycle)
{
    static uint64_t last_alloc_cycle = 0;
    static int last_dispatch_num = 0;
    static bool last_is_serialization = false;
    
    if (last_alloc_cycle < alloc_cycle) {
        last_alloc_cycle = alloc_cycle;	 // trace core alloc cycle
        last_dispatch_num = 0;
    }
    else if (last_dispatch_num == dispatch_width) {
        // exceed dispatch width
        last_dispatch_num = 0;
        last_alloc_cycle++;
    }
    last_dispatch_num++;
    
    uint64_t t_sched = last_alloc_cycle; // schedule cycle, which mean uop in rob, and dipatch to rs
    
    // check for serialization
    bool serial_alloc = is_serialization(uop) || last_is_serialization;
    if (is_serialization(uop))
    	last_is_serialization = true;
    else if (last_is_serialization)
    	last_is_serialization = false;
    	
    if (serial_alloc) {
    	uint64_t t_drain = drain_rob();		// release all resource
    	t_sched = max(t_sched, t_drain);
    }
    
    // alloc phase
    bool done = true;
    do {
        // release resource
        t_sched = update_resource(t_sched);
        done = true;
        
        // check resource alloc status
        if (rob full) {
            t_sched = max(t_sched, rob[0].t_commit);
            done = false;
        }
        // same as reg file
        else if (lq full) {
            t_sched = max(t_sched, rob[lq_head].t_commit);
            done = false;
        }
        // same as sq
        else {
        	int port = getRSPort(uop);
        	if (rs[port] full) {
            	t_sched = max(t_sched, rslink[port].t_leave_rs);
            	done = false;
        	}
        }
    } while (!done);
    
    // alloc done, now can schedule in backend
    update_counter();		// update all the resource counters
    return t_sched;
}
```

#### update_resource

工具根据当前的cycle将cycle之前各种资源回收。这里包括in-order时需要回收的各种资源(rob/reg/lsq)，也包括out-of-order释放的资源(rs)

```c
uint64_t update_resource(uint64_t end_cycle)
{
    uint64_t new_cycle = end_cycle;
    // special case: rob full, and alloc & retire in same cycle, alloc should pending 1T?
    if (rob full && (rob[0].t_commit == end_cycle))
        new_cycle++;
    while (!rob.empty() && rob[0].t_commit <= end_cycle)
        release_resource();		// decrease resource occupy counters, here only including rob/reg/lsq
    // release rs counter & update linklist
    for (port in RSs) {
        while (rslink[port].t_release_rs < end_cycle) {
            release_rs();
            release_node();
        }
    }
    // update function unit time slot, see back schedule
    return new_cycle;
}
```

#### reserved station (RS)资源的处理

RS的处理与其他的资源不同，因为RS的资源的占用是in-order的，但是释放是out-of-order的；当一条指令可以schedule的时候，对应的RS资源会被释放。工具尝试同时支持分立的RS和单一的RS。为了追踪每个RS资源的分配和释放的时间点，为此在ROB中引入t_leave_rs的时间点和双向链表的rs_next，下图为释义的算法工作流程

![rs resource](D:\doc\note\work\ipctool\dia\rs_resource.jpeg)

关于RS的处理主要包括3个调用：

- alloc stage		   如果可以占用RS entry，那么直接更新对应的RS的资源计数
- schedule stage    设置对应rob entry的leave_rs，并更新rslink的链表
- update_resource  释放已经“过时”的entry

关于分离、单一的RS的模拟

- 分离的RS可以使用各种启发式方法进行port分配，**但是port分配的函数调用必须在资源回收之后**

### schedule

这里的资源限制主要指的是对于Function Unit的竞争。这里在实际的Core中是完全out-of-order的；工具只能采用尽量贴近HW实现的调度方式，但是无法保证一致。这里采用的算法是history-based schedule。

算法：history-based schedule

- 条件：指令的latency必须是已知的，少数指令不能满足这个要求 (比如整形、浮点的div、sqrt等指令)，但是通过提前的拟合可以将这些指令的latency变为固定的；这里的指令latency指的是指令的pipe latency，即当前Function unit对于某一类的指令类型最少等待的时间

- 实现：

  ![back schedule](D:\doc\note\work\ipctool\dia\back_sched.jpeg)

- 限制：当"program order"上后序的指令在仿真时间上超前"program order"上前序的指令时，且两者在几乎相同的时间点占用Function unit，那么工具无法实现按照实际HW的方式先调度"program order"上后序的指令

  ![back schedule limitation](D:\doc\note\work\ipctool\dia\back_sched_limit.jpeg)

- 实现：在具体实现时，无需按照算法track所有的cycle的slot的占用情况，且只需要维护rob window这个窗口范围内的slot占用情况即可。工具实现的back schedule算法，不会出现time slot overlap的情况，所以只需记录已被占用的区间即可

  ```c
  list< {start, size} > occupy_table;		// function unit占用的time slot情况，按照start排序
  binary_search(occupy_table, new_rq_cycle, new_size);		// 使用二分查找是否当前的request overlap某一个{start, size}，如果overlap，那么接受这种limitation，并记录这种violation。按照经验上，这种情况也许并不多
  update_resource(end_cycle);				// 按照end_cycle的时间更新每个function unit的time slot的占用情况
  ```

### handle_branch

这里用于处理当指令出现mis-prediction的时候，需要flush前端。这里只讨论了CFG的控制算法

```c
void handle_branch(Uop *uop, uint64_t resolve_cycle)
{
    if (uop.mispredict())
    	core_fetch_cycle = resolve_cycle;
}
```

### commit

计算每条指令的commit时间

```c
void commit(Uop *uop, uint64_t complete_cycle)
{
    static uint64_t last_commit_cycle = 0;
    static int last_commit_num = 0;
    
    if (last_commit_cycle < complete_cycle) {
        last_commit_num = 0;
    }
    else if (last_commit_num == commit_width) {
        last_commit_cycle++;
        last_commit_num = 0;
    }
    last_commit_num++;
    
    rob[cur].t_commit = last_commit_cycle;
}
```

