## 环境
* 系统 
	* ubuntu 13.04
* 分区 
	* /
	* swap:8G (因为没有预留分区，所以从swap中分出一块4G大小的空白分区，试图从/下分，失败)

## 2.2
新建分区
* 安装Gpartend，调整swap为新分区准备4G空间，调整分区时候需卸载当前分区，因为只有/分区，所以无法卸载

相关命令
* fdisk -l # 查看磁盘情况
* fcdisk ... new->primary->write # 分区

lfs分区名称
* /dev/sd3

## 2.3
* mke2fs -jv /dev/sda3
<pre>
	mke2fs 1.42.5 (29-Jul-2012)
	fs_types for mke2fs.conf resolution: 'ext3'
	warning: 255 blocks unused.

	文件系统标签=
	OS type: Linux
	块大小=4096 (log=2)
	分块大小=4096 (log=2)
	Stride=0 blocks, Stripe width=0 blocks
	262656 inodes, 1048576 blocks
	52441 blocks (5.00%) reserved for the super user
	第一个数据块=0
	Maximum filesystem blocks=1073741824
	32 block groups
	32768 blocks per group, 32768 fragments per group
	8208 inodes per group
	Superblock backups stored on blocks: 
		32768, 98304, 163840, 229376, 294912, 819200, 884736

	Allocating group tables: 完成                            
	正在写入inode表: 完成                            
	Creating journal (32768 blocks): 完成
	Writing superblocks and filesystem accounting information: 完成 
</pre>
