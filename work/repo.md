[TOC]

## Repos

### HW Simulation 环境

- CHX003 DRAMC  [noncpuroom]

  /logic/nuoxuanj/get_env_CHX003

- CHX003 NB [noncpuroom]

  /cpuwrk/chx003/sim/CNX/NB_PCIE_DRAMC_PTN_16nm/scripts/get_cnx_NB_PEXC_DRAMC

- CHX003 PCIE [noncpuroom]

  /cpuwrk/chx003/sim/CNX/PEXC/script_16nm/get_cnx_pexc_env

- CHX003 CPUID [noncpuroom]

  /cpuwrk/chx003/sim/CNX/CPUIF/env/get_cpuif，拿到环境后在根目录执行run_upenv

- CHX003 HIF

- CHX003 OPI

- CHX003 ZPI

## Documents

- CHX003 MRD

  http://sps.zhaoxin.com:3234/sites/chx003/Phase0/MRD/MRD/Forms/AllItems.aspx

- CHX003 IRS

  http://sps.zhaoxin.com:3234/sites/chx003/Phase1/Design/Register/IRSA0/Forms/AllItems.aspx

- CHX003 T-SPEC

  http://sps.zhaoxin.com:3234/sites/chx003/Phase1/Design/Spec/DigitalTSpec/T_SPEC/Forms/AllItems.aspx

- ESL-CHX003 Sharepoint

  [http://sps.zhaoxin.com:2234/sites/softwareteam/projectsite/Public%20Project/Forms/Allitems2.aspx?RootFolder=%2fsites%2fsoftwareteam%2fprojectsite%2fPublic%20Project%2fESL%2dCHX003&FolderCTID=0x012000065C898043D5504BA6ED11949024178E](http://sps.zhaoxin.com:2234/sites/softwareteam/projectsite/Public Project/Forms/Allitems2.aspx?RootFolder=%2fsites%2fsoftwareteam%2fprojectsite%2fPublic Project%2fESL-CHX003&FolderCTID=0x012000065C898043D5504BA6ED11949024178E)

- ESL-CHX003 git

  ESL-CHX003

## CPU Env

### 文件夹

在CPU室中，我们有2个文件夹：家文件夹和仿真文件夹

- 家文件夹：`/haydn/junshiw`
- 仿真文件夹：`/tmp-space/arch_T3/sw`

### 机器

我们可以使用如下三个不机器：

- `archnfs`。密码与CPU室密码相同。但是，不能进行设计代码的编译和生成

```shell
ssh -X archnfs
```

- `bjcpucg0744`。用于设计代码的编译和生成

```shell
rsh bjcpucg0744
```

- 软件虚拟机环境。密码是`123456`

```shell
ssh -X cpuroom@10.29.244.122
```

### CHX003环境

#### Git仓库

- RTL设计

  RTL设计部分的Git保存在路径`/ctwrk/CHX003/git/chx003_design.git`。Core设计的分支是`coreptn`；Uncore设计的分支是`chx003_N16`

- SMDL设计

  SMDL设计保存在路径`/ctwrk/CHX003/git/cnx_fe.git`。Core设计的分支是`master`；Uncore设计的分支是`chx003_N16`

- DV环境

  DV环境保存在路径`/ctwrk/CHX003/verfication/simulation/chx003_a0/git_repository/chx003_a0_simulation.git`。Core设计的分支是`coreptn`；Uncore设计的分支是`chx003_N16`

- 生成DV环境

  DV环境生成RTL设计代码、SMDL转Verilog代码以及uCode ROM编译的二进制文件

  步骤3、4和5可以并行运行

  1. 克隆DV环境的Git并且将分支切换到`coreptn`。*一定要记得切换分支*

     ```shell
     git clone /ctwrk/CHX003/verfication/simulation/chx003_a0/git_repository/chx003_a0_simulation.git
     git checkout coreptn
     ```

  2. 初始化DV环境。脚本`init_env`会克隆RTL设计和SMDL设计

     ```sh
     ./init_env
     ```

  3. 编译uCode ROM。需要在机器`bjcpucg0744`运行，并且利用`juanli-h`路径下的`.cshrc`建立运行环境。

     ```shell
     cd uCode/src/ucode/final_ucode
     make ucode
     make release
     ```

  4. 生成RTL设计代码。需要在机器`bjcpucg0744`运行，并且利用`juanli-h`路径下的`.cshrc`建立运行环境。

     `vcs`表示产生VCS运行环境；`one_core`表示生成1个core的仿真环境

     ```shell
     ./bld -vcs -one_core
     ```

     生成文件`chiron.v`和`chiron_lib.v`可以在路径`LINUX_X86/VCS/one_core/`中找到。参考大小105 MB

  5. 编译SMDL设计。需要在机器`bjcpucg0744`运行，并且利用`juanli-h`路径下的`.cshrc`建立运行环境

     ```shell
     cd uCode/smdl/fe_smdl
     ./build.py -mp
     ```

### 软件虚拟机环境中的环境建立

#### 拷贝文件

一些文件需要拷贝到软件虚拟机中，用来生成仿真代码。

- `LINUX_X86/VCX/one_core/chiron.v`和`LINUX_X86/VCX/one_core/chiron_libs.v`。生成的完整RTL设计代码。
- `soc/processor`。RTL设计源文件和所需的SMDL文件。
- `uCode/smdl/fe_smdl`。SMDL设计文件。
- `uCode/src/ucode/release`。uCode ROM二进制文件的文件夹。