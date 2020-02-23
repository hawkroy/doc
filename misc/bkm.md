## Debian/Ubuntu为apt-get添加离线安装源

对于不能上网的机器，如果不想通过`dpkg -i <package>.deb`方式进行安装（有时无法解决依赖）,可以构建基于目录的离线安装源进行安装

1.  `apt-get clean`清除已有的安装包

2. `apt-get install <package>`安装 需要安装的软件包

3. 创建/offlinePackage目录用于离线包的保存

4. 在/offlinePackage中创建如下的目录格式

   - pool: 保存所有下载的deb安装包

     - i386: 保存i386架构下的包			xxxx_i386.deb
     - amd64: 保存amd64架构下的包   xxxx_amd64.deb

     对于通用架构的包在每个架构目录下各放一份，或者使用软连接的方式

   按照/etc/apt/sources.list中的格式来定义下面的目录格式

   `deb <url> <dist> main/....`

   - \<dist\>: 表明你的发行版本
     - main: 发行版的主要安装包，对于离线安装只需要创建这个分支就足够
       - binary-i386: 存储i386架构的包
       - binary-amd64: 存储amd64架构的包

5. 拷贝/var/cache/apt/archives目录到/offlinePackage/pool/xxx目录

6. 分别执行

   `dpkg-scanpackages pool/i386 /dev/null | gzip > <dist>/main/binary-i386/Packages.gz`

   `dpkg-scanpackages pool/amd64 /dev/null | gzip > <dist>/main/binary-amd64/Packages.gz`

7. 编辑离线机器上的 /etc/apt/sources.list文件; comment已有的行，添加如下行

   `deb file:///offlinePackage <dist> main`

8. `apt-get update`，然后安装 需要安装的软件包

------

## Ubuntu 64bit添加32bit执行环境

dpkg --print-architecture

sudo dpkg --add-architecture i386

sudo apt update

sudo apt install gcc-multilib g++-multilib

然后就可以使用32bit的执行程序

------

## Windows 10更新环境变量

windows 10更新环境变量方法：

- `My Computer » Properties » Advanced » Environment Variables`， 需要administrator密码
- `win+R`，打开命令窗口，输入`control sysdm.cpl,,3`，选择`Environment variables`，需要administrator密码
- 打开command line，输入`%windir%\System32\rundll32.exe(rundll32.exe) sysdm.cpl,EditEnvironmentVariables`，不需要administrator密码，只能修改用户用户变量
- 通过powershell的`setx`进行设置，没有具体研究