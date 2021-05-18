[TOC]

# Vim使用技巧

## 基本

- message

  使用`echo / echom`进行vim打印，使用`messages`查看echom的输出

- option setting

  使用`set[local] option_xxx=xxxx`或是`set[local] [no]option_xxx[!]`，! 表示取反二项选项，使用`set[local] option_XXX?`查看当前的vim option设置；\[local\]表明当前option只适用于当前buffer

- abbreviation

  `[i/n/v]abbre [\<buffer\>] {short-string} {replace-string}`

  abbreviation与key-mapping的区别在于mapping只根据string-sequence进行匹配，而abbreviation会考虑string-squence的上下文，只有两边都是\<space\>才会替换

- key-mapping

  只使用`[mode]noremap [<buffer>] {short-keys} {command}`的非递归方式，避免vim递归解析mapping；\<buffer\>指名当前mapping只适用于当前打开的buffer

  使用`set mapleader`和`set maplocalleader`设置mapping中的`<leader>`，maplocalleader只适用于当前buffers

- autocommand

  `:autocmd {event-name} {pattern} {command}`

  augroup {NAME}

  ​	"remove previous augroup, otherwise, all augroup will gather

  ​	autocmd!

  ​	autocmd .....

  augroup END
  
- normal [!]

  在任何情况下，将vim至于`normal mode`并执行后续的命令，加`!`表明normal后面的字符命令不会进行key-mapping

- function [!]

  定义脚本函数，当存在`!`时，如果存在同名的函数则同名函数会被替换，否则则触发错误报告

## Vim脚本编写

### Windows中Python编写Vim脚本

配置pythonXX.dll和python.exe所在的目录到用户或是系统的path中，设置python的安装目录(包含lib目录)的文件夹到PYTHONHOME的全局变量，使用```:python import os```进行测试

### Vim脚本管理

#### 脚本目录结构(远古时期)

- plugin

  vim启动时，加载的插件

- ftplugins

  根据当前`filetype`加载的插件，只能使用buffer local option

- indent

  根据当前`filetype`设置当前文件的缩进，只能使用buffer local option

- autoload

  autoload中的vim脚本会在脚本实际运行过程中用到时才加载，而不是启动时自动加载

- after

  在`.vim/plugin`加载完毕后，vim启动后，进行加载，用于改写vim的默认配置选项

- syntax

  根据文件后缀，即当前`filetype`设置当前文件的语法着色，只能使用buffer local option

- colors

  当加载新的配色方案时，由vim查找`.vim/color/*.vim`中的文件

- compiler

  根据当前`filetype`设置当前文件的编译选项，只能使用buffer local option

- doc

  用于vim的`:help`文档帮助

#### 脚本管理插件(现代时期)

##### pathogen

pathogen会将`.vim/vundle/*`目录下的所有插件全部加入到vim的`RUNTIMEPATH(rtp)`中，这样所有的插件都会被vim找到，并加载。每个vundle中的插件目录按照远古时期的目录组织方式进行组织，并且可以使用version control工具进行管理和更新

##### vundle

在pathogen的基础上，添加了从远程github等托管平台或是vim-scripts.org上自动下载插件并配置的能力，并将所有管理的插件放入到bundle目录中。exvim使用vundle进行插件管理。其基本用法如下：



### 脚本变量作用域

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

