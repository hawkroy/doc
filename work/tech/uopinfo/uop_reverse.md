[TOC]

## 背景

[uops.info](https://**uops.info**)网站提供了Intel / AMD各代处理器的指令执行延迟和吞吐率的数据，包括最新的Sunnycove, ZEN2等。通过这些测试数据，不仅可以搞清楚这两家厂商在各代际之间对于X86 ISA指令集的支持情况，也可以从一个侧面了解各家对于X86 ISA的实现的机制。同时，对于实现一个X86的性能模拟器而言，也同样需要每条指令的执行延迟和吞吐率数据，更为重要的是，目前的X86处理器会把X86指令翻译为称为Micro-OP (uop) 的类RISC指令执行，这对于一个X86的模拟器而言是致命的，没有准确的X86指令的解码信息，很难在微架构层面基于Micro-OP完成对于X86处理器的性能的准确建模。所以，本工具试图尝试通过[uops.info](https://**uops.info**)网站提供的测试信息，逆向出不同厂商对于X86指令的准确解码信息和一些微架构参数信息，并以此作为X86性能模拟器的配置输入完成对于某一款微架构的仿真；同时，在逆向的过程中，也会发现一些厂商对于X86指令实现上的不同之处。

本工具试图逆向以下信息：

- 每条X86指令的解码信息——翻译为几条uop，是否trap MSROM

<u>*注：对于trap MSROM的X86，本工具不能进一步逆向，只能将该X86指令作为一个整体对待和处理；进入MSROM的X86指令的翻译过程与该指令的执行上下文有关，不是纯粹的静态信息，所以无法通过一个测试文件获得准确结果*</u>

- 每条uop发送的执行端口(port)，执行延迟(delay)，以及吞吐率(throughput)
- 每条X86指令的uop间的依赖关系，uop间是否存在微融合(micro-fusion)
- 微架构中load / store的执行延迟(latency)、load-store forwarding的延迟(latency)
- 微架构中bypass network的延迟(latency)

## 方法

工具基于如下假设进行分析：

- 大部分的X86指令都是会 1 vs. 1的翻译为对应的uop；即大比率的指令都是翻译为1 uop的X86指令

- 大部分带有访存功能的X86指令，大概率可以按照如下的格式翻译为对应的uop序列

  <u>*以上两种情况，工具自行处理，不需要人工介入*</u>

  ```txt
  x86_with_memop:
  	load_uop			<----dep-----+
  	compute_uop	<----dep-----+		 -|
  	store_uop			   -|
  ```

- 其他不符合上述两种情况的指令，需要人工分析，并形成相应的配置文件用于工具进行后处理

  <u>*人工编码的意义：虽然初看起来，人工编码是一件费时、费力的工作(确实如此)；但是基于ISA的设计的稳定度，一次人工解码后，相应的解码文件大概率可以在同一厂家的各代际处理器中通用(可能面临一些小的修改)，只需要把不同代际间的ISA指令差异添补到新的微架构的配置文件中。这个工作是一个递增迭代的过程*</u>

### 工具执行流程

1. 从[uops.info](https://**uops.info**)网站下载指令执行结果的文件

2. 在工具目录执行如下命令

   ```bash
   # 需要python环境安装ruamel的yaml解析工具
   (python3/python2) python ./xml2config.py instruction.xml <uarch>
   # 脚本会自动去相同文件夹下的config目录查找对应的<uarch>.yml的人工配置文件，不存在，则不进行加载
   ```

3. 工具加载后，会按照如下步骤运行

   1. 读取脚本相同文件夹下的config目录中的\<uarch\>.yml人工配置文件，用于使用人工解码的X86指令解码，并存储为解码指令列表
   2. 遍历执行结果文件，生成当前微架构下的所有指令列表，并存储相关的测试结果信息
   3. 扫描指令列表，找到所有翻译为uop=1的指令，进行处理，并进行签名标记，用于后续自动处理复杂指令；此过程中会处理load指令
   4. 扫描指令列表，找到store指令，并进行处理，标记store-address/store-data间的micro-fusion
   5. 解码其他X86指令，按照如下算法进行：
      1. 如果指令有load操作，生成load uop
      2. 如果指令有唯一的计算操作，查找步骤2中的签名标记，如果找到，则翻译为对应的指令；否则，将当前指令加入待人工处理的队列
      3. 如果指令有store操作，生成store uop
   6. 对于已经解码的指令，执行解码逻辑检查，包括执行时间(latency)和执行端口(port)两方面的检查；如果任意一个没有通过，则在终端打印错误信息，用于后续的人工检查
   7. 产生当前阶段的输出文件，包括3个文件；以及若干当前微架构的统计信息
      - xxx.dump： 当前已经被工具解码的指令列表，包括工具解码和trap MSROM无法进一步解码的指令
      - xxx.todo：需要人工处理的指令列表，并形成工具需要的配置文件
      - xxx.portmap：工具目前已经解码，且翻译为1uop的X86指令在不同的执行port上的分类；这些指令是basic uop，可以在进行人工翻译的过程中使用，且这些指令在port上的分类一定程序反映了微架构对于不同类型uop在后端执行单元上的组织结构

## 工具配置文件格式

配置文件用于人工分析无法被工具自动处理的X86指令文件后，人工编码的X86解码信息。格式采用yaml语法格式

```yaml
# YAML format of X86 decoder (manual setting)
%YAML 1.1		# yaml version, follow 1.1 syntax
---
# section 1: uarch info
archinfo: {
	bypassnetwork: [delays for bypassnetwork],
	store_load_forwarding:	delay,		# load-store forwarding delay
	extra_delay: {extra delay info}		# currently, including m8/m16 load extra delay
}
---
# section 2: basic uop info, abstract uop manually, other basic uop will in xxx.portmap file
basicops:
	-
		uop name:
			desc: uop description
			port: execution port uop used
			latency: uop execution latency
	-
		others uop def
---
# section 3: X86 decode info
X86 string (uops.info format) [ADC (AL, I8)]:
	nocheck: 1/non-exist	# hint for tool, for no checking on decode result due to test error or info can't fix in config file
	bogus:  0/1		# not confirm uop sequence is correct, no more info to reverse
	msrom:  0/1		# X86 will trap MSROM or not
	alias:			# same decoded X86 instructions, list
		- X86 strings which will decode to same uop sequence
	uops:			# uop sequence, list
		-
			desc:  uop description
			# uopalias exclusive with port/latency
			uopalias:	name_in_basicops
			port: uop execution port
			latency: uop execution delay
			#
			wait: delay		# means after delay, uop will execution on same port as previous uop, see VDIVPD/PS instruction
			dep: uop dependency list, index from 0
			fuse: uop micro-fusing info, index from 0
		-
			other uop def
```

## 工具产生的解码文件格式

工具通过分析X86指令的执行结果文件会生成一个包含每条X86指令的译码和部分微架构参数的输出文件，同样采用yaml语法格式

```yaml
# section 1: uarch info
store_load_forwarding: 4		# current uarch store-load forwarding latency
# for load/store, here only descript port info, latency info not include(need? or specify in other file)
load: p23
sta_stream: p237
sta_complex: p23
sta: p237
std: p4
---
# section 2: X86 decode info
X86_XED_IFORM:
	width: xx	# operand size, some X86 instruction share same IFORM, but decode sequence not same
	# msrom/bogus/latency exclusive with uops
	# this is due to msrom/bogus instruction can't be decode correctly according to test file, so only can estimate an execution latency on X86 instruction
	msrom: 0/1	# trigger MSROM or not
	bogus: 0/1	# uop sequence confident or not
	latency: xxx	# instruction execution latency, if msrom/bogus == 1
	# exclusive with msrom/bogus/latency
	uops: 		# uop sequences, list
	# uop from 0 to ..., 
	- 	port: pxxx		# uop execution port
		latency: xxx	# uop execution latency
		delay:	xxxx	# means after execution latency, uop done (mainly for m8/m16 load uop)
		wait:	xxxx	# means after wait latency, uop will dispatch to previous port as before uop (mainly for VDIVPS/PD instruction)
		dep:	[]		# uop dependency list, start from 0
		fuse:	[]		# uop fuse list, start from 0
# other X86 instructions
....
```

## Haswell微架构逆向中的发现

使用[uops.info](https://**uops.info**)的结果对Intel的Haswell微架构的指令执行结果进行了逆向，在逆向过程中发现了如下一些有意思的现象，总结如下：

- Haswell架构不支持AVX512指令集，最高支持到AVX2的指令集，支持的指令集如下：

  AVXAES, MMX, SSE, CLFSH, BMI1, FMA, AVX2, SSSE3, SSE4, PAUSE, PCLMULQDQ, SSE2, RDRAND, PREFETCHWT1, AVX, MPX, VTX, LONGMODE, XSAVEOPT, F16C, AVX2GATHER, AES, BMI2, MONITOR, SSE3, RDTSCP, XSAVE, MOVBE, BASE, LZCNT, X87

- 测试结果中不包括X87指令，可以预想到X87在现有的软件中基本不会用到

- 除去X87指令，测试结果一共包含了3366条X86指令，其中有193条没有得出指令执行的结果(这些指令主要是一些系统控制类指令，无法或不容易进行测试)

- 在所有的3366条X86指令中，通过XLATE解码(即解码序列有HW固定)的指令为2867，占比为85.17%；仅有499条指令需要trap进入MSROM (依据Haswell 4发射的先验知识作为划分依据)

- 大部分指令符合工具**<u>方法中描述的假设</u>**，在2867条指令中有808条指令需要额外的人工分析并标注，占比为28.19%

- Intel Haswell微架构对于指令类型与执行port间的对应关系 (这个关系在Intel的各代际间有一定的普遍性)；**<u>同时，对于Intel的load uop操作来说，其不仅完成 load的执行动作，还会附带一些bit tranformation的功能，比如做些操作数的对齐，移位等操作，而这些对应的寄存器版本都是在p5上完成的</u>**

  ```txt
  p0:			simd_fp div/sqrt
  			simd logical op (immed version)
  			simd->gpr move
  p1:			simd_fp add/sub
  			shift
  			integer MUL
  			bit scan
  			POPCNT/ BMI /TZCNT
  			slow LEA
  p23:		load/store-address(complex)/store-address
  p4:			store-data
  # p5 port is very important port in Intel design, it is auxiliary port
  # many X86 instructions need p5 uop
  p5: 		shuffle
  			blend
  			simd mov with zx/sx
  			simd logical op (register version)
  			simd perm
  			broadcast
  			simd shift op
  			simd bit extract/insert op (insertps unpck, etc)
  			gpr->simd move
  p6:			direct branch
  p7:			store-address
  p01:		simd_fp mul
  			fma
  p06:		op related eflags
  			adc/sbb, btX, setcc, jcc
  			integer shift op
  p15:		simd_int add/sub
  			fast lea op
  			BMI op
  p015:		simd_int logical op
  			blend (immed version)
  p0156:		integer general op
  ```

- Haswell微架构在执行单元有一些特殊的执行限制，这主要包括如下几类指令的情况：

  VDIVPS/PD (YMM version)， VRCPPS (YMM version), VSQRTPS/PD (YMM version)

  这几类指令相对于对应的XMM version会多出一次操作的uop，这意味着在Haswell实际的执行单元中，只能同时完成4次PS操作或是2次PD操作，所以对于YMM version的情况需要多做一次，这造成了对于同一计算资源的竞争，在指令内部引入了资源依赖

  ```txt
  # 以VDIVPS YMM versio和 XMM version举例
  VDIVPS XMM (4ps calc)	 -----only 1uop----->	port0 DIV unit (support 4PS, 2PD)
  VDIVPD YMM (8ps calc)	 				------1st uop (first 4ps)-----^
  										-----2nd uop(secode 4ps)-------^
  ```

- 通过逆向，可以发现Haswell支持的bypassnetwork delay包括如下几个部分，有些在optimization手册中写明的bypassnetwork无法发现

  ```txt
  bypassnetwork:
  load	->	gpr			0T
  		->  xmm/mmx		1T
  		->  ymm			2T
  ```

- 对于EFLAG寄存器中常用的cc部分，Haswell将其分为两个部分： c, others(zopsa)；当某条指令需要读取完整的cc部分的时候，解码单元加入一条EFLAG_MERG的uop (用于生成合并后的EFLAG的cc部分)；典型的指令包括SETcc指令，比如SETBE与SETZ等指令译码的uop序列的差别

- 对于push/pop的指令，在manual手册中会有对于rsp寄存器的栈指针调整运算，但是在实际测试分析后，发现没有对应的uop存在，这说明Haswell对于栈指针的调整运算有特定优化，通过查看优化手册，这是通过硬件的stack engine机制进行处理的，消除了栈指针的调整运算部分

  ```txt
  PUSH
  	store [rsp]
  	sub rsp, 4			<--- not seen in result file, remove according to stack engine HW
  ```

- 对于大部分的8bit/16bit的X86 MOV指令，相对于32bit/64bit的X86 MOV指令，需要在load uop之外额外加入一条uop。这个原因主要在于P6架构后，寄存器的重命名采用PRF方式，在这种方式下对于寄存器的partial写的处理需要额外加入uop进行单独处理

- Haswell微架构采用特定的uop完成不同操作数类型之间的转换操作，且这些操作会进入特定的执行port进行执行，如下：

  ```txt
  int --> fp							p1
  32bit fp -> int						 p1
  64bit fp -> int						 p1
  16bit fp -> 32bit fp				 p1
  32bit fp -> 64bit fp				 p0
  gpr -> simd							p5
  simd -> gpr							p0
  ```

- 目前已知的Haswell支持的micro-fusion的操作：

  - load + alu操作
  - store分为store-address, store-data，这两个uop会进行micro-fusion

- 对于所有control register的读操作，需要经过p1端口执行

- 对于MXCSR的fp控制寄存器的操作，需要经过p0端口执行

- 对于形如[base+index*scale]的store操作，只能使用p23执行，而其他形式的store可以进入p7执行

## 如何进行人工分析和逆向

对于xxx.todo文件中的X86指令需要人工进行解码，这需要人员对于对应的X86指令的语义有一定的了解，这可以通过翻阅Intel的Vol.2的指令手册进行查看。进行人工翻译的步骤如下：

1. 查看待翻译指令的uop个数，和执行时间

2. 对照Vol.2的指令语义，查看xxx.portmap中是否包含于指令语义和执行时间对应的basic uop指令

3. 如果存在，则按照人员自身的理解标注各个uop之间的依赖关系，以及fuse信息；**配置文件中支持yaml格式的注释语法，可以用来添加人员自身对于某些指令解码分析的理解**

4. 如果不存在，可以自行概括、总结一些通用的basic uop；这往往需要将多个功能相似的X86指令放在一起进行分析，比如CVTxxx类的X86指令

5. 再次运行工具，进行分析；如果此时工具在进行解码逻辑检查的过程中报错，那么可能有如下几点需要考虑：

   - 选择的basic uop有问题，通常意味着port不对
   - uop间的依赖关系标注不正确
   - **<u>存在没有发现的bypassnetwork delay</u>**；对于这种情况，需要确认上述两种情况不存在的时候进行。如果是，那么设置相应的bypassnetwork delay到配置文件中，这种配置需要具有一定的通用性，不能只针对一个具体的X86指令有效

6. 对于load/store的特殊考虑

   load/store因为指令的特殊性，没有办法进行单独的测试以获得指令执行的latency(尤其是store，load可以)，所以必须组合起来一起考虑。所以在分析load、store的latency时候，还需要考虑相关指令的测试代码，这需要到[uops.info](https://**uops.info**)网站上查找对应指令的测试代码结构(这部分没有提供下载)。通常可以考虑的load、store指令可以如下面几条指令：

   MOV (R32, M32) / MOV (R16, M16) / MOV (M32, R32) / MOV (M16, R32) / ADD (R32, M32) / ADD (R16, M16) / ADD (M32, R32) / ADD (M16, R16)

   其中，MOV类的load指令用于考虑load的执行latency，ADD类的STORE指令用于考虑store的store-address 的latency和store-data的latency，以及store-load间的forwarding latency。之所以可以进行这样的分析，是因为网站上提供过的指令测试结果不是一个指令整体运行后的latency，而是考虑了指令间不同操作数间的依赖latency。关于这部分信息请参考文献 <u>uops.info:CharacterizingLatency,Throughput,and PortUsageofInstructionsonIntelMicroarchitectures</u> 

## 参考文献

1. uops.info:CharacterizingLatency,Throughput,and PortUsageofInstructionsonIntelMicroarchitectures ， Andreas Abel, ASPLOS'19, 2019