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

# change ${CONFIG_FILE} to alternative path, and favourite path
cabal user-config init --config-file=${CONFIG_FILE}
gvim /path/to/config
# change any path to your favourite path

# alias cabal
alias cabal='cabal --config-file=${CONFIG_FILE} --sandbox-config-file=${SANDBOX_CONFIG_FILE}'

# change config file, modify mirror to tsinghua
# default is ~/.cabal/config := ${CONFIG_FILE}
gvim ${CONFIG_FILE}
​```config
repository mirrors.tuna.tsinghua.edu.cn
  url: http://mirrors.tuna.tsinghua.edu.cn/hackage
# comment official mirror
-- repository hackage.haskell.org
--   url: http://hackage.haskell.org/
--   -- secure: False
--   -- root-keys:
--   -- key-threshold:

remote-repo-cache: /home/hawkwang/opt/haskell/cabal/packages
-- local-repo:
-- logs-dir: /home/hawkwang/.cabal/logs
world-file: /home/hawkwang/opt/haskell/cabal/world
-- store-dir: ??need change??
build-summary: /home/hawkwang/.cabal/logs/build.log
​```end of config

cabal new-install --install-dir=/path/to/cabal --install-method=copy|symlink cabal-install #更新cabal到最新

# cabal config file locate in ${CONFIG_FILE}
cabal update 	#更新已有的package

#
# install stack, stack for haskell project manage tool
# another tool compared to cabal
#
# get get_stack bash script
wget -qO- https://github.com/commercialhaskell/stack/tree/master/etc/scripts/get-stack.sh > /path/to/stack/bin/get_haskell_stack
chmod +x get_haskell_stack
# download stack program to /path/to/stack/bin
get_haskell_stack -d /path/to/stack/bin

# configure stack running dir
export STACK_ROOT=/path/to/stack
# run stack --no-install-ghc path to generate stack running dir contents
stack --no-install-ghc path
# modify ${STACK_ROOT}/config.yaml to change mirrors to tsinghua
gvim ${STACK_ROOT}/config.yaml
​```yaml
# for template using
templates:
  params:
    author-name: hawkwang
    author-email: wang09_224@163.com
    copyright: GPLv2
    github-username: hawkroy
    category: Development

# for mirror change
###ADD THIS IF YOU LIVE IN CHINA
setup-info-locations: ["http://mirrors.tuna.tsinghua.edu.cn/stackage/stack-setup.yaml"]
urls:
  latest-snapshot: http://mirrors.tuna.tsinghua.edu.cn/stackage/snapshots.json
snapshot-location-base: https://mirrors.tuna.tsinghua.edu.cn/stackage/stackage-snapshots/

package-indices:
  - download-prefix: http://mirrors.tuna.tsinghua.edu.cn/hackage/
    hackage-security:
        keyids:
        - 0a5c7ea47cd1b15f01f5f51a33adda7e655bc0f0b0615baa8e271f4c3351e21d
        - 1ea9ba32c526d1cc91ab5e5bd364ec5e9e8cb67179a471872f6e26f0ae773d42
        - 280b10153a522681163658cb49f632cde3f38d768b736ddbc901d99a1a772833
        - 2a96b1889dc221c17296fcc2bb34b908ca9734376f0f361660200935916ef201
        - 2c6c3627bd6c982990239487f1abd02e08a02e6cf16edb105a8012d444d870c3
        - 51f0161b906011b52c6613376b1ae937670da69322113a246a09f807c62f6921
        - 772e9f4c7db33d251d5c6e357199c819e569d130857dc225549b40845ff0890d
        - aa315286e6ad281ad61182235533c41e806e5a787e0b6d1e7eef3f09d137d2e9
        - fe331502606802feac15e514d9b9ea83fee8b6ffef71335479a2e68d84adc6b0
        key-threshold: 3 # number of keys required

        # ignore expiration date, see https://github.com/commercialhaskell/stack/pull/4614
        ignore-expiry: no
​``` end of yaml
wget -qO- https://github.com/commercialhaskell/stackage-content/tree/master/stack/global-hints.yaml > ${STACK_ROOT}/pantry/global-hints-cache.yaml

# configure template
cd ${STACK_ROOT}
git clone https://github.com/commercialhaskell/stack-templates.git templates

# modify ~/.bashrc to alias stack
#   - using system-ghc
#			ghc should in ${PATH}
#   - change local-bin-path to ${STACK_ROOT}/bin
#		- set resolver to system-ghc version
#			check https://www.stackage.org/ for lts-xxx mapping ghc-xxx version
alias stack='stack --system-ghc --resolver lts-14.27 --local-bin-path ${STACK_ROOT}/bin'

# stack usage
# [template-name] can be any git.website[gitlab, github] or local path
stack new new-prj [template-name]   # create a new prj
stack setup		# setup build environment, by downloading new ghc
stack build   # build prj
stack exec exec-name
stack clean		# prj clean
stack purge		# remove package
```

## Ocaml

ocaml使用opam包管理工具管理compiler和包

从https://github.com/ocaml/opam/releases下载对应平台的opam工具或者下载脚本https://raw.githubusercontent.com/ocaml/opam/master/shell/install.sh手动安装(目前这个网址无法访问)

使用opam进行ocaml的安装和使用

```sh
export OPAMROOT=/path/to/ocaml/root
# opam init will automatically install a default compiler, if no `--bare` option used
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

## Rust

```sh
# download initscript
curl https://sh.rustup.rs -sSf | cat > rustupinit
# set environment
CARGO_HOME=/path/cargo RUSTUP_HOME=/path/rustup ./rustupinit
```