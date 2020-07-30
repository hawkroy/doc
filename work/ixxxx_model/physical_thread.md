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

### Physical Thread的冲裁策略

对于SMT系统来说，Core pipeline中的每个stage在每个执行cycle都要从已经ready的phythread中进行冲裁，已决定当前cycle哪个phythread可以在stage进行执行，且每个stage可能使用的仲裁策略不一定相同。

调度策略实现的基本数据结构

下面介绍的phythread的仲裁策略，基于当前的数据结构进行实现

```c++
struct ThreadingState {
    int phytid;					   // 当前T正在处理的phythread
    int save_g_phytid;			    // 当前T开始仲裁前的初始phythread，这个值由global_priority或是g_phytid[all_core][phythread_tid]决定
    int count;					   // 当前T正在查看的phythread当前T已经处理了多少请求
    int phytids_used_count;			// 当前T已经有多少个phythread已经处理过
    int phytids_used[MAX_THREADS];	// 当前 T 哪些phythread已经处理了，哪些还没有处理
    float min_used;				   // 资源使用情况，只记录占用量最小的 (float)(used_resource[phytid]/limit_resource[phytid])
};

// static variable in each stage handle function
// used for saving initial selected phythread in next cycle
static int g_phytid[MAX_PROCES*MAX_CORES][MAX_TID_COUNTERS];

// APIs
// each stage thread arbitration start point (per selected)
// policy: thread arbitration policy
// phytid_counter: start point phythread index
#define THREADING_BEGINNING(policy, phytid_counter, condition, used_res, limit_res)
	// initial part
	static int g_phytid[MAX_PROCS*MAX_CORES][MAX_TID_COUNTERS];
	ThreadingState ts;
	ts.count = 0;
	ts.phytids_used_count = 0;
	ts.min_used = 0;
	// select the initial phythread
	if (phytid_counter >= 0)
    	ts.phytid = g_phytid[current_core->id][phytid_counter];
	else                                                                                 	 ts.phytid = curc->global_thread_priority;
	// currently cycle initial selected phythread
	ts.saved_g_phytid = ts.phytid;

	// POLICY_ICOUNT, arbitration by used resource limitation
	if (policy == POLICY_ICOUNT) {
    	for (int ii = 0; ii < num_phythreads; ii++)
   	  		ts.phytids_used[ii] = 0;
	}
	else {
		while ((policy != POLICY_TIME_INTERLEAVE) // not POLICY_TIME_INTERLEAVE
               &&
               // not meet ready condition
               !(condition)
               &&
               // still available thread to select
               (ts.phytid != (ts.saved_g_phytid + num_phythreads - 1) % num_phythreads)) {
    		ts.phytid = (ts.phytid + 1) % num_phythreads;
    	}
	}
	do {
		if (policy == POLICY_INDEPENDENT)
			ts.count = 0;
        else if (policy == POLICY_ICOUNT) {
        	ts.min_used = setting_max_rob_size + 1;		// phythread min used rob
            for (int i = 0 ; i < num_phythreads; i ++) {
            	int real_phytid = ((setting_num_inactive_phythreads <= 0) || (nthread < 2)) ? i : curc->thread[i].last_phytid;
                if (!ts.phytids_used[i] // not used in current cycle
                    &&
                    // resource is the minium
                    (((float) used_res[real_phytid] / (float) limit_res[real_phytid]) < ts.min_used)) {
                	ts.phytid = i;
                    ts.min_used = (float) used_res[real_phytid] / (float) limit_res[real_phytid];
                }
        	}
        	ts.phytids_used[ts.phytid] = 1;
        	ts.phytids_used_count++;
    	}
        ASSERT1(ts.phytid < nthread, ts.phytid);
        int phytid = ts.phytid;
		while (condition)
            action_code....

// each stage thread arbitration end point (per selected)
#define THREADING_ENDING(policy, phytid_counter)
        // pair with THREADING_BEGINNING while (condition)
        // search & check next phythread
		ts.phytid = (ts.phytid + 1) % num_phythreads;
    // pair with THREADING_BEGINNING do {
	} while (((policy == POLICY_ICOUNT) && (ts.phytids_used_count != num_phythreads) && !ts.count)
        	||
            ( (policy != POLICY_ICOUNT)
              &&
              // other policies
              ( (ts.phytid != ts.saved_g_phytid)
                // still available phythread to select
               	&&
              	( (policy != POLICY_PHYTHREAD_INTERLEAVE) || !ts.count)
               	  // POILICY_PHYTHREAD_INTERLEAVE: already handle request 
                  &&
               	  (policy != POLICY_TIME_INTERLEAVE)
               	  // POLICY_TIME_INTERLEAVE, done
                  // others:
               	  //	POLICY_INDEPENDENT
               	  //	POLICY_PHYTHREAD_ROUNDROBIN
               	  // 	POLICY_OLDEST_FIRST
                  //	POLICY_LEAST_RECENTLY_SERVICED
                  // 	POLICY_REPLICATE
               	  //	POLICY_THREAD_UNAWARE
                )
             )
         	);
	if ((ts.count > 0) && (phytid_counter >= 0))
        // update next cycle initial select phythread
		g_phytid[current_core->id][phytid_counter] = ts.phytid;
```

目前模拟器中支持如下的phythread的仲裁策略

- POLICY_INDEPENDENT (0)

  遍历所有phythread，只要满足条件就可以执行

- POLICY_PHYTHREAD_INTERLEAVE (1)

  找出满足条件的phythread，处理一次；如果没有处理，遍历 phythread直到找到一个满足条件的phythread

- POLICY_PHYTHREAD_ROUNDROBIN (2)

  当前T遍历满足条件的phythread，按照round-robin的方式

- POLICY_ICOUNT (3)

  找到使用资源最少 (目前看到的是rob的利用率) 的phythread，处理一次

- POLICY_OLDEST_FIRST (4)

  没有实现

- POLICY_LEAST_RECENTLY_SERVICED (5)

  TBD， 在FSB阶段使用

- POLICY_REPLICATE (6)

  没有使用

- POLICY_TIME_INTERLEAVE (7)

  只处理当前T初始选择的phythread，无论条件是否满足

- POLICY_THREAD_UNAWARE (8)

  在scheduler初始化调度数据结构时使用