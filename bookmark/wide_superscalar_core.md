# Review on "Wide of super scalar micro-architecture"

------

## Background

1. Frequency sustain constant due to leakage power (short CMOS) and temperature wall
2. Integrate more cores on single chip, but many application not thread-level parallelism [1]
3. Single core performance impact by frequency & instruction-level-parallelism
4. Instruction-level-parallelism mainly impact on more resources on back-end to exploit more independent instructions
5. Due to timing constrains, back-end organization grouped as clusters, which grouped similar function instructions to same execution unit. The common optimization is reducing inter-communication between clusters, which involving more wires(latency & power)
6. In clusters architecture, one important thing is to balance clusters, but seems no free lunch for only hardware implementation. The simple method is "Mod-N" dispatching policy, but only "Mod-3" works well on narrow issue(issue width < 4) Cores.
7. many micro-architecture feature aimed at power consumption, like "loop buffer"

## Contribution

1. Exploration on dispatching policy on wide issue(4 <= issue width < 8) Cores, by combining register write specialization and clustering method (based on "Mod-N"), can enlarge issue width and instruction window --- Need RTL verification ? 
2. Proposal two methods to optimization on Loop structure
   1. detect redundant micro-op which produce same result in every iterations in loop (Ex, only copy register from one to another), and then remove them from loop --- involving "Redundant load table"
   2. predict load will access store-queue or data-cache, but not access simultaneously, to reducing dynamic power --- using some predicting method like BPU ?

## Micro-architecture history  & trend



## References

1. "Evolution of thread-level parallelism in desktop applications", Geoffrey Blake, 2010