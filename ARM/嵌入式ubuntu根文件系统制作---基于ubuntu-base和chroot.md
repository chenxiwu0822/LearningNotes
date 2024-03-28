# 嵌入式ubuntu根文件系统制作---基于ubuntu-base和chroot

## 目录

1. 准备工作
2. 跟文件系统解压及挂在
3. 系统配置及常用工具
4. 开发编译库
5. 打包成.tar.gz镜像
6. 脚本文件
7. 结语
   
   <br/>

## 1.准备工作

安装软件qemu-user-static

```
安装软件qemu-user-static
```

软件及脚本准备

```
1.下载ubuntu的最小根文件系统：http://cdimage.ubuntu.com/ubuntu-base/releases/
2.拷贝以下脚本文件到工程目录下
    makefile  :拷贝相关系统文件到根文件系统中
    ch-mount.sh ：根文件系统启动脚本和卸载脚本
    sources.list ：根文件系统软件源
    passwd.list ：用户密码文件
    serial-getty@ttyPS0.service：串口终端服务
```

## 2.跟文件系统解压及挂在

整个过程中使用root权限或sudo权限，否则安装包时会提示failed，注意博主采用的是arm64的base包。

```
	sudo mkdir ubuntu-16.04-rootfs
	sudo tar -xvpf ubuntu-base-16.04-base-arm64.tar.gz
    sudo chown root:root ubuntu-16.04-rootfs
    make
    sudo chmod +x ch-mount.sh
    # 挂载根文件系统
    ./ch-mount.sh  -m  ubuntu-16.04-rootfs/
    # 退出根文件系统
    exit
    # 卸载根文件系统，重要！！！退出根文件系统一定要及时卸载
    ./ch-mount.sh  -u  ubuntu-16.04-rootfs/

```

如果使用arm32的base包，需要修改makefile中的代码，如下：

```
@sudo cp /usr/bin/qemu-arm-static  $(rootfs)/usr/bin/
```

## 3.系统配置及常用工具

进入系统后，可以配置系统基本参数，以及安装常用工具

```
#基本配置
    apt-get update
    apt-get -y install locales language-pack-en-base
	echo "LANG=en_US.UTF-8" > /etc/locale.conf
    echo "export LC_ALL=en_US.UTF-8" >> /root/.bashrc
    echo "export LC_CTYPE=en_US.UTF-8" >> /root/.bashrc
    source /root/.bashrc
    echo "ubuntu" > /etc/hostname
    echo "127.0.0.1 localhost" > /etc/hosts

    #创建用户
    addgroup admin
    useradd -m admin -g admin -d /home/admin
    usermod -s /bin/bash admin
    chpasswd</cmd/passwd.list

    #建议必须的安装库
    apt-get install -y vim sudo net-tools ethtool fish htop iputils-ping ssh ifupdown

    #建议创建终端root登陆
    cp /cmd/serial-getty@ttyPS0.service /lib/systemd/system/serial-getty@ttyPS0.service
    
    #参考编译器
    apt-get install -y gcc g++ make cmake
    apt-get install -y python  python-pip 
    apt-get install -y python3 python3-pip

```

## 4.开发编译库

```
    # 代码管理
    apt-get install  git 
    # opencv库，对应gcc或g++开发库
    apt-get install  libopencv-dev 
    # python常用库
    pip3 install  scikit-build  Cython  numpy  opencv-python

```

## 5.打包成.tar.gz镜像

```
sudo tar -cvpf ubuntu-16.04-rootfs.tar.gz ubuntu-16.04-rootfs/
```

## 6.脚本文件

makefile :拷贝相关系统文件到根文件系统中

```
all:updata
rootfs=ubuntu-16.04-rootfs2
updata:
	@sudo cp /usr/bin/qemu-aarch64-static  $(rootfs)/usr/bin/
	@sudo cp /etc/resolv.conf  $(rootfs)/etc/resolv.conf 
	@mkdir $(rootfs)/cmd
	@sudo cp ch-mount.sh $(rootfs)/cmd
	@#@sudo cp $(rootfs)/etc/opt/sources.list $(rootfs)/etc/opt/sources.list.back
	@sudo cp sources.list $(rootfs)/etc/opt/sources.list
	@sudo cp passwd.list $(rootfs)/cmd
	@sudo cp serial-getty@ttyPS0.service $(rootfs)/cmd
	@sudo chown root:root $(rootfs)

```

ch-mount.sh ：根文件系统启动脚本和卸载脚本

```
#!/bin/bash
mnt() {
	echo "MOUNTING"
	sudo mount -t proc /proc ${2}proc
	sudo mount -t sysfs /sys ${2}sys
	sudo mount -o bind /dev ${2}dev
	sudo mount -o bind /dev/pts ${2}dev/pts
	sudo chroot ${2}
}
umnt() {
	echo "UNMOUNTING"
	sudo umount ${2}proc
	sudo umount ${2}sys
	sudo umount ${2}dev/pts
	sudo umount ${2}dev
}

if [ "$1" == "-m" ] && [ -n "$2" ] ;
then
	mnt $1 $2
elif [ "$1" == "-u" ] && [ -n "$2" ];
then
	umnt $1 $2
else
	echo ""
	echo "Either 1'st, 2'nd or both parameters were missing"
	echo ""
	echo "1'st parameter can be one of these: -m(mount) OR -u(umount)"
	echo "2'nd parameter is the full path of rootfs directory(with trailing '/')"
	echo ""
	echo "For example: ch-mount -m /media/sdcard/"
	echo ""
	echo 1st parameter : ${1}
	echo 2nd parameter : ${2}
fi

```

sources.list ：根文件系统软件源

```
# See http://help.ubuntu.com/community/UpgradeNotes for how to upgrade to
# newer versions of the distribution.
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ bionic main restricted universe multiverse partner
# deb-src http://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/  bionic main restricted universe multiverse partner

deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ bionic-updates main restricted universe multiverse
# deb-src http://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/  bionic-updates main restricted universe multiverse

deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ bionic-backports main restricted universe multiverse
# deb-src http://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/  bionic-backports main restricted universe multiverse

deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ bionic-security main restricted universe multiverse
# deb-src http://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/  bionic-security main restricted universe multiverse


```

passwd.list ：用户密码文件

```
root:root
admin:admin

```

serial-getty@ttyPS0.service：串口终端服务

```
#  SPDX-License-Identifier: LGPL-2.1+
#
#  This file is part of systemd.
#
#  systemd is free software; you can redistribute it and/or modify it
#  under the terms of the GNU Lesser General Public License as published by
#  the Free Software Foundation; either version 2.1 of the License, or
#  (at your option) any later version.

[Unit]
Description=Serial Getty on %I
#Documentation=man:agetty(8) man:systemd-getty-generator(8)
#Documentation=http://0pointer.de/blog/projects/serial-console.html
#BindsTo=dev-%i.device
After=dev-%i.device systemd-user-sessions.service plymouth-quit-wait.service getty-pre.target
After=rc-local.service

# If additional gettys are spawned during boot then we should make
# sure that this is synchronized before getty.target, even though
# getty.target didn't actually pull it in.
Before=getty.target
IgnoreOnIsolate=yes

# IgnoreOnIsolate causes issues with sulogin, if someone isolates
# rescue.target or starts rescue.service from multi-user.target or
# graphical.target.
Conflicts=rescue.service
Before=rescue.service

[Service]
# The '-o' option value tells agetty to replace 'login' arguments with an
# option to preserve environment (-p), followed by '--' for safety, and then
# the entered username.
ExecStart=-/sbin/agetty -a root -8 -L 115200 %I $TERM
Type=idle
Restart=always
UtmpIdentifier=%I
TTYPath=/dev/%I
TTYReset=yes
TTYVHangup=yes
KillMode=process
IgnoreSIGPIPE=no
SendSIGHUP=yes

[Install]
WantedBy=getty.target


```