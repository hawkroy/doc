# 各种tool的编译步骤和命令

## SystemC

- version: 2.3.2
- \${src_dir}/configure \-\-prefix=\${install_dir} --enable-debug

## SST

### sst-core

- \${src_dir}/configure --prefix=\${install_dir} --disable-mpi

### sst-element

- \${src_dir}/configure --prefix=\${install_dir} --with-sst-core=\${sst-core-install_dir} --with-systemc=\${systemc-install_dir} --no-create --no-recursion

## Qemu

- \$(src_dir}/configure --target-list=x86_64-softmmu,i386-softmmu --enable-debug --disable-docs --enable-virtfs

## Verilator

- \${src_dir}/configure --prefix=\${install_dir}

