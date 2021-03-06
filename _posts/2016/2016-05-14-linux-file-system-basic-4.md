---
layout: post
title: Linux File System - 4
description: Linux file system(文件系统)模块的实现和基本数据结构。关键字：Ext4，文件系统，内核，samplefs，VFS，存储。
categories: [Chinese, Software]
tags:
- [file system, driver, crash, kernel, linux, storage]
---

>转载时请包含原文或者作者网站链接：<http://oliveryang.net>

* content
{:toc}

## 1. 背景

本文将在 Sampleblk 块设备上创建 Ext4 文件系统，以 Ext4 文件系统为例，用 debugfs 和 crash 来查看 Ext4 文件系统的磁盘格式 (File System Disk Layout)。

在 [Linux File System - 3](http://oliveryang.net/2016/02/linux-file-system-basic-3) 中，Samplefs 只有文件系统内存中的数据结构，而并未规定文件系统磁盘数据格式。
而 [Linux Block Driver - 1](http://oliveryang.net/2016/04/linux-block-driver-basic-1) 则实现了一个最简的块驱动 Sampleblk。 
Sampleblk [day1 的源码](https://github.com/yangoliver/lktm/tree/master/drivers/block/sampleblk/day1)只有 200 多行，但已经可以在它上面创建各种文件系统。
由于 Sampleblk 是个 ramdisk，磁盘数据实际上都写在了驱动分配的内核内存里，因此可以很方便的使用 Linux Crash 工具来研究任意一种文件系统的磁盘格式。

## 2. 准备

### 2.1 Ram Disk 驱动

需要按照如下步骤去准备 Sampleblk 驱动，

* Linux 4.6.0 编译，安装，引导
* Sampleblk 驱动编译和加载
* 在 `/dev/sampleblk1` 上创建 Ext4 文件系统。
* mount 文件系统到 /mnt 上

以上详细过程可以参考 [Linux Block Driver - 1](http://oliveryang.net/2016/04/linux-block-driver-basic-1)。

### 2.2 调试工具

需要做的准备工作如下，

* 升级 Crash 到支持 Linux 4.6.0 内核的版本

  详细过程请参考 [Linux Crash - coding notes](http://oliveryang.net/2015/07/linux-crash-coding-notes/) 这篇文章。

* 确认 debugfs 工具是否安装

  debugfs 是 ext2, ext3, ext4 文件系统提供的文件系统调试工具，通过它我们可以不通过 mount 文件系统而直接访问文件系统的内容，它是 e2fsprogs 软件包的一部分，如果找不到请安装。
  debugfs 的详细使用说明可以通过 [debugfs man page](http://linux.die.net/man/8/debugfs) 得到。

## 3. Ext4 磁盘格式

如 [Linux File System - 3](http://oliveryang.net/2016/02/linux-file-system-basic-3) 中所述。很多磁盘文件系统的重要数据结构都存在三个层面上的实现，

1. VFS 内存层面上的实现。
2. 具体文件系统内存层面上的实现。
3. 具体文件系统磁盘上的实现。

上述 1 和 2 共同组成了文件系统的内存布局 (memory layout)，而 3 则是文件系统磁盘布局 (disk layout) 的主要部分，即本文主要关注的部分。

本小节将对 Ext4 的磁盘格式做简单介绍。Ext2/Ext3 文件系统的磁盘格式与之相似，但 Ext4 在原有版本的基础上做了必要的扩展。

如下图所示，一个 Ext4 文件系统的磁盘格式是由一个引导块和很多个 block group 组成的,

![ext4 block groups](/media/images/2016/ext4_disk_layout_1.png)

其中，上图的 boot block (引导块) 用于存放系统引导代码，这个区域并不属于文件系统。在 x86 系统，此引导代码会被 BIOS 在系统加电后加载执行。

上图中的每个 block group 又由如下格式构成，

![ext4 layout in one group](/media/images/2016/ext4_disk_layout_2.png)

接下来我们对 Ext4 磁盘上的 block group 及其内部的构成做简单介绍。

### 3.1 Block Group

一个 block group 里的存储的数据就两大类，存放于文件的 data 和用于描述 aata 的 meta data (元数据)。
而 Ext 文件系统由多个 block group 组成，每个 block group 都有部分重复的元数据，浪费了一些空间，但是却可以带来如下好处，

* 错误恢复。例如，super block 和 group descriptor 在每个 block group 都有副本，如果发生损坏时这些副本可以用于恢复。
* 性能。一个文件的存放会尽可能不跨越 block group，因此，同一文件的访问都在 block group 内部，使得数据访问具有局部性。这样减少了 HDD 设备磁头移动和寻道时间。

### 3.2 Super Block

Super block 包含了文件系统的全局配置和信息。是关联其它文件系统元数据的核心数据，即元数据的元数据。

默认情况下，文件系统使用 block group 0 的 super block 和 group descriptor 。其它每个 block group 里都有一个 super block 和 group descriptor 的副本。
为减少空间浪费，Ext 文件系统引入了 [sparse super block](https://github.com/torvalds/linux/blob/master/Documentation/filesystems/ext2.txt#L112)
的特性，这样，只有 block group 1，和 block group ID 为 2 的 3，5，7 次幂的 block group 会存副本。这时，没有存放 super block 和 group descriptor 的 block group 起始偏移将从 data block bitmap 开始。

为了性能考虑，super block 一般都缓存在内存中。本文只关注磁盘上的 super block 格式，其对应数据结构为 `struct ext4_super_block`。

### 3.3 Group Descriptor

Group descriptor 描述文件系统所有的 block group 的信息。文件系统默认使用 block group 0 里的 group descriptor。其它副本存放情况已经在 super block 章节有所描述。

通过 group descriptor 可以定位 block group 的 data block bitmap，inode bitmap 和 inode table 的所在块号。这就意味着 data block bitmap，inode bitmap 和 inode table 的位置可以不是固定的。
Ext4 文件系统的 group descriptor 对应数据结构为 `struct ext4_group_desc`。具体信息见后续章节的相关实验。

### 3.4 Data Block Bitmap

Data block bitmap 跟踪 block group 内的 data block 使用情况，是占一个 block 大小的 bitmap。已分配数据块为 1，空闲块为 0。

### 3.5 Inode Bitmap

Inode bitmap 跟踪 inode table 内的表项使用情况，是占一个 block 大小的 bitmap。已分配表项为 1，空闲表项为 0。

### 3.6 Inode Table

Inode table 存放 block group 内部的所有文件和目录的 meta data 即 inode 的信息。对 Ext4 文件系统，就是 `struct ext4_inode` 的数组。

### 3.7 Data Block

Data block 存放 block group 的所有文件的实际数据。文件的磁盘空间使用都是以 block 为单位的。文件的数据块在磁盘上连续可以减少 fragmentation 从而改善性能。
Ext4 super block 会给出文件系统每个 block 的大小。

## 4. 实验

下面我们会利用 sampleblk 驱动做简单的实验来帮助加深对 Ext4 磁盘格式的认识。

### 4.1 查看 Block Group

在格式化 Ext4 文件系统时，`mkfs.ext4` 命令已经报告了在块设备 `/dev/sampleblk1` 上创建了 1 个 block group，并且给出这个 block group 里的具体 block，fragment 和 inode 的个数,

	$ sudo mkfs.ext4 /dev/sampleblk1
	...[snipped]...

	1 block group
	8192 blocks per group, 8192 fragments per group
	64 inodes per group

	...[snipped]...

同样的，也可以使用 `debugfs` 的 `show_super_stats` 命令得到相应的信息，

	$ sudo debugfs /dev/sampleblk1 -R show_super_stats | grep -i block
	debugfs 1.42.9 (28-Dec-2013)
	Block count:              512
	Reserved block count:     25
	Free blocks:              482
	First block:              1
	Block size:               1024 /* block 的长度 */
	Reserved GDT blocks:      3
	Blocks per group:         8192
	Inode blocks per group:   8
	Flex block group size:    16
	Reserved blocks uid:      0 (user root)
	Reserved blocks gid:      0 (group root)
	 Group  0: block bitmap at 6, inode bitmap at 22, inode table at 38
	           482 free blocks, 53 free inodes, 2 used directories, 53 unused inodes

从 `debugfs` 的命令输出，我们也可以清楚的知道 block group 0 内部的情况。它的 block bitmap，inode bitmap，inode table 的具体位置。

### 4.2 磁盘起始地址和长度

因为 sampleblk 是 ramdisk，因此在此块设备上创建文件系统时，所有的数据都被写在了内存里。所以，可以利用 crash 来查看 Ext4 文件系统在 sampleblk 上的磁盘布局。

首先，需要加载 sampleblk.ko 和 ext4 模块的符号，

	crash> mod -s sampleblk /home/yango/ws/lktm/drivers/block/sampleblk/day1/sampleblk.ko
	     MODULE       NAME                   SIZE  OBJECT FILE
	ffffffffa03dd580  sampleblk              2681  /home/yango/ws/lktm/drivers/block/sampleblk/day1/sampleblk.ko

	crash> mod -s ext4 /lib/modules/4.6.0-rc3+/kernel/fs/ext4/ext4.ko
	     MODULE       NAME                   SIZE  OBJECT FILE
	ffffffffa063cc40  ext4                 574717  /lib/modules/4.6.0-rc3+/kernel/fs/ext4/ext4.ko

然后，通过驱动的全局数据结构 `struct sampleblk_dev` 即可找回磁盘在内存中的起始地址和长度，

	crash7> p *sampleblk_dev
	$1 = {
	  minor = 1,
	  lock = {
	    {
	      rlock = {
	        raw_lock = {
	          val = {
	            counter = 0
	          }
	        }
	      }
	    }
	  },
	  queue = 0xffff880034e38000,
	  disk = 0xffff880034268000,
	  size = 524288, /* 磁盘在内存中的长度 */
	  data = 0xffffc900017bc000 /*此地址为磁盘在内存中的起始地址 */
	}

	crash7> p 524288/8
	$5 = 65536                  /* 换算成字节数 */

### 4.3 查看 Super Block

根据 Ext4 磁盘布局，由于 sampleblk 上只存在一个 block group：group 0，因此，super block 存在于 group 0 上的第一个块。

首先，block group 0 距离磁盘起始位置的偏移是 boot block，而一个 block 的大小我们从前面的例子里可以得知是 1024 字节。因此，可以得到磁盘上 super block 的起始地址，

	crash> p /x 0xffffc900017bc000+1024
	$3 = 0xffffc900017bc400

然后，根据前面得到的 super block 起始地址，利用 crash 查看 `struct ext4_super_block` 数据结构的命令就可以直接查看 ext4 在磁盘上的 super block 的布局了，

	crash> ext4_super_block 0xffffc900017bc400
	struct ext4_super_block {
	  s_inodes_count = 64,
	  s_blocks_count_lo = 512,
	  s_r_blocks_count_lo = 25,
	  s_free_blocks_count_lo = 482,
	  s_free_inodes_count = 53,
	  s_first_data_block = 1,
	  s_log_block_size = 0,
	  s_log_cluster_size = 0,
	  s_blocks_per_group = 8192,
	  s_clusters_per_group = 8192,
	  s_inodes_per_group = 64,
	  s_mtime = 1464150337,
	  s_wtime = 1464150337,
	  s_mnt_count = 2,
	  s_max_mnt_count = 65535,
	  s_magic = 61267, /* 文件系统幻数 */

	  ...[snipped]...
	}

	crash> p /x 61267
	$5 = 0xef53 /* 文件系统幻数 */

而且，可以查看 `ext4_super_block` 的大小，

	crash7> ext4_super_block | grep SIZE
	SIZE: 1024

最后，用 `debugfs` 来检查 crash 查看的 super block 是否不是和 `debugfs` 一样准确，

	$ sudo debugfs /dev/sampleblk1 -R show_super_stats
	debugfs 1.42.9 (28-Dec-2013)
	Filesystem volume name:   <none>
	Last mounted on:          <not available>
	Filesystem UUID:          5c651ae8-bc3f-4fab-9e94-5b7d14c633f3
	Filesystem magic number:  0xEF53  /* 文件系统幻数 */
	Filesystem revision #:    1 (dynamic)
	Filesystem features:      ext_attr resize_inode dir_index filetype extent 64bit flex_bg sparse_super huge_file uninit_bg dir_nlink extra_isize
	Filesystem flags:         signed_directory_hash
	Default mount options:    user_xattr acl
	Filesystem state:         not clean
	Errors behavior:          Continue
	Filesystem OS type:       Linux
	Inode count:              64
	Block count:              512
	Reserved block count:     25
	Free blocks:              482
	Free inodes:              53
	First block:              1
	Block size:               1024
	Fragment size:            1024
	Group descriptor size:    64
	Reserved GDT blocks:      3
	Blocks per group:         8192
	Fragments per group:      8192
	Inodes per group:         64
	Inode blocks per group:   8
	Flex block group size:    16
	Filesystem created:       Mon May 23 08:07:40 2016
	Last mount time:          Tue May 24 21:25:37 2016
	Last write time:          Tue May 24 21:25:37 2016
	Mount count:              2
	Maximum mount count:      -1
	Last checked:             Mon May 23 08:07:40 2016
	Check interval:           0 (<none>)
	Lifetime writes:          93 kB
	Reserved blocks uid:      0 (user root)
	Reserved blocks gid:      0 (group root)
	First inode:              11
	Inode size:	          128
	Default directory hash:   half_md4
	Directory Hash Seed:      31db6c7b-1f26-4ddb-87b9-2e9a9216ba8f
	Directories:              2
	 Group  0: block bitmap at 6, inode bitmap at 22, inode table at 38
	           482 free blocks, 53 free inodes, 2 used directories, 53 unused inodes
	           [Checksum 0x01c7]

### 4.4 查看 Group Descriptor

根据 block group 的格式，可以知道，磁盘的第三个块就是 Group Descriptor 的起始地址，

	crash> p /x 0xffffc900017bc000+1024+1024
	$6 = 0xffffc900017bc800

因此用 crash 可以看到这个地址对应的磁盘上的 Group Descriptor 的内容。很容易就找到 block bitmap, inode bitmap, 和 inode table 的所在块编号,

	crash> ext4_group_desc 0xffffc900017bc800
	struct ext4_group_desc {
	  bg_block_bitmap_lo = 6,  /* block bitmap at 6 */
	  bg_inode_bitmap_lo = 22, /* inode bitmap at block 22 */
	  bg_inode_table_lo = 38,  /* inode table at block 38 */
	  bg_free_blocks_count_lo = 482,
	  bg_free_inodes_count_lo = 53,
	  bg_used_dirs_count_lo = 2,
	  bg_flags = 4,
	  bg_exclude_bitmap_lo = 0,
	  bg_block_bitmap_csum_lo = 0,
	  bg_inode_bitmap_csum_lo = 0,
	  bg_itable_unused_lo = 53,
	  bg_checksum = 455,
	  bg_block_bitmap_hi = 0,
	  bg_inode_bitmap_hi = 0,
	  bg_inode_table_hi = 0,
	  bg_free_blocks_count_hi = 0,
	  bg_free_inodes_count_hi = 0,
	  bg_used_dirs_count_hi = 0,
	  bg_itable_unused_hi = 0,
	  bg_exclude_bitmap_hi = 0,
	  bg_block_bitmap_csum_hi = 0,
	  bg_inode_bitmap_csum_hi = 0,
	  bg_reserved = 0
	}

### 4.5 查看 inode table

inode table 就是以 inode record 为元素的数组，不同版本 Ext 文件系统，这里有些差别，

* Ext2/Ext3 时代，磁盘 inode 数据结构和在 inode table 里的 inode record 长度一样，都是 128 字节。一个 record 就是一个磁盘 inode 数据结构。
* Ext4时，为兼容旧版本文件系统，同时又为扩展新能力，磁盘 inode 结构 `ext4_inode` 和 inode table 里 inode record 长度不一致了。
  此时，磁盘上的 inode record 长度在 superblock 的 `s_inode_size` 成员记录。
  因此，如果 superblock 的 `s_inode_size` 超出 128 字节，其磁盘上的 inode record 是分两部分的，第一部分可能还是 128 字节，这样可以兼容原有的 ext2/ext3 文件系统。
  第二部分的大小在 `ext4_inode` 结构的 `i_extra_isize` 成员记录。

例如，本实验中 `ext4_inode` 长度是 160 字节，但 inode record 的长度是 128 字节，因此，`ext4_inode` 结构超出 128 字节的部分并未被使用。

	crash> ext4_inode | grep SIZE
	SIZE: 160
	crash> ext4_super_block.s_inode_size 0xffffc900017bc400
	  s_inode_size = 128

从 Group Descriptor 得知，块 38 就是 inode table 的起始地址，因此可以得出其地址为，

	crash> p /x 0xffffc900017bc000+1024*38  /* Super block 里显示块长度为 1024 */
	$8 = 0xffffc900017c5800

	crash> struct ext4_inode -x 0xffffc900017c5800
	struct ext4_inode {
	  i_mode = 0x0,
	  i_uid = 0x0,
	  i_size_lo = 0x0,
	  i_atime = 0x57431cbc,
	  i_ctime = 0x57431cbc,
	  i_mtime = 0x57431cbc,
	  i_dtime = 0x0,
	  i_gid = 0x0,
	  i_links_count = 0x0,
	  i_blocks_lo = 0x0,
	  i_flags = 0x0,

用 `debugfs` 可以直接查看块号是 38 的对应 `ext4_inode` 的内容。可以看到，两个结果是一致的。

	$ sudo debugfs /dev/sampleblk1 -R 'bd 38'
	debugfs 1.42.9 (28-Dec-2013)
	0000  0000 0000 0000 0000 bc1c 4357 bc1c 4357  ..........CW..CW
	0020  bc1c 4357 0000 0000 0000 0000 0000 0000  ..CW............
	0040  0000 0000 0000 0000 0000 0000 0000 0000  ................

如果我们在新创建的文件系统中，创建一个文件。我们可以在 inode table 中定位这个文件的 inode。
如下命令可以创建一个文件，并且得到磁盘上文件对应的 inode 号,

	$ sudo touch /mnt/a
	$ sudo echo "Linux file system - 4" > /mnt/a
	$ ls -i /mnt/a
	12 /mnt/a

同样的，`debugfs` 也可以得到这个文件对应的 inode 号是 12，

	$ sudo debugfs /dev/sampleblk1 -R 'show_inode_info a'
	debugfs 1.42.9 (28-Dec-2013)
	Inode: 12   Type: regular    Mode:  0644   Flags: 0x80000
	Generation: 603494144    Version: 0x00000001
	User:     0   Group:     0   Size: 22
	File ACL: 21    Directory ACL: 0
	Links: 1   Blockcount: 4
	Fragment:  Address: 0    Number: 0    Size: 0
	ctime: 0x5745313c -- Tue May 24 21:59:40 2016
	atime: 0x5745313c -- Tue May 24 21:59:40 2016
	mtime: 0x5745313c -- Tue May 24 21:59:40 2016
	EXTENTS:
	(0):4109  /* 该文件的 Extent 区段号，稍后将会用到 */

[Ext4_Disk_Layout](https://ext4.wiki.kernel.org/index.php/Ext4_Disk_Layout) 给出了通过 inode 号定位 inode table 里的 inode record 的具体方法，

>Each block group contains sb->s_inodes_per_group inodes. Because inode 0
>is defined not to exist, this formula can be used to find the block group
>that an inode lives in: bg = (inode_num - 1) / sb->s_inodes_per_group.
>The particular inode can be found within the block group's inode table at
>index = (inode_num - 1) % sb->s_inodes_per_group. To get the byte address
>within the inode table, use offset = index * sb->s_inode_size.

因为我们 block group 只有一个，因此略过。只需要算出偏移即可，

	crash> p /x 0xffffc900017bc000+1024*38+128*(12-1)
	$14 = 0xffffc900017c5d80
	crash> struct ext4_inode -x 0xffffc900017c5d80
	struct ext4_inode {
	  i_mode = 0x81a4,
	  i_uid = 0x0,
	  i_size_lo = 0x0,
	  i_atime = 0x5745313c,
	  i_ctime = 0x5745313c,
	  i_mtime = 0x5745313c,
	  i_dtime = 0x0,
	  i_gid = 0x0,
	  i_links_count = 0x1,
	  i_blocks_lo = 0x2,
	  i_flags = 0x80000,
	[...snipped...]
	  i_block = {127754, 4, 0, 0, 1, 4109, 0, 0, 0, 0, 0, 0, 0, 0, 0},  /* 数据映射区域，B+ 树节点 */

### 4.6 查看 Extent Tree

Ext3 文件系统使用了直接、间接、二级间接和三级间接块的形式来定位磁盘中的数据块。
通过 `ext3_inode->i_block[]` 数组可以定位到该 inode 的数据存储的位置。

* i_block[0] 到 i_block[11] 存储的是 data block 号。如果文件小于12 个块时，就存储在这里。

* i_block[12] 存储的是 Indirect block (一级间接映射) ，即其指向的块存储的不是数据，而是块号。
  如果文件大于 12 个块，而小于 blocksize/4 + 12 时，i_block[12] 指向的数据有效。其存储的数据块个数为：blocksize/4.

* i_block[13] 存储的是 Double indirect block (二级间接映射). 其存储的数据块个数可以为：blocksize/4 * blocksize/4.

* i_block[14] 存储的是 Triple indirect block (三级间接映射). 其存储的数据块个数为：blocksize/4 * blocksize/4 * blocksize/4.

Ext3 的这种数据块映射和存储方式比较低效，1000 个连续的数据块也需要相应的 1000 个映射这些数据块的元数据记录。而且也因此限制了单个文件的大小。

Ext4 则引入了 Extent (区段) 的概念，使用 B+ 树来索引数据块。这时，1000 个连续的数据块只需要一个 12 字节的 `ext4_extent` 数据结构就可以描述。
通过 `ext4_inode->i_block[]` 数组可以定位到 Extent (区段) 的数据存储的位置。

* i_block[0] 到 i_block[2] 3 个元素一定是一个 `ext4_extent_header` 结构
* i_block[3] 到 i_block[15] 共 12 个元素，每 3 个元素可能是一个 `ext4_extent` 或 `ext4_extent_idx` 结构，这取决于所表示的文件的大小。

例如，本例中，`ext4_extent_header` 的 `eh_depth` 为 0，就表示后面 4 组元素都应该是 B＋ 树的叶子节点，即 `ext4_extent` 结构。

	crash> struct -o ext4_inode.i_block 0xffffc900017c5d80
	struct ext4_inode {
	  [ffffc900017c5da8] __le32 i_block[15];
	  }

	crash> ext4_extent_header -x ffffc900017c5da8
	struct ext4_extent_header {
	  eh_magic = 0xf30a,  /* Magic number, 0xF30A */
	  eh_entries = 0x1,   /* Inode 对应的 Extent 区段数 */
	  eh_max = 0x4,		  /* 最大的区段数 */
	  eh_depth = 0x0,     /* 段节点在段树中的深度。0 则表示为叶子节点，指向数据块；否则指向其它段节点 */
	  eh_generation = 0x0
	}

本例中，`eh_entries` 为 1，表明只有一个区段，且存放于 i_block[3] 到 i_block[5]，

	crash> p /x 0xffffc900017c5da8+0x12
	$13 = 0xffffc900017c5dba
	crash> struct ext4_extent 0xffffc900017c5dba
	struct ext4_extent {
	  ee_block = 0,
	  ee_len = 1,         /* Extent 里的块的个数，因为是 __le16 类型，所以一个 extent 块数是 <= 32768 */
	  ee_start_hi = 0,
	  ee_start_lo = 4109  /* 文件对应的区段的数据块号，与 debugfs 显示结果一致 */
	}

由于一个 extent 最大块数是 32768，因此对于 4k 大小的块来说，一个 extent 的最大长度就是 128M。

### 4.7 查看 Data Block

前面我们已经通过遍历 `ext4_inode->i_block[]` 的 B＋ 树节点找到了 `/mnt/a` 文件对应的数据块号。所以很容易计算出 4109 块号对应的内存地址，并且打印其内容，

	crash7> p /x 0xffffc900017bc000+4109*1024  /* Super block 里显示块长度为 1024 */
	$15 = 0xffffc90001bbf400
	crash7> rd -a 0xffffc90001bbf400
	0xffffc90001bbf400:  Linux file system - 4

可以看到，正是我们创建该文件时写入的字符串。此外，我们也可以利用 `debugfs` 工具的命令来查看指定数据块号的内容，

	$ sudo debugfs /dev/sampleblk1 -R 'bd 4109'
	debugfs 1.42.9 (28-Dec-2013)
	0000  4c69 6e75 7820 6669 6c65 2073 7973 7465  Linux file syste
	0020  6d20 2d20 340a 0000 0000 0000 0000 0000  m - 4...........
	0040  0000 0000 0000 0000 0000 0000 0000 0000  ................

至此，我们清楚地了解了 Ext4 文件系统是如何从文件的 inode 号定位到磁盘上的 inode 结构，然后定位到具体数据存放的数据块的。

## 5. 延伸阅读

* [Linux Block Driver - 1](http://oliveryang.net/2016/04/linux-block-driver-basic-1)
* [Linux File System - 1](http://oliveryang.net/2016/01/linux-file-system-basic-1)
* [Linux File System - 2](http://oliveryang.net/2016/01/linux-file-system-basic-2)
* [Linux File System - 3](http://oliveryang.net/2016/02/linux-file-system-basic-3)
* [Linux Crash - background](http://oliveryang.net/2015/06/linux-crash-background)
* [Linux Crash - coding notes](http://oliveryang.net/2015/07/linux-crash-coding-notes/)
* [Ext4 Disk Layout](https://ext4.wiki.kernel.org/index.php/Ext4_Disk_Layout)
* [在Fedora 20环境下安装系统内核源代码](http://www.cnblogs.com/kuliuheng/p/3976780.html)
* [Linux Crash White Paper (了解 crash 命令)](http://people.redhat.com/anderson/crash_whitepaper)
* [Debugfs man page](http://linux.die.net/man/8/debugfs)
