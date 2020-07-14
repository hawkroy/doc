# 各种tool的安装步骤

## SystemC

```sh
# version 2.3.2
${src_dir}/configure --prefix=${install_dir} --enable-debug
```

## SST

```sh
# build core
${src_dir}/configure --prefix=${install_dir} --disable-mpi

# build elements
${src_dir}/configure --prefix=${install_dir} --with-sst-core=${sst-core-install_dir} --with-systemc=${systemc-install_dir} --no-create --no-recursion
```

## Qemu

```sh
# build qemu
$(src_dir}/configure --target-list=x86_64-softmmu,i386-softmmu --enable-debug --disable-docs --enable-virtfs
```

## Verilator

```sh
${src_dir}/configure --prefix=${install_dir}
```

## Haskell

```sh
#
# get ghcup
#
mkdir ~/opt/haskell/bin
curl https://gitlab.haskell.org/haskell/ghcup/raw/master/ghcup >  ghcup
chmod +x ghcup
# change ghcup source
# INSTALL_BASE="$GHCUP_NSTALL_BASE_PREFIX/.ghcup" to INSTALL_BASE="$GHCUP_INSTALL_BASE_PREFIX"
vi ./ghcup
export GHCUP_INSTALL_BASE_PREFIX=/path/to/haskell

#
# install ghc
#
ghcup install	# 安装最新版本
ghcup install x.x.x	# 安装特定版本
ghcup set x.x.x		#设置当前使用版本

#
# install cabal, cabal for haskell package manager
#
ghcup install-cabal			#安装cabal到INSTALL_BASE/cabal
cabal new-install --install-dir=/path/to/cabal --install-method=copy|symlink cabal-install #更新cabal到最新

# cabal config file locate in ~/.cabal/config
cabal update 	#更新已有的package

#
# install stack, stack for haskell project manage tool
#

```

## Ocaml

ocaml使用opam包管理工具管理compiler和包

从https://github.com/ocaml/opam/releases下载对应平台的opam工具或者下载脚本https://raw.githubusercontent.com/ocaml/opam/master/shell/install.sh手动安装(目前这个网址无法访问)

使用opam进行ocaml的安装和使用

```sh
export OPAMROOT=/path/to/ocaml/root
opam init --disable-sandboxing		# if no --disable-sandboxing, should install bubblewrap package
eval `opam env`
opam switch create 4.10.0
eval `opam env`
which ocaml
ocaml -version
```

## RISC-V

### rocket-chip

```sh
git clone https://github.com/ucb-bar/rocket-chip.git
cd rocket-chip
git submodule update --init
```

### rocket-tools

```sh
git clone https://github.com/freechipsproject/rocket-tools
cd rocket-tools
git submodule update --init --recursive

# select install dir
export RISCV=/path/to/install/riscv/toolchain
export MAKEFLAGS="$MAKEFLAGS -jN" # Assuming you have N cores on your host system

# build
./build.sh
./build-rv32ima.sh  #if you are using RV32.
```

## openrisc toolchain

### bare-metal compiler

```sh
sudo apt-get install git libgmp-dev libmpfr-dev libmpc-dev \
    zlib1g-dev texinfo build-essential flex bison
export PREFIX=/opt/toolchains/or1k-elf

git clone https://github.com/openrisc/binutils-gdb.git
git clone https://github.com/openrisc/or1k-gcc.git gcc
git clone https://github.com/openrisc/newlib.git
ln -s binutils-gdb binutils
ln -s binutils-gdb gdb
# build binutils
mkdir build-binutils; cd build-binutils
../binutils/configure --target=or1k-elf --prefix=$PREFIX --disable-itcl --disable-tk --disable-tcl --disable-winsup --disable-gdbtk --disable-libgui --disable-rda --disable-sid --disable-sim --disable-gdb --with-sysroot --disable-newlib --disable-libgloss --with-system-zlib
make
make install
cd ..

# build gcc stage1
mkdir build-gcc-stage1; cd build-gcc-stage1
../gcc/configure --target=or1k-elf --prefix=$PREFIX --enable-languages=c --disable-shared --disable-libssp
make
make install
cd ..

# build newlib
mkdir build-newlib; cd build-newlib
../newlib/configure --target=or1k-elf --prefix=$PREFIX
make
make install
cd ..
# alternative for multicore
../newlib/configure --target=or1k-elf --prefix=$PREFIX CFLAGS_FOR_TARGET="-D__OR1K_MULTICORE__"

# build gcc stage2
mkdir build-gcc-stage2; cd build-gcc-stage2
../gcc/configure --target=or1k-elf --prefix=$PREFIX --enable-languages=c,c++ --disable-shared --disable-libssp --with-newlib
make
make install
cd ..

# build gdb
mkdir build-gdb; cd build-gdb
../gdb/configure --target=or1k-elf --prefix=$PREFIX --disable-itcl --disable-tk --disable-tcl --disable-winsup --disable-gdbtk --disable-libgui --disable-rda --disable-sid --with-sysroot --disable-newlib --disable-libgloss --disable-gas --disable-ld --disable-binutils --disable-gprof --with-system-zlib
make
make install
cd ..
```

### linux-based compiler

```sh
./build.sh
# if need custom
vi config.sh
```

## GHDL

```sh
# ghdl develop environment
sudo apt install gnat-6		# ada environment
```



