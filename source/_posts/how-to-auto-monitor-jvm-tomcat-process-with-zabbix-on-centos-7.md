title: how to auto monitor jvm tomcat process with zabbix on centos 7?
date: 2017-09-03 20:46:27
categories: programming
tags:
- zabbix
- devops
- low level discovery
---

# 背景
---

上一篇[文章](http://www.zhuxiaodong.net/2017/how-to-install-zabbix-on-centos-7/)当中，我们介绍了如何安装 zabbix ，以及如何使用 zabbix-java-gateway 来监控 jvm进 程，但是这种方式有如下的缺点：

* 如果一台机器上有多个 jvm 进程需要监控，会需要进行大量重复地手工配置工作，繁琐并容易出错。
* 需要依赖于 zabbix-java-gateway，多台机器上安装有一定的工作量。

利用 zabbix 的 low level discovery 机制，能够很好地解决这个问题。

<!--more-->

# 一些前期研究方案的过程中，发现的坑
---

无论 google 还是 baidu ，基本上能够搜索出的关于 Low Level Discovery 机制运用在自动监控**多个** tomcat or jvm 进程的文章大概有如下几篇：

* [zabbix监控多个tomcat实例--自动发现](http://825536458.blog.51cto.com/4417836/1945905)
* [监控一台主机上多个tomcat实例5.11修正一个语法问题](https://www.iyunv.com/thread-227674-1-1.html)

这些文章当中介绍的方式存在如下几个问题：

1. 安装步骤介绍地不是非常详细。
2. bash 脚本存在一定的问题。
3. 缺少对应的 template 文件（或者是说无法下载）。

庆幸的是，笔者在 github 上找到了一份比较靠谱和完善的信息：[qiueer/zabbix](https://github.com/qiueer/zabbix)

在此，感谢作者 qiueer 提供了一份非常好的学习资料。

# 实验环境说明
---

所有的Lab实验机器都在 Mac OS X 下使用 Parallel Desktop 搭建， Linux 版本为

```
rpm --query centos-release

# output
centos-release-7-2.1511.el7.centos.2.10.x86_64

uname -r

# output
Centos 7 (3.10.0-327.4.5.el7.x86_64)
```

| Host Name | IP | Hardware | Memo |
| --- | --- | --- | --- |
| lab-ci-monitor | 10.211.55.6  | 2 core, 4GB memory, 10GB disk | 部署jenkins和zabbix，**并作为外网唯一访问的跳板机** |
| lab-mysql01 | 10.211.55.7 | 2 core, 4GB memory, 10GB disk | 部署mysql |
| lab-web01 | 10.211.55.8 | 2 core, 4GB memory, 10GB disk | 部署java web application |
| lab-web02 | 10.211.55.9 | 2 core, 4GB memory, 10GB disk | 部署java web application |
| lab-lb | 10.211.55.10 | 2 core, 4GB memory, 10GB disk | 部署nginx |


# 安装 zabbix-server 和 zabbix-agent
---

参考上一篇[文章](http://www.zhuxiaodong.net/2017/how-to-install-zabbix-on-centos-7/)

# 使用 Active agent auto-registration 机制自动注册 zabbix agent
---

在zabbix当中，如果我们需要对大量的机器进行监控，有两种方式可以自动化地完成该操作：

* Network discovery：
https://www.zabbix.com/documentation/3.2/manual/discovery/network_discovery

* Active agent auto-registration：
https://www.zabbix.com/documentation/3.2/manual/discovery/auto_registration

在 cloud 环境下，尤其是那种目标监控机器无法提前知道 IP 地址的情况下，非常适合使用 auto-registration 机制。下面我们就来详细介绍具体如何使用。

## 步骤1：在 lab-web01 ， lab-web02 上设置 zabbix_agentd.conf

以 lab-web01 为例：

[vim /etc/zabbix/zabbix_agentd.conf]

```
# 设置 ServerActive 为 zabbix-server 的 IP 地址
ServerActive=10.211.55.6

Hostname=lab-web01

HostMetadataItem=system.uname
```

配置好了之后，需要重启 zabbix-agent：

```
systemctl restart zabbix-agent
```

## 步骤2：在 zabbix web portal 上配置 actions

### step 1：在 Configuration --> Actions 面板下，点击[Create action]，注意: Event source 选择 Auto registration

![zabbix-find-action](http://static.zhuxiaodong.net/blog/static/images/zabbix-find-action.png)

### step 2：在 action tab 页下，填写创建的名称，以及自动发现的规则。在这里我们将所有 Host name 的字符串中，包含： lab-web 字符的 zabbix agent 能够被自动发现。

![zabbix-create-action-1](http://static.zhuxiaodong.net/blog/static/images/zabbix-create-action-1.png)

### step 3：在 Operation tab 页下，填写当自动发现 zabbix agent 了之后，执行的相关操作。在这里，我们执行的操作为：

1. Add Host：自动添加该 agent 
2. Add to host groups：自动添加至 lab-java-web-server host group 中
3. Link to template：自动将对应的 host 添加 Template OS Linux 模板

![zabbix-create-action-2](http://static.zhuxiaodong.net/blog/static/images/zabbix-create-action-2.png)

完成了之后，大概等一定的时间之后（具体的时间间隔根据 zabbix_agentd.conf 配置文件当中的：RefreshActiveChecks=60 节点值来决定）， 就能够看到对应的 Host 已经被自动添加。

![zabbix-create-action-3](http://static.zhuxiaodong.net/blog/static/images/zabbix-create-action-3.png)

# 使用 Low level discovery 机制自动监控多个 tomcat 进程：
---

## 一些说明：

我们将会使用 qiueer 的方案来进行配置，有几个问题需要注意：

* 该项目下有 All In One 和 JVM 两个目录，注意请使用 All In One 之下的模板和 python 脚本。

![zabbix-qiueer](http://static.zhuxiaodong.net/blog/static/images/zabbix-qiueer.png)

* All In One 下除了普通 JVM 和 Tomcat 的方案之外，还包括了 Mysql Redis Nginx MongoDB MemoryCache 等方案。具体参考[read me](https://github.com/qiueer/zabbix/blob/56a0a5b7743c2684e3426a2dfcc3e6a22ea11fe8/All%20In%20One/readme.md)文档

* All In One/src/jvm.py 和 All In One/src/tomcat.py 这两个脚本，我在具体采集数据的时候，会发现以下的错误日志：

```
can not find java command, exit.
```

后面发现原因是脚本当中的 which 函数无法正确的获取到 java 的安装路径，估计与我安装的 python 版本是2.7有关系，还没有来得及仔细去分析源代码，因此采取了直接写死 java 安装路径的方案，具体方式为：

**修改 All In One/src/jvm.py 脚本**

```
class JMX(object):
    def __init__(self, logpath, debug=False):
        self._logpath = logpath
        self._debug = debug

        # 这里直接写死了java的执行路径
        self._java_path = "/opt/jdk/bin/java"

        self._cmdclient_jar = get_realpath() + "/" + "cmdline-jmxclient-0.10.3.jar"
        self._logger = slog(self._logpath, debug=debug, size=5, count=5)


    def get_item(self, beanstr, key, port, iphost='localhost', auth='-'):
        """
        java -jar cmdline-jmxclient-0.10.3.jar - localhost:12345 java.lang:type=Memory NonHeapMemoryUsage
        参数：
        """
        pre_key = key
        sub_key = None
        if "." in key:
            pos = str(key).rfind(".")
            pre_key = key[0:pos]
            sub_key = key[pos + 1:]
        
        # 下面的3行代码直接注释掉
        # if self._java_path == None:
        #    print "can not find java command, exit."
        #    return None
        
        cmdstr = "%s -jar %s %s %s:%s '%s' '%s'" % (self._java_path, self._cmdclient_jar, auth, iphost, port, beanstr, pre_key)    
```

**修改 All In One/src/tomcat.py 脚本**

```
class JTomcat(object):
    def __init__(self, logpath, debug=False):
        self._logpath = logpath
        self._debug = debug

        # 这里直接写死了java的执行路径
        self._java_path = "/opt/jdk/bin/java"

        self._cmdclient_jar = get_realpath()+"/" +"cmdline-jmxclient-0.10.3.jar"
        self._logger = slog(self._logpath, debug=debug, size=5, count=2)
```

## 具体安装步骤：

### step 1：在 zabbix web portal 上导入相关的模板

由于 tomcat 我选择的使用是 APR 的 I/O 方式，因此这里导入的是 Qiueer-Template JMX Tomcat With IO-APR.xml 和 Qiueer-Template JVM.xml

![zabbix-qiueer-template](http://static.zhuxiaodong.net/blog/static/images/zabbix-qiueer-template.png)

### step 2：调整之前创建的 Auto registration action，添加上述两个模板的link。

![zabbix-qiueer-action](http://static.zhuxiaodong.net/blog/static/images/zabbix-qiueer-action.png)

### step 3：在 zabbix agent 的机器上，下载相关的源代码和脚本：

```
cd /usr/local/src/

git clone https://github.com/qiueer/zabbix.git
```

### step 4：确定 zabbix agent 扩展功能需要包括的配置文件路径，例如 /usr/local/zabbix_agent_extend/conf  
 ，修改 zabbix agent 的主配置文件 zabbix_agentd.conf ，加入如下内容：

```
vim /etc/zabbix/zabbix_agentd.conf

# 必须允许zabbix-agent以root账号执行，才能够让python脚本执行时不会有权限问题。
AllowRoot=1

# 设置包含的配置文件路径
Include=/usr/local/zabbix_agent_extend/conf/*.conf
```

### step 5：在 root 用户下执行 zabbix_extend_init.sh 脚本，脚本第一个参数为配置文件存放目录，第二个参数为源码脚本存放路径。  
例如执行如下命令，其中 /usr/local/zabbix_agent_extend/conf 是 step 3 中确定的文件路径：

```
mkdir -p /usr/local/zabbix_agent_extend/conf/
mkdir -p /usr/local/zabbix_agent_extend/scripts/  
cd /usr/local/src/zabbix/zabbix/All\ In\ One/

bash zabbix_extend_init.sh /usr/local/zabbix_agent_extend/conf /usr/local/zabbix_agent_extend/scripts  
```

其作用是：  
* 将 confs 目录下的文件拷贝到 /usr/local/zabbix_agent_extend/conf 目录  
* 将 src 目录下的文件拷贝到 /usr/local/zabbix_agent_extend/scripts 目录

### step 6：重启zabbix-agent

```
systemctl restart zabbix-agent
```

上述步骤都完成了之后，我们就能够看到 zabbix portal 当中能够正确的获取到相关的 tomcat 数据，并且是按照 JMX 的端口号进行的分别监控。

![zabbix-qiueer-data1](http://static.zhuxiaodong.net/blog/static/images/zabbix-qiueer-data1.png)

![zabbix-qiueer-data2](http://static.zhuxiaodong.net/blog/static/images/zabbix-qiueer-data2.png)

如果无法正确的监控到数据，可以按照如下的方式测试：

* 查看 /tmp/zabbix_jvm_info.log 是否有错误日志
* 查看 /tmp/zabbix_tomcat_info.log 是否有错误日志
* 设置 /tmp/zabbix_jvm_info.log 和 /tmp/zabbix_tomcat_info.log 权限为 777

```
chmod 777 /tmp/zabbix_jvm_info.log
chmod 777 /tmp/zabbix_tomcat_info.log
```

更多信息，请参考 github 项目的 [read me](https://github.com/qiueer/zabbix/blob/56a0a5b7743c2684e3426a2dfcc3e6a22ea11fe8/All%20In%20One/readme.md)


# 未解决的问题
---

1. 该项目的 Qiueer-Template JVM.xml 模板是基本 JDK 1.7 的，因此和永久代相关的内存数据无法正确的获取，后续需要调整该模板，以支持 JDK 1.8 版本下能够获取到 Metaspace 的内存数据。

2. Qiueer-Template JMX Tomcat With IO-APR.xml 模板相关的数据未能正确的获取，还需要继续研究如何解决。
