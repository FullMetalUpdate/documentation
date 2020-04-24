# Booting sequence in the case of FullMetalUpdate : 

 1. **Primary processor steps :**

     1. Primary bootstrap to initialize interrupt/exception vectors, clocks and RAM
     1. **U-boot** is decompressed and loaded into the RAM
     1. Execution is passed to U-boot

 1. **U-boot boot sequence :**

     1. Flash, serial console, Ethernet MAC Address (etc.) are configured
     1. Environment variables are loaded from non-volatile memory (allows for configuration)
     1. A delay is added for any user to enter the U-boot shell (usually a few
        seconds, can be configured by the `bootdelay` env. variable)
     1. A custom u-boot script does several things : 
         1. It declares custom kernel boot arguments (`bootargs`) : the ostree root, a
            ramdisk root, a ramdisk size along with other arguements such as the rootfs
            type (e.g. ext4) and mode (ro/rw)
         1. It loads the OSTree script
         1. It declares a custom boot command that sets the previously declared kernel
            arguments, loads the kernel and the ramdisk from memory
        > *Note :* 
        >
        > This custom environment is written in the
        > `meta-fullmetalupdate-extra/…/recipes-bsp/bootfiles/<machine>/uEnv.txt`. A important thing to note
        > is that this script is indeed loaded at boot time, but you’ll also find a `uEnv.txt`
        > file in `/boot/loader/uEnv.txt` when the device has booted. This environment is also
        > **loaded at boot time** thanks to the command `mmc ${bootmmc} $loadaddr
        > /boot/loader/uEnv.txt; env import -t loadaddr $filesize` ran inside the yocto script.
        > 
        > This allows to run a custom script from written in our layerside and also load
        > the OSTree script which will determine on which deployment it will boot.
     1. The initramfs image is loaded (named `initramfs-ostree-image-<machine>`)
     1. The kernel is loaded and the execution is passed to the kernel

 1. **Kernel boot sequence :**

    The kernel initializes the various components/drivers of the target.

    > *Note :*
    >  - The typical kernel command line looks like this (on the IMX8) :
    >
    >    `rw rootwait console=ttymxc0,115200 ostree=/ostree/boot.0/poky/<hash>/0
    >    ostree_root=/dev/mmcblk1p2 root=/dev/ram0 ramdisk_size=8192 rw rootfstype=ext4
    >    consoleblank=0 vt.global_cursor_default=0 video=mxcfb0:dev=ldb,bpp=32,if=RGB32`
    >    where `<hash>` is the current deployment's hash.
    >
    >    It contains interesting arguments such as the `ostree=` which is parsed by OSTree
    >    to determine the current deployment.
    >
    >  - the kernel arguments are written both from our `uEnv.txt` and also from the OSTree
    >    `uEnv.txt`.

 1. **OSTree initrd script :**
    
    OSTree has a custom initrd script since the file system is a bit different on OSTree
    devices (see [the ostree documentation][ostreedoc_syslayout]). The script mounts the
    different file systems and determines the current deployment by parsing the kernel
    command line.

    More details this script are found in the [meta-updater][metaupdater_script] layer.

 1. **systemd :**
    
    systemd, the init manager, does his usual job : starting the usual target units, but also start the
    FullMetalUpdate client with a custom service file (which is found
    [here][fmu_service]).

## Useful resources : 
 - `man bootup`, `man boot`, `man bootparams`
 - https://ostree.readthedocs.io/
 - https://www.denx.de/wiki/U-Boot/Documentation

[ostreedoc_syslayout]: https://ostree.readthedocs.io/en/latest/manual/adapting-existing/#system-layout
[metaupdater_script]: https://github.com/advancedtelematic/meta-updater/blob/master/recipes-sota/ostree-initrd/files/init.sh
[fmu_service]: https://github.com/FullMetalUpdate/meta-fullmetalupdate/blob/warrior/recipes-fullmetalupdate/fullmetalupdate/files/fullmetalupdate.service
