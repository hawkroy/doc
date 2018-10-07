# User-Mode Linux (UML)

## Configure & Build

```bash
% make O=${build_dir} mrproper ARCH=um		# remove all configuration and *.o file
% make O=${build_dir} defconfig ARCH=um    # default UML configuration
% make O=${build_dir} menuconfig ARCH=um
% make O=${build_dir} ARCH=um							# make it
```



