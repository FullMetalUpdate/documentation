# Booting sequence in the case of FullMetalUpdate : 

 1. **Primary processor steps :**

     1. Primary bootstrap to initialize interrupt/exception vectors, clocks and RAM
     1. **U-boot** is decompressed and loaded into the RAM
     1. Execution is passed to U-boot

 1. **U-boot boot sequence :**

     1. Flash, serial console, Ethernet MAC Address (etc.) are configured
     1. A minimal bootscript called `boot.scr` is executed and is used only to load a
        **custom** script called `uEnv.txt`, written by us. These two scripts (the minimal
        one and the custom one) are both defined in
        [meta-fullmetalupdate-extra][meta_fmu_e] in the dynamic-layers folder. Those are
        both handled by the `imx-bootfiles.bb` recipe.
     1. The custom `uEnv.txt` u-boot script does several things : 
        - It defines the proper storage interfaces and addresses (`bootmmc`, `bootiaddr`…)
        - It defines multiple boot commands (`bootcmd*`) which all serve different
          purposes :
           - `bootcmd` is the main boot command : it contains conditional branches used to
             boot either onto the current deployment or the rollback deployment. It also
             execute the ostree boot command `bootcmd_otenv`
           - `bootcmd_otenv` is used to load OStree environment variables, needed by
             OStree to boot on the proper deployment. Note that this command imports
             theses variables from **another `uEnv.txt`** which **totally differs** from
             our custom one, defined in Yocto. This `uEnv.txt` is defined in
             `/boot/loader/uEnv.txt` and written by OSTree when staging commits. Please
             see the [OSTree documentation][ostreedoc_boot] for more info.
           - When this script runs, it will execute these different commands and the
             kernel, ramdisk image, device tree blob will be loaded. The execution is then
             passed to the kernel.

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

 1. **OSTree initrd script :**
    
    OSTree has a custom initrd script since the file system is a bit different on OSTree
    devices (see [the ostree documentation][ostreedoc_syslayout]). The script mounts the
    different file systems and determines the current deployment by parsing the kernel
    command line.

    More details this script are found in the [meta-updater][metaupdater_script] layer.

 1. **systemd :**
    
    systemd, the init manager, does his usual job : starting the usual target units, but
    also start the FullMetalUpdate client with a custom service file (which is found
    [here][fmu_service]).

 1. **Client initialization :**

    The client initializes the container ostree repo and starts the different preinstalled
    containers.

## Useful resources : 
 - `man bootup`, `man boot`, `man bootparams`
 - https://ostree.readthedocs.io/
 - https://www.denx.de/wiki/U-Boot/Documentation

[meta_fmu_e]: https://github.com/FullMetalUpdate/meta-fullmetalupdate-extra
[ostreedoc_boot]: https://ostree.readthedocs.io/en/latest/manual/deployment#the-system-boot
[ostreedoc_syslayout]: https://ostree.readthedocs.io/en/latest/manual/adapting-existing/#system-layout
[metaupdater_script]: https://github.com/advancedtelematic/meta-updater/blob/master/recipes-sota/ostree-initrd/files/init.sh
[fmu_service]: https://github.com/FullMetalUpdate/meta-fullmetalupdate/blob/warrior/recipes-fullmetalupdate/fullmetalupdate/files/fullmetalupdate.service
