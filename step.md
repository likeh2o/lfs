## 记录说明
readme中的翻译文档已经很清晰了，这里只记录必要步骤以及碰到问题的解决方法

## 环境
* 系统 
	* ThinkPad-X61 Intel(R) Core(TM)2 Duo CPU     T7300  @ 2.00GHz
	* ubuntu 13.04
	* lfs 7.3
* 分区 
	* /
	* swap:8G (因为没有预留分区，所以从swap中分出一块4G大小的空白分区，试图从/下分，失败)

## 2.2
新建分区
* 安装Gpartend，调整swap为新分区准备4G空间，调整分区时候需卸载当前分区，因为只有/分区，所以无法卸载

相关命令
* fdisk -l # 查看磁盘情况
* cfdisk ... new->primary->write # 分区

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

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
    
    make
    make install
    
    ln -sv libgcc.a `$LFS_TGT-gcc -print-libgcc-file-name | sed 's/libgcc/&_eh/'`
````

## 5.6 Linux API Headers (内核暴露的接口) 这一节比较迷茫

    make mrproper # Delete the current configuration, and all generated files
    make headers_check
    make INSTALL_HDR_PATH=dest headers_install
    cp -rv dest/include/* /tools/include
    
## 5.7 Glibc 提供内存分配，文件读写等等基础

    mkdir -v ../glibc-build
    cd ../glibc-build
    ../glibc-2.17/configure                             \
      --prefix=/tools                                 \
      --host=$LFS_TGT                                 \
      --build=$(../glibc-2.17/scripts/config.guess) \
      --disable-profile                               \
      --enable-kernel=2.6.25                          \
      --with-headers=/tools/include                   \
      libc_cv_forced_unwind=yes                       \
      libc_cv_ctors_header=yes                        \
      libc_cv_c_cleanup=yes
      
    make 
    make install
      
      
测试安装是否正确
    
    
    echo 'main(){}' > dummy.c
    $LFS_TGT-gcc dummy.c
    readelf -l a.out | grep ': /tools'
        [Requesting program interpreter: /tools/lib64/ld-linux-x86-64.so.2]
    rm -v dummy.c a.out


## 5.8 Binutils

    mkdir -v ../binutils-build
    cd ../binutils-build
    CC=$LFS_TGT-gcc            \
    AR=$LFS_TGT-ar             \
    RANLIB=$LFS_TGT-ranlib     \
    ../binutils-2.23.1/configure \
      --prefix=/tools        \
      --disable-nls          \
      --with-lib-path=/tools/lib
      
    make 
    make install
    
Now prepare the linker for the “Re-adjusting” phase in the next chapter

    make -C ld clean
    make -C ld LIB_PATH=/usr/lib:/lib
    cp -v ld/ld-new /tools/bin
    
    
## 5.9 GCC 看得迷惑

    cat gcc/limitx.h gcc/glimits.h gcc/limity.h > \
      `dirname $($LFS_TGT-gcc -print-libgcc-file-name)`/include-fixed/limits.h
      
    cp -v gcc/Makefile.in{,.tmp}
    sed 's/^T_CFLAGS =$/& -fomit-frame-pointer/' gcc/Makefile.in.tmp \
      > gcc/Makefile.in
      
    for file in \
    $(find gcc/config -name linux64.h -o -name linux.h -o -name sysv4.h
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
	
下面这些不用处理，用第一次安装时候的源文件目录即可

    tar -Jxf ../mpfr-3.1.1.tar.xz
    mv -v mpfr-3.1.1 mpfr
    tar -Jxf ../gmp-5.1.1.tar.xz
    mv -v gmp-5.1.1 gmp
    tar -zxf ../mpc-1.0.1.tar.gz
    mv -v mpc-1.0.1 mpc
    
不要编译.info文件

    sed -i 's/BUILD_INFO=info/BUILD_INFO=/' gcc/configure
    
第一次编译用文件夹需删除

    mkdir -v ../gcc-build
    cd ../gcc-build
    CC=$LFS_TGT-gcc \
	AR=$LFS_TGT-ar                  \
	RANLIB=$LFS_TGT-ranlib          \
	../gcc-4.7.2/configure          \
	    --prefix=/tools             \
	    --with-local-prefix=/tools  \
	    --with-native-system-header-dir=/tools/include \
	    --enable-clocale=gnu        \
	    --enable-shared             \
	    --enable-threads=posix      \
	    --enable-__cxa_atexit       \
	    --enable-languages=c,c++    \
	    --disable-libstdcxx-pch     \
	    --disable-multilib          \
	    --disable-bootstrap         \
	    --disable-libgomp           \
	    --with-mpfr-include=$(pwd)/../gcc-4.7.2/mpfr/src \
	    --with-mpfr-lib=$(pwd)/mpfr/src/.libs
    make
    make install
    
    ln -sv gcc /tools/bin/cc
    
测试

	echo 'main(){}' > dummy.c
	cc dummy.c
	readelf -l a.out | grep ': /tools'
	
输出

	 [Requesting program interpreter: /tools/lib64/ld-linux-x86-64.so.2]
	 
	
## 5.10

	cd unix
	./configure --prefix=/tools
	make
	TZ=UTC make test
	make install
	chmod -v u+w /tools/lib/libtcl8.6.so
	make install-private-headers
	ln -sv tclsh8.6 /tools/bin/tclsh
	
	
## 5.13
按照手册执行make的时候会报错，使用如下命令
http://www.linuxfromscratch.org/lfs/errata/stable/

	CFLAGS="-L/tools/lib -lpthread" ./configure --prefix=/tools
	CFLAGS="-L/tools/lib -lpthread" make
	make install

## 5.15
编译找不到yacc，一并把flex也装上

	sudo apt-get install byacc flex
	
## 5.29 sed
因为没有空间可用了，删除sources下面所有的文件夹

	for i in `ls -l|grep drwxr-xr-x|awk '{print $9}'`;do rm -rf $i; done;
	
## 5.31 textinfo
编译的时候提示perl版本不够，又重新编译了perl才通过

## 5.34
后续使用root操作，并备份 $LFS/tools

	chown -R root:root $LFS/tools
	
	
## 6.2
man mount多看看mount的用法

## 6.4 chroot
第一次接触chroot命令

## 6.5 这个时候适合一开始提到的

[Filesystem Hierarchy Standard](http://www.pathname.com/fhs/pub/fhs-2.3.html)
[文件系统层次结构标准](http://zh.wikipedia.org/wiki/%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%E5%B1%82%E6%AC%A1%E7%BB%93%E6%9E%84%E6%A0%87%E5%87%86)


	
执行的时候，下面这句总是报错，没看到那里有问题，直接执行case的命令即可

	case $(uname -m) in
 	x86_64) ln -sv lib /lib64 && ln -sv lib /usr/lib64 ;;
	esac
	
## 6.7
两行看似异常的数据，一个是跟重启有关，一个跟声音有关，暂不理会

	/sources/linux-3.8.1/usr/include/linux/kexec.h:49: userspace cannot reference function or variable defined in the kernel
	/sources/linux-3.8.1/usr/include/linux/soundcard.h:1054: userspace cannot reference function or variable defined in the kernel
	
## 6.9
tee glibc-check-log 写入标准输出同时写入文件
make 的时候报错
	
	 /tools/lib/gcc/x86_64-unknown-linux-gnu/4.7.2/../../../../x86_64-unknown-linux-gnu/bin/ar x ../libc_pic.a; \
 	rm $(/tools/lib/gcc/x86_64-unknown-linux-gnu/4.7.2/../../../../x86_64-unknown-linux-gnu/bin/ar t ../sunrpc/librpc_compat_pic.a | sed 's/^compat-//'); \
 	/tools/lib/gcc/x86_64-unknown-linux-gnu/4.7.2/../../../../x86_64-unknown-linux-gnu/bin/ar x ../sunrpc/librpc_compat_pic.a; \
 	/tools/lib/gcc/x86_64-unknown-linux-gnu/4.7.2/../../../../x86_64-unknown-linux-gnu/bin/ar cr libc_pic.a *.os; \
 	rm *.os)
	/bin/sh: command substitution: line 3: syntax error near unexpected token `)'
	/bin/sh: command substitution: line 3: `/tools/lib/gcc/x86_64-unknown-linux-gnu/4.7.2/../../../../x86_64-unknown-linux-gnu/bin/ar t ../sunrpc/librpc_compat_pic.a | sed 's/^compat-//')'
	make[1]: *** [/sources/glibc-build/linkobj/libc_pic.a] Error 1
	make[1]: Leaving directory `/sources/glibc-2.17'
	make: *** [all] Error 2
	
这个时候想起来，6.5节的时候也报错了，推测是/bin/sh的错误，尝试重新编译bash，网上有很多人碰到这样的问题。

重新编译，然后就不能chroot了，准备重头来过

	/tools/bin/env: /tools/bin/bash: No such file or directory
	
## 原因排查
编译到bash的时候发现，还是不能正确编译），google到，在5.15状态的yacc是不对的，要装bison，然后从tcl重新编译

	sudo apt-get remove byacc
	sudo apt-get install bison

参看:http://jhshi.me/2012/09/18/lfs-6-9-1-command-substitution-line-3-syntax-error-near-unexpected-token/

## 6.9 redo
make 通过
没有必要全部安装语言包，好慢

## 插曲，编译速度相当慢，以后有需要弄一个自动安装的东西

## 6.14
Note 

## 6.17 
很慢很慢
这一节比较无语，有错误不可避免？

## 6.27
下面命令check的时候，执行到 PASS: tests/rm/v-slash.sh ，卡住了，实在等不下去了，先ctrl+c，继续吧.
	
	su nobody -s /bin/bash \
          -c "PATH=$PATH make RUN_EXPENSIVE_TESTS=yes check"

## 6.33
测试有异常，忽略之

## 6.49
Move some programs that do not need to be on the root filesystem
why???

## 6.50
arpd 需要Berkeley DB，因为之前调查过这个数据库，这里记一下

## 6.62 vim最后一个包

## 6.63,6.64 留着debug

## 6.65 
看到这里才更明白，6.2.2,6.2.3

## 电脑关机，昨天整lfs一直没有关机，笔记本过热，把主板烧了，换了两个电容，so，今天关机。

## 明天第七章 7. Setting Up System Bootscripts

## 7.5 
第7章一直看的比较晕

## 8.3 内核了
发现 grep 命令没有安装，还好之前没有把/tools下面的命令删除。退出chroot，然后用第一次chroot的命令重新进入，  
重新编译安装grep，然后退出，用后一次的chroot命令进入。

make LANG=<host_LANG_value> LC_ALL= menuconfig  
$LANG = zh_CN.UTF-8，这里一定要注意之前 7.13设置 /etc/profile 的时候写对即可。

这个命令执行之后，会出现一个内核配置的界面，再选择上面让提示注意的内容。

## 8.4 使用grub设置引导过程
让创建一个引导设备，太麻烦，掠过。按着步骤向下，按着自己磁盘分区的情况创建grub配置文件：

	cat > /boot/grub/grub.cfg << "EOF"
	# Begin /boot/grub/grub.cfg
	set default=0
	set timeout=5
	
	insmod ext2
	set root=(hd0,3) # 最开始这里写错了，写成2了，主要是没理解grub对分区的处理，这里需要补习了。
	
	menuentry "GNU/Linux, Linux 3.8.1-lfs-7.3" {
	        linux   /boot/vmlinuz-3.8.1-lfs-7.3 root=/dev/sda3 ro
	}
	EOF
	
这个文件一开始写错了，进入不了系统。

## 9.3 重启系统
shutdown -r now 之后，迎接我的是 error no such partition  
在选择启动哪个系统的界面（这里因为只有lfs，所以ubuntu找不到了），e 是编辑grub信息，c是命令行。  
进入命令行模式，使用如下命令，启动lfs：

	set root=(hd0,3)
	linux   /boot/vmlinuz-3.8.1-lfs-7.3 root=/dev/sda3 ro
	boot
	
注意：以上路径可以tab健提示。成功启动，但是悲剧的是root没有设置密码，也或许是忘记了。  
简单起见，准备进到ubuntu，然后重新chroot，重新设置root密码。

	set root=(hd0,5)
	linux   /boot/vmlinuz-3.8.0-27-generic root=/dev/sda5 ro
	initrd  /boot/initrd.img-3.8.0-27-generic
	boot
	
还好有强大的tab提示。进入ubuntu，进行密码设置。  
同时修改grub.cfg文件，让lfs，ubutnu在进入系统的时候是可选的。  
修改后的lfs系统的文件grub配置文件（/boot/grub/grub.cfg，在ubuntu看就是/mnt/lfs/boot/...）如下：

	# Begin /boot/grub/grub.cfg
	set default=0
	set timeout=10
	
	menuentry "GNU/Linux, Ubuntu" {
		insmod ext2
		set root=(hd0,5)
	        linux   /boot/vmlinuz-3.8.0-27-generic root=/dev/sda5 ro
		initrd  /boot/initrd.img-3.8.0-27-generic
	}
	
	menuentry "GNU/Linux, Linux 3.8.1-lfs-7.3" {
		insmod ext2
		set root=(hd0,3)
	        linux   /boot/vmlinuz-3.8.1-lfs-7.3 root=/dev/sda3 ro
	}

重新启动成功。

## 9.4 接下来做什么
这个不是文档中的接下来做什么  
在安装的过程中，基本都是一知半解的，需要回去咀嚼才行。  
在回味完之后，在看本节。
