# 搭建CHX002硬件测试平台

- 硬件平台

  - CPU：单核CHX002， 8C[***目前BIOS的ROMSIP配置为4C，如需打开8C，需要重新烧录BIOS或是使用Shell Tool改写ROMSIP***]， 2.0GHz
  - DRAM：两条16G DDR4@2400MHz
  - 硬盘：1T SATA盘

- 操作系统

  为了兼容使用Pin-2.14，使用Xubuntu 14.04.3发行版

  - Kernel：3.19.8
  - gcc/g++/gfortran：4.8.4
  - libc: 2.19

1. 硬件设置

   ==TBD==

   1. BIOS中C-State和P-State的设置
   2. 最大工作频率设定

2. 安装Xubuntu 14.04.3发行版，创建测试用的test(安装时创建)，后续所有的工作目录都在test目录中进行

   在test的home目录中的.bashrc中添加`umask 002`用于设置test创建目录/文件的初始权限(group组设置为rwx)

3. 设置启动脚本,添加禁止address_random和ptrace的功能

   ==TBD==

4. CHX002使用的有线网络是realtek的RTL8111/RTL8168/8411型PCIe网卡，在官方ISO中使用的是rtl8169.ko的驱动，这个驱动无法正常的工作，所以需要下载realtek的官方驱动[https://www.realtek.com/zh-tw/component/zoo/category/network-interface-controllers-10-100-1000m-gigabit-ethernet-pci-express-software]，下载完成后，运行安装目录中的`sudo autorun.sh`

5. 安装所有必要的开发库`sudo apt-get install build-essential`

6. 测试的时候需要使用perf工具，所以需要自己单独编译kernel

   1. 首先安装build kernel需要的工具集

      ```bash
      sudo apt-get build-dep linux-image-`uname -r`
      sudo apt-get install kernel-package
      ```

   2. 下载Xubuntu对应的kernel source文件

      ```bash
      apt-get source linux-image-`uname -r`
      ```

   3. 下载针对兆芯CPU的perf driver的patch文件[X:\BJSW2_Internal\OSSolution\cpu_optimization\tasks\performance_monitor_counter\3-released_package\CHX002\src\linux-3.18]

   4. 进入kernel source的目录，执行如下命令

      ```bash
      patch -p1 < zhaoxin_patch_path
      cp /boot/config-`uname -r` .config
      make menuconfig      # 按照后面的图修改对应的配置信息
      fakeroot make-kpkg --initrd --append-to-version -perf kernel_image kernel_headers
      # --initrd: 创建initramfs
      # --append-to-version：在最终生成的.deb文件和最终安装到系统的linuz文件中加入tag，用于和主线内核区分
      # kernel_image/kernel_headers：分别用于编译kernel image和kernel include(用于编译module driver使用)
      ```

   5. 安装编译好的kernel_image和kernel_headers

      ```bash
      sudo dpkg -i linux-image-*.deb linux-headrs-*.deb
      ```

   6. 进入kernel source目录，再进入tool/perf目录，编译perf tool

      ```bash
      {sudo} make -j prefix={install_dir} install # 有一些依赖的库，可以按照提示使用apt-get安装
      ```

   7. 编译完成后，使用`perf list`查看新的kernel和PMC是否正常

7. 安装spec 2006测试套件，仿照[X:\BJSW2_Internal\OSSolution\eco-system\test_tool_optimization\speccpu\speccpu2006安装测试步骤.doc]中的步骤进行安装

   1. 文档中关于安装32位库的问题

      ```bash
      dpkg --add-architecture i386  # 添加i386的支持
      sudo apt-get update
      sudo apt-get install gcc-multilib g++-multilib libc6:i386 libstdc++6:i386
      ```

   2. 安装spec2006的问题，在执行./install进行安装的时候，需要在解压后的目录中创建install_archives，并将之前的压缩包拷贝进入目录，重命名为spec2006.tar.gz，否则安装过程中会出现错误，然后执行`./install.sh -e linux-suse101-AMD64[选择spec使用的若干管理程序版本] -d {install_dir}`

8. 安装intel xe2015 编译器，仿照[X:\BJSW2_Internal\OSSolution\eco-system\test_tool_optimization\speccpu\ICC安装文件]中的说明进行安装，使用的时候使用`source ${icc_dir}/bin/iccvars.sh`进行环境初始化

9. 安装python开发环境，用于后续sniper的开发或者进行测试时python脚本开发

   ```bash
   sudo add-apt-repository ppa:fkrull/deadsnakes-python2.7	#新的python 2.7的版本
   sudo add-apt-repository ppa:fkrull/deadsnakes  #新的python3的版本
   sudo apt-get update
   sudo apt-get installl python2.7 python3.5 python-virtualenv
   
   # create python virtualenv directory
   mkdir ${python_virtualenv_dir}
   virtualenv --python=python2.7 ${python_virtual_dir}  #安装python2的虚拟环境，可以同样安装同样的python3，目前没有安装
   source ${python_virtual_dir}/bin/activate   #进入python的虚拟环境
   pip install matplotlib numpy pandas scipy jupyter
   ```

10. 配置新的测试用户

   1. 修改test用户的所有文件目录的group权限为rwx，`chmod g+w ${all_file}`

   2. 创建新的真正测试用户帐户，以hawkwang为例

      ```bash
      sudo useradd hawkwang -g test -b /home -m -s /bin/bash
      sudo usermod -G sudo hawkwang
      sudo passwd hawkwang
      ```

   3. 在.bashrc中添加`umask 002`

   4. 添加ssh的远程访问，`sudo apt-get install openssh-server`

   5. 添加samba服务，`sudo apt-get install samba libtalloc2`，配置/etc/samba/smb.conf文件,具体配置参考bash脚本auto_install_env

11. 配置jupyter，可以远程进行python编程

    ```bash
    source ${python_virtual_dir}/bin/activate   #进入python的虚拟环境
    jupyter notebook --generate-config
    vim ${home}/.jupyter/jupyter_notebook_config.py  #编辑配置文件如下
    
    配置内容：
    c.NotebookApp.ip = u'10.29.8.53'      # 指定server监听的ip
    c.NotebookApp.port = 8889             # 监听的端口
    c.NotebookApp.token = ''              # 不使用加密登陆
    c.NotebookApp.open_browser = False    # 不自动打开浏览器
    c.NotebookApp.notebook_dir = u'/home/hawkwang/jupyter'    # 配置jupyter根目录
    ```

    配置结束后，使用命令`nohup jupyter notebook  > /dev/null 2>&1 &`使用后台方式启动jupyter，然后在客户端电脑可以在浏览器中输入http://ip:port进行jupyter登录

