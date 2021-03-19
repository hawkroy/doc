## Memory Dis-ambiguous

### 应用场景

load/store指令间因为访存地址存在依赖关系，比如如下的指令序列

```assembly
mov ebx, [0x1000]		; long latency, miss load
mov [ebx+0x800], 1		; suppose [ebx+0x800] = 0x5000
mov rax, [0x5000]		; load from 0x5000, RAW with inst#2
```

此种情况下，指令3必须从指令2获得正确数据；但是因为乱序执行的影响，指令3可能超前指令2执行。在目前的模拟器实现中，当load在MOB流水线执行时，会查看"老于"它的store指令是否都已经算出有效的VA地址。如果有任一个store没有获得有效VA(即没有执行过)，那么当前load会被block，直到所有的store都计算出有效的VA地址，此过程称为内存消歧(memory dis-ambiguous)过程。但是处理器中，很多时候load会超前之前的store执行，所以此种实现会引入不必要的等待。所以，模拟器中实现了针对memory dis-ambiguous的投机预测算法

### 算法实现逻辑

为了预测某个load是否可以在之前的store没有计算出有效VA的时候，投机执行，模拟器中引入了memory dis-ambiguous table(mdisamb)，并对处理器的流水线进行了修改。

**表结构**

目前模拟器中实现了具有64 entries的预测表，该表为多个phythread共享，没有tid信息。每个entry为一个饱和计数器，用于表示某个load是否可以投机执行的置信度

index = {msb_xor_bit, lip_directly_bits}, msb_xor_bit =\^ lip[9:5], 缩位为1位；lip_directly_bits = lip[5:0]

**uOP域**

| 添加的域         | 含义                                                         |
| ---------------- | ------------------------------------------------------------ |
| mdisamb_allow    | 表明当前load为mdisamb_load，即遇到unknown-address store，也不需要violate，而是投机继续MOB load流水线执行；当前只有counter > 15才允许设置为mdisamb_load |
| mdisamb_reset    | 表明当前load对应的mdisamb表中entry的counter清零              |
| mdisamb_update   | 表明当前load hit了一次unknown-address store，可以对mdisamb表中的counter+1；是否可以+1，需要看是否设置了mdisamb_reset标志 |
| mdisamb_done     | 表明当前mdisamb_load已经投机完成执行；此标志用于old store进行地址冲突检查使用 |
| mdisamb_bigflush | 表明当前mdisamb_load投机完成，但是与old store有冲突(old store比mdisamb_load后执行)，所以mdisamb_load读取了错误的结果，需要进行machine clear；此标志在retire阶段进行检查，并触发相应的machine clear |

**流水线**

![mdisamb](../spec/dia/mdisamb_flow.jpeg)

- schedule phase

  当load uOP调度执行时，通过lip查看mdisamb表中的counter。如果当前counter>15则设置mdisamb_allow标志表明为mdisamb_load，否则为normal_load

- MOB load exec phase

  load在MOB的load流水线执行时，需要进行unkown-address store检查，如果当前为mdisamb_load，则即使发现unknown-address violation，也可以继续执行；否则，设置mdisamb_update标志，block load执行

- MOB store exec phase

  store的流水线流程是对投机执行的mdisamb_load执行结果的一次检查。当store与load之间存在地址冲突的情况时，相应的load的mdisamb表对应的counter需要清零；如果存在mdisamb_load已经完成的情况(mdisamb_done设置)，则需要设置mdisamb_bigflush表示需要触发machine clear。

- retire phase

  根据load uOP上的mdisamb相应的扩展位进行相关处理。流程参考上面流程图