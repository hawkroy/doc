# 安装CUDA-8.0
## 安装bumble-bee
```sudo apt-get install bumblebee bumblebee-nvidia primus bbswitch-dkms```
使用```lspci```命令显示NVIDIA-CARD的rev为ff，表明N卡不起动，只有使用```optirun XXX```命令才会使用N卡
可以使用```optirun glxgears```进行测试，如果正常运行，则说明已经成功
目前Debian-9官方的nvidia显卡驱动为rev-375

有的时候，运行optirun会遇到如下错误
```[ERROR]Cannot access secondary GPU - error: [XORG] (EE) /dev/dri/card0: failed to set DRM interface version 1.4: Permission denied```
当有此错误的时候，可以在.bashrc中添加命令```alias optirun='optirun --no-xorg'```，虽然无法启动图形显示，但是运行CUDA程序没有问题
## 源中安装必备软件
```sudo apt-get install libcuda1```
安装 必备的ring3的cuda驱动，否则后续的cuda framework无法运行
同时，cuda-8.0只支持gcc-5.3.1，所以debian-9中需要使用clang++-3.8进行编译，需要安装clang++-3.8相关软件
## NVIDIA CUDA官网下载CUDA的安装包
[nvidia-80-ga2(working)](https://developer.nvidia.com/cuda-80-ga2-download-archive) 
安转之前，设置环境变量```export PERL5LIB=.```
* cuda_8.0.61_375.26_linux.run	下载driver驱动对应的cuda安装包
* cuda_8.0.61_2_linux.run		cuda补充包patch-2
* cudnn-8.0-linux-x64-v7.tgz	DNN网络包
* TensorRT-2.1.2.x86_64.cuda-8.0-16-04.tar.bz2	tensorRT的安装包
按照上述顺序一次安装cuda的软件包，注意：在安装cuda的时候，不需要安装包里自带的NVIDIA驱动，会覆盖bumblebee的驱动设置
安装包里已经包含了Sample程序，主要选择好具体的安装位置就好
## 配置编译/运行环境
某些cuda的sample code需要x11的支持，所以下载如下packets
```sudo sudo apt-get install freeglut3-dev libx11-dev libxmu-dev libxi-dev libglu1-mesa libglu1-mesa-dev```
最后，编写脚本
```
    export HOST_COMPILER=clang++-3.8
    export GL_PATH=/usr/lib
```
每次运行cuda程序前，使用```. setenv.sh```设置cuda的编译和运行环境

# 安装WPS
## 官网下载WPS的最新安装程序
从官网下载的WPS安装程序可以安装，dpkg过程中会报错，也无法运行，因为缺少必要的libpng12-0的动态库。所以必须自己制作可以使用的WPS安装程序
debian-9已经不包含WPS需要的libpng的动态库版本，需要从老的debian版本中进行下载
libpng12-0_1.2.44-1+squeeze6_amd64.deb
## 制作自己的WPS安装包
使用```dpkg-deb```命令来解压和重新制作deb包
* 解压wps安装包
```dpkg-deb -x wps.deb  wps```		解压filesystem部分，这部分表示包的系统能够安装路径
```dpkg-deb -e wps.deb  wps/DEBIAN```	解压安装过程中的控制部分
* 解压libpng包
```dpkg-deb -x libpng.deb  libpng```
```dpkg-deb -e libpng.deb  libpng/DEBIAN```
* copy libpng库文件到wps filesystem部分
```
    cd wps/opt/kingsoft/wps-office/office6
    cp ../../../../../libpng/lib/libpng12.so.0.44.0 .
    ln -s libpng12.so.0.44.0 libpng12.so.0
```
* 修改wps的控制部分文件
TBD
* 重新打包wps的deb文件
```dpkg-deb -b wps wps-new.deb```
* 安装
## 添加WPS需要的字体
###方法1
下载WPS官网上的wps-office-font的deb包，进行安装。会安装到全局位置
###方法2（推荐使用)
下载wps-office-font的zip包，更全
```
   cd ~
   mkdir .local/share/fonts
   unzip wps-office-font.zip
   fc-cache -fv
```