# GCC配置

## 配置时需要注意的问题

- GCC的编译目录必须位于源码目录之外，不能再源码目录之内进行configure, src_dir != obj_dir
- 对于使用automounted NFS进行编译的情况，在配置和编译前，设置PWDCMD，`pawd or amq -w`

## 配置参数说明

