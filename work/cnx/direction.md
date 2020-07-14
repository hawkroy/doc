## 目前 HW正在学习的一些内容

### Core part

- AMD(zen1/zen2), intel skylake, ZX cpu参数对比	-- mac/ricky
- SMT                                                                         -- mac希望有c-model，他们大概率基于ixxxx-model进行修改，且有详细的文档，对于我们来说，需要投一些人看这个问题
- Uop cache / early trap   -- mac，希望CHX005加入，这个目前我们没有看到太多的机会
- CPU uarch的问题
  - enlarge PL2 to 512K
  - multi-level BP: hard to enhance miss penalty without uop
  - one more BR unit    -- mac timing可能不收敛
  - multi group RS with more RS entries
  - one more AGU for store
  - retire比issue多对performance的影响
  - dual table walker

### Uncore part

- NENI   -- michael
- MESH -- michael / tony
- DRAM带宽 / OPI, ZPI互联带宽 / latency / PCIe优化       -- CHX003

### problem

- 进行2D显示的时候，ricky觉得与主CPU性能有关，尝试通过测试分析CPU哪个部分影响最大     --  ricky测试后认为是整体微架构性能不够，不是某一部分的问题。这个我们是否有机会进行分析，如何做？