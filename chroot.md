## 设置环境变量，挂载系统

	export LFS=/mnt/lfs
	mount -v -t ext3 /dev/sda3 $LFS

## 设置虚拟内核信息

	/sbin/swapon -v /dev/sda1
	mount -v --bind /dev $LFS/dev
	mount -vt devpts devpts $LFS/dev/pts
	mount -vt proc proc $LFS/proc
	mount -vt sysfs sysfs $LFS/sys

	if [ -h $LFS/dev/shm ]; then
	  link=$(readlink $LFS/dev/shm)
	  mkdir -p $LFS/$link
	  mount -vt tmpfs shm $LFS/$link
	  unset link
	else
	  mount -vt tmpfs shm $LFS/dev/shm
	fi

## chroot
重新进入有问题,先chroot "$LFS"，再执行就可以了，不会再提示/tools/bin/env不存在

	chroot "$LFS" /tools/bin/env -i \
	    HOME=/root                  \
	    TERM="$TERM"                \
	    PS1='\u:\w\$ '              \
	    PATH=/bin:/usr/bin:/sbin:/usr/sbin:/tools/bin \
	    /tools/bin/bash --login +h

	exec /tools/bin/bash --login +h


## chroot 第六章结束之后

	chroot "$LFS" /usr/bin/env -i \
	    HOME=/root TERM="$TERM" PS1='\u:\w\$ ' \
	    PATH=/bin:/usr/bin:/sbin:/usr/sbin \

	    /bin/bash --login
