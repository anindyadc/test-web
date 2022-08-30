---
title: "Extend Root Partition"
date: 2022-08-18T18:00:04+05:30
draft: true
---

### Extend partition

```
fdisk -l
```

```
Disk /dev/sda: 32.2 GB, 32212254720 bytes, 62914560 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x000420d1

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048     2099199     1048576   83  Linux
/dev/sda2         2099200    41943039    19921920   8e  Linux LVM

Disk /dev/mapper/centos-root: 18.2 GB, 18249416704 bytes, 35643392 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/mapper/centos-swap: 2147 MB, 2147483648 bytes, 4194304 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```


```
fdisk /dev/sda
```

Enter p to print your initial partition table.

Enter d (delete) followed by 2 to delete the existing partition definition.

Enter n (new) followed by p (primary) followed by 2 to re-create partition number 2 and enter to accept the start block and enter again to accept the end block which is defaulted to the end of the disk.

Enter t (type) then 2 then 8e to change the new partition type to "Linux LVM".

Enter p to print your new partition table and make sure the start block matches what was in the initial partition table printed above.

Enter w to write the partition table to disk. You will see an error about Device or resource busy which you can ignore.

### Update kernel in-memory partition table

After changing your partition table, run the following command to update the kernel in-memory partition table:

```
partx -u /dev/sda
```

### Resize physical volume

Resize the PV to recognize the extra space

```
pvresize /dev/vda2
```

### Resize LV and filesystem

In this command centos is the PV, root is the LV and /dev/vda2 is the partition that was extended. Use pvs and lvs commands to see your physical and logical volume names if you don't know them. The -r option in this command resizes the filesystem appropriately so you don't have to call resize2fs or xfs_growfs separately.

```
lvextend -r centos/root /dev/sda2
```