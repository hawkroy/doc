Perf的底层交互接口

```c
// 用于perf应用程序设置对应的PMC counter
int sys_perf_event_open(struct perf_event_attr *hw_event_uptr, pid_t pid, int cpu,
                        int group_fd,
                        unsigned long flags);

struct perf_event_attr { /*
         * The MSB of the config word signifies if the rest contains cpu
         * specific (raw) counter configuration data, if unset, the next
         * 7 bits are an event type and the rest of the bits are the event
         * identifier.
         */
  		  //config[63]=0
  			//		config[62:56]指定event type，包括HW_EVENT/SW_EVENT/TRACE_EVENT
  			//		config[55:0]指定event_id，这些id都是kernel规定
  			//config[63]=1
  			//		config[62:0]指定event_id，这些id都是HW vendor规定
  			__u64                   config;
        __u64                   irq_period;
        __u32                   record_type;
        __u32                   read_format;
        __u64                   disabled       :  1, /* off by default        */ 																		inherit        :  1, /* children inherit it   */
                                pinned         :  1, /* must always be on PMU */ 																		exclusive      :  1, /* only group on PMU     */
                                exclude_user   :  1, /* don't count user      */ 																		exclude_kernel :  1, /* ditto kernel          */
                                exclude_hv     :  1, /* ditto hypervisor      */ 																		exclude_idle   :  1, /* don't count when idle */
                                mmap           :  1, /* include mmap data     */ 																		munmap         :  1, /* include munmap data   */
                                comm           :  1, /* include comm data     */
                                __reserved_1   : 52;
        __u32                   extra_config_len;
        __u32                   wakeup_events;  /* wakeup every n events */
        __u64                   __reserved_2;
        __u64                   __reserved_3; 
};
```

