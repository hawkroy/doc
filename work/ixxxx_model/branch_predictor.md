[TOC]

## Branch Predictor的结构

模拟器中的Branch Predictor结构采用了BG(bimodal+global history)或是BGG(bimodal + big global history)的结构，其branch predictor的具体结构和相互关系如下：

![predictor_stru](dia/predictor_stru.jpeg)

模拟器中各部分预测结构的基本信息（以cpu-demo.cfg为例）

targetglobalS entry {target, tag, tid}

| module            | parameters                                                   | entry structure                                              |
| ----------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| RAS               | 16 entries                                                   |                                                              |
| iBTB              | 256 entries, direct map<br />hash_func:<br />     index = stew[15:0] ^ IP[15:0]<br />     index = stew[5:0, 15:6] ^ IP[20:5] | {target_VA}                                                  |
| BTB               | 2048 entries, 4-way                                          | target_VA<br />uop_opcode<br />counter<br />last_stew<br />last_bigstew<br />mru_bit<br />miss<br />disagree_static_pred<br />tag (offset + btb_tag_size(9)) |
| global predictor  | 2048 entries, 4-way<br />hash_func:<br />     hash_index = stew[15:0] ^ IP[15:0]<br />     hash_index = stew[15:0] ^ IP[19:4]<br />index = {IP[4:0], hash_index} | satuar counter<br />counter_bl_0   not tk satuar counter<br />counter_bl_1   tk satuar counter |
| bimodal predictor | 4096 entries, direct-map<br />IP[tid[0], bimodal_len[-1, -2]:0] | satuar counter                                               |
| loop predictor    | 128entries, 2-way<br />tag[5:0], MSB=tid                     | learn_mode<br />predict_mode<br />relearn_mode<br />validate_mode<br />spec<br />prediction<br />max_counter<br />real_counter<br />spec_counter |
| stew length       | 15-bit                                                       |                                                              |

### 投机执行的处理



### BPU的预测



### BPU的更新



## Mis-Predict的处理