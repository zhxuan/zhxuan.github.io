---
layout: post
title: 离线安装CDH5步骤吐血总结
category: Hadoop
tags: [cdh5]
keywords: cdh5
description: 
---

>  
在线安装CDH简单省事，但是蛋疼的网速让人发疯，本文根据网上许多大神的离线安装经验，通过虚拟机安装N次后，总结出安装CDH5.0.2完整安装步骤以及安装所使用的命令，在文中我尽量使用通用的命令，以备日后使用，节省时间。

###一 集群规划及系统环境
- Linux系统：Centos 6.5 64位
- JDK:      1.7.0.45 64位

- 本地源： 172.16.130.145
- CDH  ： 172.16.130.140
- 节点一： 172.16.130.131
- 节点二： 172.16.130.132
- 节点二： 172.16.130.133

###二 准备工作
####1、安装JDK（`所有节点`）
- 查看系统中有哪些OpenJdk相关包：

	```
	rpm -qa | grep java
	```
- 卸载JDK包：

	```
	rpm -e --nodeps java-1.5.0-gcj-1.5.0.0-29.1.el6.x86_64
	```
- 安装JDK:

	```
	rpm -ivh jdk-7u55-linux-x64.rpm
	```
- 配置环境变量:

	```
	echo "JAVA_HOME=/usr/java/latest/" >> /etc/environment
	```

####2、修改主机名（`所有节点`）
- 修改/etc/sysconfig/network文件
	
	```
	NETWORKING=yes
	HOSTNAME=slave2
	NETWORKING_IPV6=no
	GATEWAY=172.16.130.1
	```
- 修改hosts文件

	```
	[root@slave2 ~]# cat /etc/hosts
	127.0.0.1 localhost
	172.16.130.145 archive.cloudera.com
	172.16.130.140 cdh
	172.16.130.131 slave1
	172.16.130.132 slave2
	172.16.130.133 slave3
	```
- 重启网络
	
	```
	service network restart
	```
	
####3、关闭防火墙（`所有节点`）	
- 临时关闭

	```
	service iptables stop
	```
- 永久关闭

	```
	chkconfig iptables off
	```
	
####4、关闭SELINUX（`所有节点`）	
- 临时关闭

	```
	setenforce 0
	```
- 永久关闭
	修改/etc/selinux/config
		
	```
	SELINUX=disabled
	```	
	
####5、配置NTP（`所有节点`）

> 集群中所有的节点必须保持时间同步，否则系统会引起许多问题，CDH做为NTP服务端，slave1、slave2、slave3与CDH进行时间同步。

- slave1服务端开启（`所有节点`）
	网上有许多教程，google解决。

- 手动同步时间（`slave1、slave2、slave3`）
	
	```
	ntpdate 172.16.130.131
	```
- crontable设置同步周期（`slave1、slave2、slave3`）
	
	```
	crontab -e
	0 * * * * /usr/sbin/ntpdate 172.16.130.140  //(每个小时同步一次)
	```	
- 配置开机启动（`所有节点`）

	```
	chkconfig ntpd on
	```
####6、SSH节点互信
- 生成密钥（`在所有节点`）

	```
	ssh-keygen -t dsa -P '' -f ~/.ssh/id_dsa
	cat ~/.ssh/id_dsa.pub >~/.ssh/authorized_keys
	```
- 更改权限（`在所有节点`）

	```
	chmod 600 ~/.ssh/authorized_keys
	chmod 700 ~/.ssh/
	```
- 拷贝公钥至master节点（`在Slave1操作`）

	```
	ssh root@slave2 cat ~/.ssh/authorized_keys >> authorized_keys
	ssh root@slave3 cat ~/.ssh/authorized_keys >> authorized_keys
	```
- 拷贝master公钥拷贝至其它节点（`在Slave1操作`）
	
	```
	scp ~/.ssh/authorized_keys root@slave1:~/.ssh/	
	```
- 测试
	在各节点测试一下，ssh登陆是否不需要密码就可以访问，第一次会让输入yes，以后就可以直接登陆了
	
	```
	scp ~/.ssh/authorized_keys root@slave1:~/.ssh/	
	```
	
###三 本地Yum软件源安装Cloudera Manager 5
> 本章节内容来源于[此博客](http://blog.csdn.net/yangzhaohui168/article/details/30118175)，为了方便以后自己查找，故转到本文，根据这些博文，成功完成本地yum安装cdh5以及hadoop集群的安装，在此对原作者表示深深的感谢。

###四 CDH5上安装Hive,HBase,Impala,Spark等服务
本章节内容来源于[此博客](http://blog.csdn.net/yangzhaohui168/article/details/33403555)，与第三节是同一出处。

###五 总结
- 2011年研究生期间使用10台PC机用Apache安装方式来搭建集群，当时hadoop刚起步资料较少，经过一周的时间才搭建好，步骤很繁琐，而且后期使用中，经常出现一些莫明的错误，包括系统版本中文、时间不同步等等问题。

- 通过使用CDH安装后，安装和维护集群变的很简单，比如我修改HDFS块大小为64M，在CDH管理界面上会显示出**系统配置过期，需要重启**这样的提示，点击确定重启，即完成配置的修改，非常非常的方便。而且CDH方式安装集群比较稳定。

> 最后，感谢同学小黑，让我重新找到奋斗的方向；感谢前人经验和指点；最后，感谢家人给我的支持!   
 谢你们，五台笔记本搭建的集群，是我人生的新起点，我会一直坚持下去，直到实现我的梦想！















