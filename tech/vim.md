# Vim使用技巧

## 基本

只使用`[mode]noremap`的非递归方式，避免vim递归解析mapping

使用`set mapleader`和`set maplocalleader`设置mapping中的`<leader>`

## Vim在Windows中使用Python编写Vim脚本

配置pythonXX.dll和python.exe所在的目录到用户或是系统的path中，设置python的安装目录(包含lib目录)的文件夹到PYTHONHOME的全局变量，使用```:python import os```进行测试

## 脚本目录结构
- Plugin
- Ftplugins
- Indent
- Autoload
- Syntax
- Colors

## 脚本变量作用域
- v: vim-predefined全局作用域
- g: 全局作用域，在整个vim运行过程中都可见，不加变量域修饰时的默认作用域
- b: 只作用于buffer定义中
- t: 只作用于tab定义中
- w: 只作用于window定义中，即当前到的"焦点"
- l: 函数局部变量作用域
- s: 在某个文件中定义的，与<SID>相关
- a: 函数的参数作用域

## mode-line

在不同的文件格式下，使用注释的方式在行末或行首添加

```vim
vim: set tagstop=2, nowrap, ...
```



## Cscope支持

使用的命令
- cscope add  添加一个cscope数据库
- cscope find进行查找, Vim支持8种cscope查询： 
  - s: 查找C语言符号，即查找函数名、宏、枚举值等出现的地方 
  - g: 查找函数、宏、枚举等定义的位置，类似ctags所提供的功能 
  - d: 查找本函数调用的函数 
  - c: 查找调用本函数的函数 
  - t: 查找指定的字符串 
  - e: 查找egrep模式，相当于egrep功能，但查找速度快多了 
  - f: 查找并打开文件，类似vim的find功能 
  - i: 查找包含本文件的文件

插入自[http://easwy.com/blog/archives/advanced-vim-skills-cscope/](http://easwy.com/blog/archives/advanced-vim-skills-cscope/)

## Windows, Tabs, Buffers
### Buffer
Buffer是指在编辑文件的缓冲区，包括对于这个文件的设置和标记信息
状态

| 状态      | 描述                                      |
| :-------- | :--------                                |
| Active    | 在编辑的文件,显示在屏幕上                   |
| Hidden    | 隐藏不显示,可以包含修改,设置和标记信息       |
| Inactive	| 不包含任何文件,不显示                      |

可以使用命令 ```ls```, ```buffers``` 查看
对于Hidden的窗口，可以采用 ```hid[e] +cmd file``` 的方式执行初始的vim command
'buftype' option可以设置buffer的不同属性

| Status    | Description                                                 |
| :-------- | :--------                                                   |
| empty     | Normal buffer                                               |
| Acwrite   | A buffer will always written with an autocmd ```:autocmd``` |
| Help      | Help window                                                 |
| Nofile    | Not associated with file and not written                    |
| Nowrite   | Not-be-written                                              |
| quickfix  | Quickfix list                                               |
