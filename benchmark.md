# Benchmark 

This document is meant to provide some statitics about the performance of the project, in
order for any user to evaluate the advantages and drawbacks of the different solutions
used.

Note that even if these statistics are specific to a use case, but will still provide a
general idea about the performances.

### Storage usage when porting the imx8mqevk from warrior to zeus

This change of version updates the kernel, the ramdisk image, the rootfs, and the device
tree. The resulting OSTree commit is about 61MB. This is the amount that the target will
have to download for this update.

In this scenario, we will update from warrior to zeus, and then add strace on the target
and observe the rootfs storage usage (`df -h /`) :

|     Action     |  Yocto  | `df -h /` |    Diff   |  Pulled  |
|:--------------:|:-------:|:---------:|:---------:|:--------:|
|        -       | warrior |  208.9 MB |     -     |     -    |
| warrior → zeus |   zeus  |  397.9 MB |   189 MB  |  61.0 MB |
|    + strace    |   zeus  |  256.8 MB | −141.1 MB | 388.7 kB |

Actions are the steps described in the previous paragraph. Yocto is the release version.
`df -h /` is the storage usage on the target. Diff is the difference between before and
after the update in terms of storage usage. Pulled is how much data the client OSTree has
pulled.

This table shows that OSTree is great in terms of storage for multiple reasons :
 - the update are deltas updates, meaning that only updated packages are actually shipped
   in the update
 - OSTree's [repositories features][ostree_repos] clearly show here as we take advantage
   of the archive repo to optimize the data we have to download
 - OSTree only keeps the last two commits in storage, meaning that the target storage will
   not continuously increase

Note that if the updates fails, the system can still [rollback][rollback_feat] even if the
full system is upgraded.

[ostree_repos]: https://ostree.readthedocs.io/en/latest/manual/repo/#repository-types-and-locations
[rollback_feat]: https://github.com/FullMetalUpdate/documentation/blob/master/rollback.md
