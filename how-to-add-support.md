# Adding a new supported board to FullMetalUpdate

* [Setting up the sources](#setting-up-the-sources)
    * [Gathering the pre-existing resources that support your board](#gathering-the-pre-existing-resources-that-support-your-board)
        * [Cloning the repository in your sources](#cloning-the-repository-in-your-sources)
        * [Using repo, a tool for synchronizing repositories](#using-repo-a-tool-for-synchronizing-repositories)
    * [Cloning the FullMetalUpdate layers](#cloning-the-fullmetalupdate-layers)
* [Configuring your board to use the project](#configuring-your-board-to-use-the-project)
    * [Adding board specific configs](#adding-board-specific-configs)
    * [U-boot setup](#u-boot-setup)
        * [U-boot script setup](#u-boot-script-setup)
        * [U-boot environment options setup](#u-boot-environment-options-setup)
    * [`u-boot-fw-utils` setup](#u-boot-fw-utils-setup)
    * [Adding board specific bbclasses](#adding-board-specific-bbclasses)
    * [Configuring the wks file](#configuring-the-wks-file)
    * [Adding programs, preinstalled containers and other configurations](#adding-programs-preinstalled-containers-and-other-configurations)
        * [`fullmetalupdate-os-package`](#fullmetalupdate-os-package)
        * [`fullmetalupdate-containers-package`](#fullmetalupdate-containers-package)

FullMetalUpdate is a software solution to upgrade devices remotely. This solutions uses
softwares which require to be configured in order to properly work.

To add support, you will have to add files into [meta-fullmetalupdate-extra][meta_fmu_e],
the Yocto layer containing all machine-specific recipes and files required to make
FullMetalUpdate functional.

This document will through the different steps needed to configure you board to use
the project. To sum it up, this is what you will have to do :
 - **clone** the pre-existing BSP layers for your machine
 - setup U-boot and the U-boot environment
 - setup SOTA specific variables

## Setting up the sources
### Gathering the pre-existing resources that support your board

Yocto being a community-driven project, there are already a lot of recipes and
configuration files already written for a lot of boards. For example,
[meta-raspberrypi][meta_rpi] is the Yocto layer supporting [Raspberry Pi][rpi_website]
boards.

Check if your board is supported [here][open_embedded_layers]. If it is, you need to add
the layer to the sources. There are two ways for that that we describe below.

#### Cloning the repository in your sources

Add the repository supporting your board to your layers sources by cloning the repository.
Then, add the path to the new layer to you `build/conf/bblayers.conf`.

#### Using repo, a tool for synchronizing repositories

Your board may be part of layers we are already using for our project (iMX boards,
raspberrypi), please take a look at the branches of our [manifest][manifest_git] repo.

If you board isn't already part of what we use, please follow these steps :
 - Fork the [manifest][manifest_git] repo
 - Add a branch named `<yocto-release>/<machine-family>` :
   ```git
   git checkout -b <yocto-release>/<machine-family>
   ```
   `<yocto-release>` is the Yocto release you want to work with. Check the branches of the
   layer you want to add. Try to work on a [recent release][yocto_releases] if possible -
   this will ensure your board is actively supported.

   `<machine-family>` is the board family of your board.
 - Open the `dev.xml` file. You will need to add the lines that correspond to the layer
   you want to add to the project :
    1. The remote URL and its name :
       ```
       <remote fetch="remote-url" name="remote-name"/>
       ```

       `remote-url` is the URL of the remote. `remote-name` will be used at step 2 below.
    2. The repository to fetch :
       ```
       <project remote="remote-name" revision="yocto-release" name="layer-name" path="sources/layer-name"/>
       ```
       `remote-name` is the remote name you defined in step 1. `revision` is the yocto
       release you want to work with, and `layer-name` is the name of the layer, which is
       usually something like `meta-*`
 - If you are using [fullmetalupdate-yocto-demo][fmu_yocto_demo], you need to modify
  `yocto-entrypoint.sh` :
  1. Add your machine name to the corresponding `SUPPORTED_MACHINES_*`
  1. In `yocto_sync ()`, add a `case` statement that looks like this :
     ```bash
     "machine-name")
        BRANCH_REPO="machine-family"
     ;;
     ```
     where `machine-name` is your machine name and `machine-family` is the machine family
     that you specified in your branch name (`<yocto-release>/<machine-family>`).
  1. Change the URL that `repo` uses fetch the `dev.xml` file containing the repository
     list :
     ```
       echo "N" | repo init -u https://github.com/FullMetalUpdate/manifest -b "${YOCTO}/${BRANCH_REPO}" -m "${FULLMETALUPDATE}.xml"
     ```
     Change `FullMetalUpdate` to the name of your repository.
  1. Then, you can execute `./Startbuild.sh sync <machine> <yocto_version> dev` depending
     on what you have configured previously. The new repository should pop up in the
     sources directory.

### Cloning the FullMetalUpdate layers

If you haven't done it already, you will also need to clone the FullMetalUpdate layers and
add them to `bblayers.conf`:
 - [meta-fullmetalupdate][meta_fmu]
 - [meta-fullmetalupdate-extra][meta_fmu_e]

## Configuring your board to use the project

All the machine-specific configuration is done inside the
[meta-fullmetalupdate-extra][meta_fmu_e] repository. You normally don't need to modify
anything from [meta-fullmetalupdate][meta_fmu].

### Adding board specific configs

To add board-specific configurations, you need to add a directory with the name of your
board in `meta-fullmetalupdate-extra/conf/<machine-name>`. In this directory, add the
`local.conf.sample`, `layer.conf` and `bblayers.conf.sample` accordindly, which you can
base off other boards.

You may need to add a *dynamic layer* thanks to the variable `BBFILES_DYNAMIC` in the
`layer.conf`.

### U-boot setup
#### U-boot script setup

Our setup requires a specific U-boot configuration since we use ostree defined U-boot
variable to boot on the target and eventually rollback. More generally, this environment
is used by OSTree to configure on which rootfs OStree will chroot into.

FullMetalUpdate uses a custom recipe to configure the custom U-boot
environment. The configuration is located in a **dynamic layer** located inside
[meta-fullmetalupdate-extra][meta_fmu_e]. This layer is specific to your machine, and must
contain a `recipes-bsp` that contains these configurations:

```
meta-fullmetalupdate-extra/dynamic-layers/<yourbsp>-layer/recipes-bsp/bootfiles
├── <your-machine>
│   ├── boot.cmd
│   └── uEnv.txt
└── imx-bootfiles.bb
```

Take a look at other machines' configurations. There is a bootfiles recipes that is used
to deploy this custom script.

After having based off the work from another machine, verify that these variables are set
correctly in the `uEnv.txt`:

  - `bootiaddr` is the address when the initramfs image is loaded in RAM. You can lookup to
    the U-boot sources' config file of your machine to set this variable;
  - `bootmmc` represent the sd card id and partition id of the root file system. The card
    id should be set accordingly to your board's config.

`bootfiles` contains a recipes that deploys `boot.cmd`, a script used to
load `uEnv.txt` in RAM. You normally don't need to change this file.

#### U-boot environment options setup

U-boot's environment can be setup by patching the U-boot configuration files of your
board. Please take a look at the setup done in the `recipes-bsp` directory for NXP boards:

```
meta-fullmetalupdate-extra/dynamic-layers/freescale-layer/recipes-bsp/u-boot
├── imx8mqevk
│   ├── imx8mqevk_update.patch
│   └── update_common.h
├── u-boot-common.inc
├── u-boot-fslc_%.bbappend
├── u-boot-fslc_imx6qdlsabresd.inc
├── u-boot-imx_%.bbappend
└── u-boot-imx_imx8mqevk.inc
```

This should give you the proper path to add your own configurations.

### `u-boot-fw-utils` setup

`u-boot-fw-utils` is the software used to read and write U-boot's environment from the
userspace. It is required since the FullMetalUpdate uses this software to modify the
environment.

This environment can be configured through `libubootenv`:

```
meta-fullmetalupdate-extra/recipes-bsp/u-boot
├── <your-machine>
│   └── fw_env.config
└── libubootenv_%.bbappend
```

You need to modify `fw_env.config` and set the correct mmc id, the start address of the
U-boot environment, and the environment size. You can look this up in U-boot sources'
configs with the variables `CONFIG_ENV_OFFSET` and `CONFIG_ENV_SIZE`, or depending on
whether you have patched the configs' headers as described in the last section.

### Adding board specific bbclasses

You will need to add two classes which specify sota-related configurations in:
```
meta-fullmetalupdate-extra/classes
├── fullmetalupdate_<your-machine>.bbclass
└── fullmetalupdate_sota_<your-machine>.bbclass
```

Please take a look at other machine's configurations to base off your work.

### Configuring the wks file

The wks file is located in:

```
meta-fullmetalupdate-extra/scripts/lib/wic/canned-wks
└── fullmetalupdate-<your-machine>.wks.in
```

You can base off the wks file from the bsp layer that you are using. You need to add the
`/apps` partition to store the containers. Take a look at other wks file to add this
configuration line to your wks file.

### Adding programs, preinstalled containers and other configurations

You can add programs and pre-installed containers to your target by adding machine
specific bbappends in:

```
meta-fullmetalupdate-extra/recipes-core/images/
├── fullmetalupdate-containers-package_<your-machine>.inc
└── fullmetalupdate-os-package_<your-machine>.inc
```

#### `fullmetalupdate-os-package`

Add any program to the fullmetalupdate final package by using the `IMAGE_INSTALL_append`
variable.

You can increase the partition size of the OS by using the variable
`IMAGE_ROOTFS_EXTRA_SPACE` variable.

#### `fullmetalupdate-containers-package`

You can add pre-installed containers on your target by specifying containers in
`PREINSTALLED_CONTAINERS_LIST_append`.

You can also increase the partition size of the apps partitions by using the variable
`IMAGE_ROOTFS_EXTRA_SPACE` variable.

[uboot_setup_imx8]: https://github.com/FullMetalUpdate/meta-fullmetalupdate-extra/tree/warrior/dynamic-layers/freescale-layer/recipes-bsp
[meta_fmu_e]: https://github.com/FullMetalUpdate/meta-fullmetalupdate-extra
[meta_fmu]: https://github.com/FullMetalUpdate/meta-fullmetalupdate
[meta_rpi]: http://git.yoctoproject.org/cgit/cgit.cgi/meta-raspberrypi
[rpi_website]: https://www.raspberrypi.org/
[open_embedded_layers]: https://layers.openembedded.org/layerindex/branch/master/layers/
[manifest_git]: https://github.com/FullMetalUpdate/manifest
[yocto_releases]: https://wiki.yoctoproject.org/wiki/Releases
[fmu_yocto_demo]: https://github.com/FullMetalUpdate/fullmetalupdate-yocto-demo
