---
title: PVE 安装 OpenWRT 虚拟机
comments: true
date: 2026-01-09 21:01:07
tags:
---

<!-- more -->

## VM Config

![](hardware.png)

## Openwrt Combined-EFI Installation

### Step 1

在 PVE Hardware Options 中选中对应的虚拟磁盘，点击上方菜单中的 Detach

### Step 2 写入对应磁盘镜像

```sh
dd if=./openwrt-23.05.6-x86-64-generic-ext4-combined-efi.img of=/dev/pve/vm-104-disk-0
```

### Step 3 Resize 分区

```sh
$ fdisk /dev/pve/vm-104-disk-0 
```

在 fdisk 中按照以下步骤操作：

**1. `p` 打印分区表**

```sh
Command (m for help): p
Disk /dev/pve/vm-104-disk-0: 2 GiB, 2147483648 bytes, 4194304 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 65536 bytes / 65536 bytes
Disklabel type: gpt
Disk identifier: 552FB917-9C3E-2018-DD4D-13FCFD198400

Device                     Start    End Sectors  Size Type
/dev/pve/vm-104-disk-0p1     512  33279   32768   16M Linux filesystem
/dev/pve/vm-104-disk-0p2   33280 246271  212992  104M Linux filesystem
/dev/pve/vm-104-disk-0p128    34    511     478  239K BIOS boot
```

**2. `d` 删除分区**

删除分区时会要求输入分区编号，这里我们输入 `2`

```sh
Command (m for help): d
Partition number (1,2,128, default 128): 2

Partition 2 has been deleted.
```

**3. `n` 新建分区**

- 分区编号：`2`，必须保持一致
- 起始扇区：`33280`，也必须和原来保持一致
- 结束扇区：直接回车，或输入 `+` 指定大小
- 移除 ext4 签名：`N`，这里一定要保留原有的 ext4 签名否则就直接格式化了

```sh
Command (m for help): n
Partition number (2-127, default 2): 2
First sector (33280-4194270, default 34816): 33280
Last sector, +/-sectors or +/-size{K,M,G,T,P} (33280-4194270, default 4192255): 

Created a new partition 2 of type 'Linux filesystem' and of size 2 GiB.
Partition #2 contains a ext4 signature.

Do you want to remove the signature? [Y]es/[N]o: N
```

**4. `w` 写入分区表**

```sh
Command (m for help): w

The partition table has been altered.
Calling ioctl() to re-read partition table.

The kernel still uses the old table. The new table will be used at the next reboot or after you run partprobe(8) or partx(8).
```

此时我们磁盘的分区大小已经顺利调整完成了，但是磁盘上 ext4 文件系统的大小还保持原来的大小

### 扩展 ext4 文件系统

**1. `kpartx`挂载虚拟设备**

```sh
$ kpartx -av /dev/pve/vm-104-disk-0 
add map pve-vm--104--disk--0p1 (252:11): 0 32768 linear 252:10 512
add map pve-vm--104--disk--0p2 (252:12): 0 4158976 linear 252:10 33280
add map pve-vm--104--disk--0p128 (252:13): 0 478 linear 252:10 34
```

在 `pve` 中，通过 `kpartx` 挂载好的虚拟设备通常可以在 `/dev/mapper/` 中找到

**2. 扩容文件系统**

依次执行

```sh
$ e2fsck /dev/mapper/pve-vm--104--disk--0p2
e2fsck 1.47.0 (5-Feb-2023)
rootfs: clean, 1409/6656 files, 5727/26624 blocks
```

```sh
$ resize2fs /dev/mapper/pve-vm--104--disk--0p2
resize2fs 1.47.0 (5-Feb-2023)
Resizing the filesystem on /dev/mapper/pve-vm--104--disk--0p2 to 519872 (4k) blocks.
The filesystem on /dev/mapper/pve-vm--104--disk--0p2 is now 519872 (4k) blocks long.
```

**3. 修改对应启动引导**

未完待续...