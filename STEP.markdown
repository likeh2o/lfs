## 记录说明
readme中的翻译文档已经很清晰了，这里只记录必要步骤以及碰到问题的解决方法

## 环境
* 系统 
	* ThinkPad-X61 Intel(R) Core(TM)2 Duo CPU     T7300  @ 2.00GHz
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
````
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
````
* debugfs -R feature /dev/sda3
````
	debugfs 1.42.5 (29-Jul-2012)
	Filesystem features: has_journal ext_attr resize_inode dir_index filetype sparse_super large_file
````
* 使用宿主系统swap，否则：mkswap /dev/xxx

## 2.4
* export LFS=/mnt/lfs (该命令放入.bashrc)
* mkdir -pv $LFS
* mount -v -t ext3 /dev/sda3 $LFS
* /sbin/swapon -v /dev/sda1
	* 直接使用这个命令会提示“swapon failed: 设备或资源忙”，用gparted停用swap，命令通过，先记着，后续看原因
* mount命令查看mount状态

## 3.1
* mkdir -v $LFS/sources
* chmod -v a+wt $LFS/sources

###### 获取文件列表,文件对应的md5校验码 (睡觉去，这网速要下好一会儿)
* wget http://www.linuxfromscratch.org/lfs/view/7.3/wget-list
	* 列表里有些文件已经不存在了，这里有全部的文件：ftp://ftp.lfs-matrix.net/pub/lfs/lfs-packages/7.3/
* wget -i wget-list -P $LFS/sources

###### 校验 (一般都不会有问题)
* pushd $LFS/sources
* wget http://www.linuxfromscratch.org/lfs/view/7.3/md5sums
* md5sum -c md5sums
* popd 
###### 之前不知道gnu是何物...
	* http://www.gnu.org

## 4.2
* mkdir -v $LFS/tools
* ln -sv $LFS/tools /

## 4.3 设置环境变量
* groupadd lfs
* useradd -s /bin/bash -g lfs -m -k /dev/null lfs
* passwd lfs
* chown -v lfs $LFS/tools
* chown -v lfs $LFS/sources
* su - lfs

## 4.4
###### 注意/etc/profile,/etc/bashrc,.bash_profile,.bashrc的区别
````
cat > ~/.bash_profile << "EOF"
exec env -i HOME=$HOME TERM=$TERM PS1='\u:\w\$ ' /bin/bash
EOF
````

* set +h 关闭bash的hash功能

````
cat > ~/.bashrc << "EOF"
set +h
umask 022
LFS=/mnt/lfs
LC_ALL=POSIX
LFS_TGT=$(uname -m)-lfs-linux-gnu
PATH=/tools/bin:/bin:/usr/bin
export LFS LC_ALL LFS_TGT PATH
EOF
````
* source ~/.bash_profile

## 4.4 咱是双核 	
* export MAKEFLAGS='-j 2' 
* make -j2


## 5.4 Binutils 第一遍
````
tar -xvf binutils-2.23.1.tar.bz2
cd binutils-2.23.1
mkdir -v ../binutils-build
cd ../binutils-build/
../binutils-2.23.1/configure     \
    --prefix=/tools            \
    --with-sysroot=$LFS        \
    --with-lib-path=/tools/lib \
    --target=$LFS_TGT          \
    --disable-nls              \
    --disable-werror
make
# x86_64
mkdir -v /tools/lib && ln -sv lib /tools/lib64
make install
````
## 5.5 GCC 第一遍
* 流水帐

````
tar -Jxf ../mpfr-3.1.1.tar.xz
mv -v mpfr-3.1.1 mpfr
tar -Jxf ../gmp-5.1.1.tar.xz
mv -v gmp-5.1.1 gmp
tar -zxf ../mpc-1.0.1.tar.gz
mv -v mpc-1.0.1 mpc

for file in \
 $(find gcc/config -name linux64.h -o -name linux.h -o -name sysv4.h)
do
  cp -uv $file{,.orig}
  sed -e 's@/lib\(64\)\?\(32\)\?/ld@/tools&@g' \
      -e 's@/usr@/tools@g' $file.orig > $file
  echo '
#undef STANDARD_STARTFILE_PREFIX_1
#undef STANDARD_STARTFILE_PREFIX_2
#define STANDARD_STARTFILE_PREFIX_1 "/tools/lib/"
#define STANDARD_STARTFILE_PREFIX_2 ""' >> $file
  touch $file.orig
done

sed -i '/k prot/agcc_cv_libc_provides_ssp=yes' gcc/configure
sed -i 's/BUILD_INFO=info/BUILD_INFO=/' gcc/configure
mkdir -v ../gcc-build
cd ../gcc-build

../gcc-4.7.2/configure         \
    --target=$LFS_TGT          \
    --prefix=/tools            \
    --with-sysroot=$LFS        \
    --with-newlib              \
    --without-headers          \
    --with-local-prefix=/tools \
    --with-native-system-header-dir=/tools/include \
    --disable-nls              \
    --disable-shared           \
    --disable-multilib         \
    --disable-decimal-float    \
    --disable-threads          \
    --disable-libmudflap       \
    --disable-libssp           \
    --disable-libgomp          \
    --disable-libquadmath      \
    --enable-languages=c       \
    --with-mpfr-include=$(pwd)/../gcc-4.7.2/mpfr/src \
    --with-mpfr-lib=$(pwd)/mpfr/src/.libs
    
    ln -sv libgcc.a `$LFS_TGT-gcc -print-libgcc-file-name | sed 's/libgcc/&_eh/'`
````

## 5.6 Linux API Headers （linux内核暴露的接口）这一节比较迷茫

    make mrproper # Delete the current configuration, and all generated files
    make headers_check
    make INSTALL_HDR_PATH=dest headers_install
    cp -rv dest/include/* /tools/include
    
