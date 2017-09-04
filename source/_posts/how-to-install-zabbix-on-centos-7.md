title: how to install zabbix on centos 7?
date: 2017-09-03 20:31:29
categories: programming
tags:
- zabbix
- devops
---

# 使用rpm包的方式进行安装：
---

## 下载rpm包和安装zabbix-server-mysql和zabbix-web-mysql

* 使用阿里云的镜像源，会比官方的镜像源快很多。

```
rpm -ivh https://mirrors.aliyun.com/zabbix/zabbix/3.2/rhel/7/x86_64/zabbix-release-3.2-1.el7.noarch.rpm
```

<!--more-->

* 验证是否rpm包是否安装成功：

```
rpm -ql zabbix-release

# output
/etc/pki/rpm-gpg/RPM-GPG-KEY-ZABBIX
/etc/pki/rpm-gpg/RPM-GPG-KEY-ZABBIX-A14FE591
/etc/yum.repos.d/zabbix.repo
/usr/share/doc/zabbix-release-3.2
/usr/share/doc/zabbix-release-3.2/GPL
```

* 安装zabbix必要的包：

```
yum install zabbix-server-mysql zabbix-web-mysql
```

## 安装mysql

参考[How to install mysql5.7 in centos](http://www.zhuxiaodong.net/2017/how-to-install-mysql5-7-in-centos/)

## 初始化zabbix的mysql数据库：

```
# 登录mysql
mysql -uroot -p

# 创建zabbix数据库
create database zabbix character set utf8 collate utf8_bin;

# 创建zabbix用户和授权
# 密码请自行设置
grant all privileges on zabbix.* to zabbix@localhost identified by '<password>';

quit;
```

## 初始化zabbix的相关数据库脚本

```
zcat /usr/share/doc/zabbix-server-mysql-3.2.*/create.sql.gz | mysql -uzabbix -p zabbix
```

## 修改zabbix的配置文件中的DBUserName：

vim /etc/zabbix/zabbix_server.conf

```
# 修改zabbix数据库的密码
DBPassword=<password>
```

## 启动zabbix-server服务并设置开机自动启动：

```
systemctl start zabbix-server
systemctl enable zabbix-server
```

## 设置zabbix的php运行时区：

```
vim /etc/httpd/conf.d/zabbix.conf

# 设置时区
php_value date.timezone Asia/Shanghai
```

## 启动apache服务并设置开机自动启动

```
systemctl start httpd
systemctl enable httpd
```

## 根据zabbix web portal进行配置

参考: https://www.zabbix.com/documentation/3.2/manual/installation/install#installing_frontend

## 为需要监控的server安装zabbix-agent

```
yum install zabbix-agent

systemctl enable zabbix-agent.service

systemctl start zabbix-agent.service
```

## 配置zabbix-agent

vim /etc/zabbix/zabbix_agentd.conf

```
##### 配置被动模式相关信息

# 设置为zabbix-server的ip address
Server=xx.xx.xx.xx

# 开启的进程数
StartAgents=3

##### 配置主动模式相关信息

##### Active checks related

# 设置为zabbix-server的ip address
ServerActive=xx.xx.xx.xx

# 设置agent的主机名称, 注意：需要和portal当中添加的host的名称一致。
# 如果配置的不一样的话, 会出现http://www.linuxidc.com/Linux/2014-05/102257.htm描述的错误.
Hostname=testweb02
```

## 安装zabbix-java-gateway

> NOTE: 
> 下一篇文章当中将介绍使用Low Level Discover的方式来自动发现并监控多台Server上多个JVM进程，而不使用zabbix-java-gateway的方式来进行监控。 

* 需要通过zabbix-java-gateway结合JMX来对java应用进行监控。
* 请将zabbix-java-gateway安装到与zabbix-server相同的机器上。
* 由于zabbix-java-gateway是一个java应用，因此需要在目标机器上安装java。

```
yum install zabbix-java-gateway

systemctl enable zabbix-java-gateway.service

systemctl start zabbix-java-gateway.service
```

## 配置zabbix java gateway.

vim /etc/zabbix/zabbix_java_gateway.conf

```
# 监听地址
LISTEN_IP=”0.0.0.0″

# 监听端口
LISTEN_PORT=10052

# PID_FILE文件
PID_FILE=”/var/run/zabbix/zabbix_java.pid”

# 开启的工作线程数
START_POLLERS=3
```

配置完成了之后，重启zabbix-java-gateway服务

```
systemctl restart zabbix-java-gateway.service
```


vim /etc/zabbix/zabbix_server.conf

```
# JavaGateway的服务器IP地址
JavaGateway=127.0.0.1

# JavaGateway的服务端口
JavaGatewayPort=10052

# 从javaGateway采集数据的进程数
StartJavaPollers=3
```

配置完成了之后，重启zabbix-server服务

```
systemctl restart zabbix-server.service
```

## 为本机的JVM进程设置JMX：

Tomcat的进程设置：

[vim CATALINA_HOME/bin/catalina.sh]

```
CATALINA_OPTS="$CATALINA_OPTS -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.port={JMX的端口}
-Djava.rmi.server.hostname={本机的IP地址}"
```

ref: 
* http://www.jianshu.com/p/e3825a885a1b
* http://wzlinux.blog.51cto.com/8021085/1692444


# 参考：
---

[zabbix官方安装文档](https://www.zabbix.com/documentation/3.2/manual/installation/install_from_packages/repository_installation)

[CentOS 7.2安装配置zabbix 3.0 LTS](https://www.centos.bz/2017/08/centos-7-2-install-zabbix-3-0-lts/)

[配置自定义的JMX模板](http://niceaz.com/zabbix%E7%9B%91%E6%8E%A7tomcat%E9%85%8D%E7%BD%AE/)

[zabbix从放弃到入门](http://www.zsythink.net/archives/category/%e8%bf%90%e7%bb%b4%e7%9b%b8%e5%85%b3/zabbix/)