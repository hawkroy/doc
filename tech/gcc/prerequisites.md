# GCC预置条件

编译gcc需要系统中安装有gmp、mpfr、mpc 3个应用库

1. 对于已经包含相关开发包的发行版来说，可以直接下载安装。以Debian/Ubuntu系统为例，执行下面的命令进行安装

   ```shell
   sudo apt install gmp-devel mpfr-devel libmpc-devel
   ```

2. 或者在安装gcc的过程中进行安装编译，gcc会自动下载相应的代码包

   ```shell
   ${gcc_src}/contrib/download_prerequisites
   # could set GRAPHITE_LOOP_OPT=no, to build GCC without ISL, only for graphite loop opt
   ```

如下列出了编译GCC需要安装的各种包

| 包名                              | 用途                                         |
| --------------------------------- | -------------------------------------------- |
| ISO c++11 compiler                | 用于编译bootstrap的编译器                    |
| C Standard library & headers      |                                              |
| GNAT                              | 对于有Ada需求，则需要GNAT编译器(4.7以上版本) |
| GNU bash / posix compatible shell | zsh not work                                 |
| awk                               | above 3.1.5                                  |
| GNU binutils                      |                                              |
| gzip(>= 1.2.4) / bzip(>=1.0.2)    |                                              |
| make (>= 3.80)                    |                                              |
| tar (>= 1.14)                     |                                              |
| perl ([5.6.1 .. 5.6.24])          |                                              |
| GMP / MPFR / MPC                  | GMP (>=4.3.2), MPFR(>=3.1.0), MPC(>=1.0.1)   |
| ISL                               | ISL(>= 0.15), for graphite_loop_opt          |
| zstd                              | for LTO bytecode compression                 |

如下列出了编译过程中用于改变GCC的包

| 包名                     | 用途                                                         |
| ------------------------ | ------------------------------------------------------------ |
| autoconf / m4 / automake | autoconf (>=2.69) / m4(>= 1.4.6) / automake (>= 1.15.1)      |
| gettext                  | gettext(>=0.14.5), for regenerate gcc.pot                    |
| gperf                    | gperf(>=2.7.2), for regenerate gperf input file. Ex, gcc/cp/cfns.gperf => gcc/cp/cfns.h |
| DejaGNU                  | 1.4.4, optional                                              |
| Expect                   |                                                              |
| TCL                      | for run testsuite                                            |
| autogen / guile          | autogen (>=5.5.4), guile (>= 1.4.1)<br />1. regenerate fixinc/fixincl.x from fixinc/inclhack.def and fixinc/*.tpl<br />2. 'make check' for fixinc<br />3. regenerate top level Makefile.in from Makefile.tpl and Makefile.def |
| Flex                     | flex (>=2.5.4)<br />1. modifing *.l lex file<br />2. build GCC in development in version-controlled source repo |
| texinfo                  | texinfo (>=4.7)<br />1. 'make info' for modifing *.texi files<br />2. 'make dvi' / 'make pdf' to produce documents, for pdf >=4.8 |
| Tex                      | 'make dvi/pdf' for texi2dvi / texi2pdf                       |
| Sphinx                   | sphinx(>=1.0), regenerate jit/docs/_build/texinfo from .rst to jit/docs |
| git                      |                                                              |
| SSH                      |                                                              |
| GNU diffutils            | diffutils (>=2.7), for submitting patch                      |
| patch                    | patch (>= 2.5.4), applying patchs                            |

