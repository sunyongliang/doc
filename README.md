
## ulimit限制 ##

> `功能说明：控制shell程序的资源。`
> 
> `ulimit为shell内建指令，可用来控制shell执行程序的资源。`

**1. 可执行命令**

    ulimit -SHn 65535

**2. 编辑文件limits.conf**

    [root@www ~]# vim /etc/security/limits.conf #文末添加内容
    root soft nofile 65535
    root hard nofile 65535
    * soft nofile 65535
    * hard nofile 65535

----------

## 时间同步 ##

**安装ntpdate**

    [root@www ~]# yum -y install ntpdate

**执行命令**

    [root@www ~]# /usr/sbin/ntpdate ntp1.aliyun.com && /sbin/clock -w

**添加定时任务**

    [root@www ~]# crontab -e
    0 3 * * * /usr/sbin/ntpdate ntp1.aliyun.com

----------

## SSH反向代理 ##

    应用场景：
    通常服务器是通过nat方式访问互联网，外部无法直接远程到本地服务器，通过ssh穿透局域网访问内部服务器。

**操作步骤：**



> 	准备环境：
> 	局域网主机： 192.168.80.128  windows2008
> 	局域网主机： 192.168.80.129  centos7
> 	阿里云主机： 47.93.252.108   centos6`

`修改阿里云主机配置文件`

    [root@108 ~]# vim /etc/ssh/sshd_config
    GatewayPorts yes

`重启sshd服务`

    [root@108 ~]# service sshd restart     #centos6
	[root@108 ~]# systemctl restart sshd   #centos7

`局域网主机centos6连接到阿里云主机`

    [root@129 ~]# ssh -CqTfnN -R 0.0.0.0:2222:192.168.80.129:22 root@47.93.252.108
    输入阿里云root用户密码

`在阿里云主机查看端口监听`

    [root@108 ~]# netstat -tunlap |grep 2222
    tcp    0    0 0.0.0.0:2222    0.0.0.0:*    LISTEN     9294/sshd

`现在可以在另外一台能连网的机器连接`

    ssh -p 2222 root@47.93.252.108
    输入局域网主机的root密码
>     [root@129 ~]# ip a
>     1: lo: <LOOPBACK,UP,LOWER_UPmtu 65536 qdisc noqueue state UNKNOWN qlen 1
>         link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
>         inet 127.0.0.1/8 scope host lo
>            valid_lft forever preferred_lft forever
>         inet6 ::1/128 scope host 
>            valid_lft forever preferred_lft forever
>     2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UPmtu 1500 qdisc pfifo_fast state UP qlen 1000
>         link/ether 00:0c:29:5b:51:06 brd ff:ff:ff:ff:ff:ff
>         inet 192.168.80.129/24 brd 192.168.80.255 scope global dynamic ens33
>            valid_lft 1655sec preferred_lft 1655sec
>         inet6 fe80::20c:29ff:fe5b:5106/64 scope link 
>            valid_lft forever preferred_lft forever

`或者登录阿里云主机在本地连接`

    [root@108 ~]# ssh -p 2222 root@localhost
    输入局域网主机的root密码

`把Windows远程连接端口3389代理到阿里云主机`

    [root@129 ~]# ssh -CqTfnN -R 0.0.0.0:23389:192.168.80.128:3389 root@47.93.252.108
    输入阿里云root用户密码

`客户端使用mstsc远程连接`

    远程地址栏输入：47.93.252.108:23389
    用户名密码为Windows服务器用户密码

`#上述代理的端口在阿里云上要开通安全组访问权限`

**到这里反向代理结束。**

----------

## 端口映射 ##

	应用场景：
    通过外网来访问内网的服务

**环境准备**

	 需要有一台能够外网访问的机器做端口映射，通过数据包转发来实现外部访问阿里云的内网服务

`安装配置`

	在阿里云有公网IP的服务器上安装Rinetd
	[root@www ~]# wget http://www.boutell.com/rinetd/http/rinetd.tar.gz
	[root@www ~]# tar -xvf rinetd.tar.gz
	[root@www ~]# cd rinetd
	[root@www ~]# sed -i 's/65536/65535/g' rinetd.c    # 修改端口范围，否则会报错
	[root@www ~]# mkdir /usr/man&&make&&make install

`创建配置文件`

	[root@www ~]# vim /etc/rinetd.conf
	# allow 192.168.2.*
	# deny 192.168.1.*
	# bindaddress bindport connectaddress connectport
	0.0.0.0 13306 192.168.9.82 3306
	logfile /var/log/rinetd.log

`启动Rinetd`

	[root@www ~]# rinetd -c /etc/rinetd.conf
	# 检查进程是否启动
	[root@www ~]# ps aux | grep rinetd
	# 检查监控的端口是否开启
	[root@www ~]# netstat -tanop | grep 13306

`加入开机启动`

	[root@www ~]# echo rinetd >> /etc/rc.local

`验证`

	待rinetd启动后，就已经可以通过外网的13306端口连接到处于内网模式的192.168.9.82:3306数据库了

`注：由于FTP协议相对特殊，无法实现转发`

----------

## 文件共享 ##

`NFS是Network File System的缩写，即网络文件系统`

**环境准备**

>   NFS Server: 192.168.80.128 CentOS7
> 	NFS Client: 192.168.80.129 CentOS7

`关闭防火墙:`

	[root@docker-2 ~]# systemctl stop firewalld  #CentOS7
	[root@docker-2 ~]# service stop iptables	 #CentOS6

`SElinux:`

	[root@docker-2 ~]# vim /etc/sysconfig/selinux
	SELINUX=disabled

	[root@docker-2 ~]# getenforce 
	Disabled

**安装软件**

	[root@docker-2 ~]# yum -y install rpcbind nfs-utils

> 	#rpcbind	RPC主程序
> 	#nfs-utils	NFS服务主程序

**启动RPC服务**

	[root@docker-2 ~]# systemctl start rpcbind 	 #启动服务
	[root@docker-2 ~]# systemctl enable rpcbind	 #开机启动

`查看rpcbind服务`

	[root@docker-2 ~]# netstat -tunlap |grep rpcbind
	udp      0     0 0.0.0.0:111    0.0.0.0:*      71795/rpcbind

**启动NFS服务**

	[root@docker-2 ~]# systemctl start nfs		#启动服务
	[root@docker-2 ~]# systemctl enable nfs		#开机启动

`查看注册在RPC上的服务`

	[root@docker-2 ~]# rpcinfo -p localhost | grep nfs
    	100003    3   tcp   2049  nfs
    	100003    4   tcp   2049  nfs
    	100227    3   tcp   2049  nfs_acl
		100003    3   udp   2049  nfs
    	100003    4   udp   2049  nfs
    	100227    3   udp   2049  nfs_acl

**配置共享目录**

	[root@docker-2 ~]# mkdir -p /data/share/
	[root@docker-2 ~]# vim /etc/exports
	/data/share/ 192.168.80.129(rw,sync,no_root_squash,no_subtree_check)
	#多主机共享将192.168.80.129改为192.168.80.0/24网段

`重载NFS服务:`

	[root@docker-2 ~]# systemctl reload nfs

`确认服务、目录:`

	[root@docker-2 ~]# showmount -e
	Export list for docker-2:
	/data/share 192.168.80.129

**客户端配置**

`安装软件：`

	[root@client ~]# yum -y install rpcbind

`启动服务：`

	[root@client ~]# systemctl start rpcbind

`命令挂载：`

	[root@client ~]# mount -t nfs 192.168.80.128:/data/share/ /data/test

`开机自动挂载：`

	[root@client ~]# echo "mount -t nfs 192.168.80.128:/data/share/ /data/test" >> /etc/rc.local
	[root@client ~]# chmod a+x /etc/rc.local

----------

