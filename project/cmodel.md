开发计划

1. 使用perf等PMC工具测试现有硬件与Intel的差别，获得第一手测量数据，各种metrics
   1. 已经测试了IPC，不同测试集下IPC的变化规律
   2. 不同x86指令的执行latency/throughput/uops/port-binding关系
   3. 基于不同metricx的侧面测试每一个benchmark所属的分类，决定需要在sniper中添加的feature和研究的方向 (按照TAMA的分类，Core执行主要分为retire/bad-speculation/frontend-stall/backend-stall)
   4. 最好可以获得interval区间段的PMC执行划分，以此为依据进一步进行分析
2. 修改sniper支持`readpmc`指令，使得ager_fog的测试框架可以在sniper上面运行
3. 目前先研究test测试集和reference测试集变化一致的若干benchmark

