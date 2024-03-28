# 移植ubuntu16.04 到arm

ubuntu-base是一个基础的Ubuntu系统，可以理解为最小的Ubuntu系统，本文适用所有arm/aarch64

## 1.下载ubuntu for arm的源码

```
[wget方式 32 for arm](wget http://cdimage.ubuntu.com/ubuntu-base/releases/16.04.4/release/ubuntu-base-16.04.4-base-armhf.tar.gz )
[wget方式 64 for arm](wget http://cdimage.ubuntu.com/ubuntu-base/releases/16.04.1/release/ubuntu-base-16.04.2-base-arm64.tar.gz )
```

## 2.安装qemu-user-static

```
sudo apt-get install qemu-user-static
```

## 3.解压ubuntu源码到自己的目录

```
mkdir ubu_s5p6818
tar -xvf ubuntu-base-16.04.6-base-arm64.tar.gz -C ubu_s5p6818/

拷贝qemu-arm-static/qemu-aarch64-static到刚刚解压出来的目录
cp /usr/bin/qemu-arm-static ubu_s5p6818/usr/bin/
为了制作成功的根文件系统能够联网，可以直接拷贝本机的dns配置文件到根文件系统的相应位置
cp /etc/resolv.conf ubu_s5p6818/etc/resolv.conf 
```

## 4.qemu挂载

挂在根文件系统并chroot，首先在本机挂载刚刚下载好的文件系统，联网完成相应的配置，然后载烧录到开发板上，需要挂载proc, sys, dev, dev/pts等文件系统，可以写个脚本，如下

```
#!/bin/bash 
mnt () 
{ 
	echo "MOUNTING" 
	sudo mount -t proc /proc ${2}proc 
	sudo mount -t sysfs /sys ${2}sys 
	sudo mount -o bind /dev ${2}dev 
	sudo mount -o bind /dev/pts ${2}dev/pts 
	sudo chroot ${2} 
}
umnt () 
{ 
	echo "UNMOUNTING" 
	sudo umount ${2}proc 
	sudo umount ${2}sys 
	sudo umount ${2}dev/pts 
	sudo umount ${2}dev 
} 

if [ "$1" = "-m" ] && [ -n "$2" ]; 
then 
	mnt $1 $2 
	echo "mnt -m pwd" 
elif [ "$1" = "-u" ] && [ -n "$2" ]; 
then 
	umnt $1 $2 
	echo "mnt -u pwd" 
else 
	echo "" 
	echo "Either 1'st, 2'nd or bothparameters were missing" 
	echo "" 
	echo "1'st parameter can be one ofthese: -m(mount) OR -u(umount)" 
	echo "2'nd parameter is the full pathof rootfs directory(with trailing '/')" 
	echo "" 
	echo "For example: ch-mount -m/media/sdcard/" 
	echo "" 
	echo 1st parameter : ${1} 
	echo 2nd parameter : ${2} 
fi
```

```
然后执行脚本
sh ms.sh -m $(xxxx)/ubuntu16.04/
终端响应如下

zw@zw-pc:~/work/v1.2/ubuntu$ sudo sh ms.sh -m ./ubuntu16.04/
[sudo] password for zw: 
MOUNTING
root@zw-pc:/# ls
bin  boot  dev	etc  home  lib	media  mnt  opt  proc  root  run  sbin	srv  sys  tmp  usr  var
root@zw-pc:/# 
```

<br/>

## 5.更新源并安装必要的软件

```
apt-get update


apt-get install python3
apt-get install vim
apt-getinstall net-tools
apt-get install iputils-ping
apt-get install iproute2
apt-get install isc-dhcp-client
apt-get install telnetd
```

<br/>

## 6.设置root密码，增加新的普通用户

```
passwd root
adduser xxxxx
```

## 7.修改/etc/fstab

根据自己的情况

```
# stock fstab - you probably want to override this with a machine specific one

/dev/root            /                    auto       defaults              1  1
proc                 /proc                proc       defaults              0  0
devpts               /dev/pts             devpts     mode=0620,gid=5       0  0
tmpfs                /run                 tmpfs      mode=0755,nodev,nosuid,strictatime 0  0
tmpfs                /var/volatile        tmpfs      defaults              0  0

# uncomment this if your device has a SD/MMC/Transflash slot
dev/mmcblk0p3       /media/card          ext4       defaults  0  0

```

<br/>

## 8.执行exit和解绑

```
exit
sudo sh ms.sh -u $(xxxx)/ubuntu16.04

终端回应如下:
root@zw-pc:/# exit
exit
mnt -m pwd
zw@zw-pc:~/work/v1.2/ubuntu$ sudo sh ms.sh -u ./ubuntu16.04/
[sudo] password for zw: 
UNMOUNTING
mnt -u pwd
zw@zw-pc:~/work/v1.2/ubuntu$ 
```

<br/>

## 遇到的问题

```

xxxx is not in the sudoers file. This incident will be reported.
解决方法:
在/etc/sudoer文件中添加如下:
# Members of the admin group may gain root privileges
%admin ALL=(ALL) ALL
autobrain ALL=(ALL) ALL

# Allow members of group sudo to execute any command
%sudo   ALL=(ALL:ALL) ALL




```

1.提示

```
xxxx is not in the sudoers file. This incident will be reported.
```

解决方法
在/etc/sudoer文件中添加如下

```
# Members of the admin group may gain root privileges
%admin ALL=(ALL) ALL
autobrain ALL=(ALL) ALL

# Allow members of group sudo to execute any command
%sudo   ALL=(ALL:ALL) ALL
```

<br/>

<br/>

2.提示

```
sudo: unable to resolve host xxxx但sudo 还是可以正常执行,
```

解决方法
在/etc/hosts添加

```
127.0.0.1 localhost.localdomain localhost
127.0.0.1 autobrain
```

<br/>

3.串口无法进入到控制台
解决方法

```
cp /lib/systemd/system/serial-getty@.service /lib/systemd/system/serial-getty@ttyS2.service
ln -s /lib/systemd/system/serial-getty@ttyS2.service /etc/systemd/system/getty.target.wants/
再修改/lib/systemd/system/serial-getty@ttyS2.service把里面的“%i.device”改为“%i”
```

4.chroot: failed to run command ‘/bin/bash’: No such file or directory

查看/bin/bash文件所依赖的动态链接库，然后依次拷贝到相应目录。

```
$ ldd /bin/bash
linux-vdso.so.1 (0x00007fffce3cd000)
/lib/$LIB/liblsp.so => /lib/lib/x86_64-linux-gnu/liblsp.so (0x00007f9cbd3c1000)
libtinfo.so.5 => /lib/x86_64-linux-gnu/libtinfo.so.5 (0x00007f9cbd197000)
libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007f9cbcf93000)
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f9cbcba2000)
/lib64/ld-linux-x86-64.so.2 (0x00007f9cbd8e0000)

```

```
$ mkdir lib64
$ cp /lib64/ld-linux-x86-64.so.2 ./lib64/
$ mkdir ./lib/x86_64-linux-gnu
$ cp /lib/x86_64-linux-gnu/liblsp.so ./lib/x86_64-linux-gnu/
$ cp /lib/x86_64-linux-gnu/libtinfo.so.5 ./lib/x86_64-linux-gnu/
$ cp /lib/x86_64-linux-gnu/libdl.so.2 ./lib/x86_64-linux-gnu/
$ cp /lib/x86_64-linux-gnu/libc.so.6 ./lib/x86_64-linux-gnu/ 
$ cp /bin/bash ./bin

```

检验结果：

```
$ sudo chroot .
bash-4.4# 
```