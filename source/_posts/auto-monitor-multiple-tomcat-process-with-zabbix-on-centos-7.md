title: Auto monitor multiple tomcat process with zabbix on centos 7
date: 2017-09-04 15:58:37
categories: programming
tags:
- zabbix
- devops
- low level discovery
- JMX
- jconsole
---

# 目的
---

为了继续解决[上篇文章](http://www.zhuxiaodong.net/2017/how-to-auto-monitor-jvm-tomcat-process-with-zabbix-on-centos-7/#未解决的问题)提到的暂未解决的2个问题，以及研究如何在 zabbix 当中扩展相关的 JMX 监控 Item ，因此编写了该篇 blog 。

<!--more-->

# 使用 jconsole 查看 JMX 的 Mbean
---

由于我们需要在 JMX 查看到 JDK 1.8 当中 Metaspace 的内存情况，同时，如果今后有新的基于 JMX zabbix 监控项需要添加的话，我们需要知道该监控项对应 Mbean 的 ObjectName 以及 Attributes 是什么？

## How to?

step 1： Tomcat （普通 JVM 进程配置方式类似） 进程 JMX 配置

[vim CATALINA_HOME/bin/catalina.sh]

```
CATALINA_OPTS="$CATALINA_OPTS -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.port={JMX的端口}
-Djava.rmi.server.hostname={本机的IP地址}"
```

step 2： 打开 jconsole ，输入远程连接 JMX 机器的 IP 和 Port

![jconsole-1](https://www.zhuxiaodong.net/static/images/jconsole-1.png)

step 3： 连接了之后，选择 Mbean tab 页，我们能看到左边一颗树型菜单中列出了当前 JMX 中所有的 Mbean 对象。

![jconsole-2](https://www.zhuxiaodong.net/static/images/jconsole-2.png)

我们关注的是 Mbean 当中的 ObjectName ，例如上述图中的: java.lang:type=Threading

step 4： 展开 Attributes 之后，我们能够看到该类别下具体属性和对应的监控数据。

![jconsole-3](https://www.zhuxiaodong.net/static/images/jconsole-3.png)

例如上述图中的: java.lang:type=Threading 下的 ThreadCount attribute

## JMX Mbean 与 Zabbix Item Key 之间的对应关系

我们以 Qiueer-Template JVM Generic Basic.xml 模板中的 "[$3] mem Heap Memory used" Item prototypes 为例

![zabbix-item-prototypes-1](https://www.zhuxiaodong.net/static/images/zabbix-item-prototypes-1.png)

可以看到，其实最关键的部分就是 Key ：

jmx.jvm.item["java.lang:type=Memory",HeapMemoryUsage.used,{JVMPORT}]

其核心的构造规则为：

```
jmx.jvm.item["{ObjectName}",{AttributeName}.{ItemName},{JVMPORT}]
```

* {ObjectName} ： 与我们在 jconsole 当中看到的 Mbean ObjectName一致。

![jconsole-4](https://www.zhuxiaodong.net/static/images/jconsole-4.png)


* {AttributeName} ：即 MbeanAttributeInfo 的 Name

![jconsole-5](https://www.zhuxiaodong.net/static/images/jconsole-5.png)

* {ItemName} ： 
某些 MbeanAttributeInfo 的类型是 javax.management.openmbean.CompositeData 复杂结构（我们可以理解为数组），这种情况下，需要再指定 {ItemName} 才能够获取到监控数据。例如下图当中，我们需要获取到 Metaspace 的 Usage 下的 committed 数据，committed 即是 {ItemName}

![jconsole-6](https://www.zhuxiaodong.net/static/images/jconsole-6.png)


* {JVMPORT} ：这个参数变量其实与 JMX Mbean 无关，只是为了在 Zabbix 当中监控一台机器上的多个 JVM 进程时，通过 JVM PORT 来进行对 Zabbix Item Key进行区分。（Zabbix 当中要求每一个 Item 的 Key 不能重复）

## 如何测试数据能否正确地获取？

我们可以使用 cmdline-jmxclient-0.10.3.jar 来进行测试，如果对这个 jar 包的源代码感兴趣，可以参考[这里](https://sourceforge.net/p/jmxcmd/code/HEAD/tree/jmxcmd/src/de/layereight/jmxcmd/Client.java#l165)。

测试方法：

```
java -jar cmdline-jmxclient-0.10.3.jar - {jmx_ip_address}:{jmx_port} {ObjectName} {AttributeName}
```

例如：

```
java -jar cmdline-jmxclient-0.10.3.jar - lab-web01:8040 java.lang:type=MemoryPool,name=Metaspace Usage

# output
09/04/2017 16:47:33 +0800 org.archive.jmx.Client Usage:
committed: 73531392
init: 0
max: 536870912
used: 71020352
```

PS：这个代码貌似是上古时期的了，05年之后就没有再更新过。笔者在测试的过程还发现，通过该工具无法获取某些tomcat的数据（ jconsole 当中能够正确地获取），例如:

```
java -jar cmdline-jmxclient-0.10.3.jar - lab-web01:8040 Catalina:type=GlobalRequestProcessor,name=http-apr-8000 bytesSent

# output
09/04/2017 17:09:14 +0800 Client Catalina:name=http-apr-8000,type=GlobalRequestProcessor is not a registered bean
```

如果调整为 name=* ，就能够正确地获取到数据：

```
java -jar cmdline-jmxclient-0.10.3.jar - lab-web01:8040 Catalina:type=GlobalRequestProcessor,name=* bytesSent

# output
09/04/2017 17:11:45 +0800 Client bytesSent: 0
```

# Issue 1
---

调整 Qiueer-Template JVM.xml 模板，以支持 JDK 1.8 版本下能够获取到 Metaspace 的内存数据。

我们需要监控的 Metaspace 内存数据包括:

* java.lang:type=MemoryPool,name=Metaspace,Usage.used
* java.lang:type=MemoryPool,name=Metaspace,Usage.max
* java.lang:type=MemoryPool,name=Metaspace,Usage.committed

step 1：
在 Qiueer-Template JVM Generic Basic 模板的 Item prototypes 下，选择一个合适的 Item prototype，clone 出一个新的 Item prototype。这里我们选择的是对 [{JVMPORT}] mp Code Cache used 进行 clone 操作。

![zabbix-item-prototypes-clone](https://www.zhuxiaodong.net/static/images/zabbix-item-prototypes-clone.png)

step 2：
在 clone 出的新 Item 中，修改 Key 值为：
jmx.jvm.item["java.lang:type=MemoryPool,name=Metaspace",Usage.used,{JVMPORT}] ；修改 Name 值为：[$3] mp Metaspace used

![zabbix-item-prototypes-2](https://www.zhuxiaodong.net/static/images/zabbix-item-prototypes-2.png)

step 3：
重复步骤1和2，添加 Usage.max ， Usage.committed 的数据。

完成了之后，我们可以在 Monitor --> Latest data 当中获取到相关的 Metaspace 数据。

![zabbix-latest-data-1](https://www.zhuxiaodong.net/static/images/zabbix-latest-data-1.png)

# Issue 2
---

解决 Qiueer-Template JMX Tomcat With IO-APR.xml 模板，相关的数据未能正确获取的问题。

## 问题描述

之前我在导入了 Qiueer-Template JMX Tomcat With IO-APR.xml 模板导入 zabbix 了之后，发现相关的 tomcat 监控数据并没有产生。偶然发现模板当中的 Item prototypes 的 Key 设置中有一个参数叫： {BIZPORT} ，依稀记得在执行 tomcat.py 脚本时，并没有列出这个参数，验证了一下，果然没有。

```
cd /usr/local/zabbix_agent_extend/scripts/

python tomcat.py --list

# output
{
       "data":[
              {
                     "{#CTLN_HOME}":"/opt/tomcat-8000",
                     "{#DIRNAME}":"tomcat-8000",
                     "{#JVMPORT}":8040,
                     "{#PID}":1012,
                     "{#RUNUSER}":"root"
              },
              {
                     "{#CTLN_HOME}":"/opt/tomcat-8001",
                     "{#DIRNAME}":"tomcat-8001",
                     "{#JVMPORT}":8041,
                     "{#PID}":1084,
                     "{#RUNUSER}":"root"
              },
              {
                     "{#CTLN_HOME}":"/opt/tomcat-8002",
                     "{#DIRNAME}":"tomcat-8002",
                     "{#JVMPORT}":8042,
                     "{#PID}":1160,
                     "{#RUNUSER}":"root"
              }
       ]
}
```

嗯嗯， {BIZPORT} 参数未能正确地获取到。

分析了一下 tomcat.py 脚本的代码：

```
class TCSerHandler(xml.sax.ContentHandler):
    def __init__(self):
        self._connectors = list()
    
    def startElement(self, tag, attributes):
        if tag == "Connector":
            protocal = attributes["protocol"]
            confitem = dict(attributes)
            self._connectors.append(confitem)
            
    def get_connectors(self):
        return self._connectors
    
    def get_biz_port(self):
        cons = self.get_connectors()
        for item in cons:
            protocol = item.get("protocol", None)

            # 问题出在这里
            if protocol == "HTTP/1.1":
                biz_port = item.get("port", None)
                return int(biz_port)
        return None
```

由于我使用的 APR 的 I/O 方式，因此在 tomcat 的 server.xml 中显示将 protocol 节点指定为了 org.apache.coyote.http11.Http11AprProtocol ，导致无法正确地获取到 tomcat 端口号。

解决方案：去掉了上面的 if 判断逻辑，重启 zabbix-agent 之后，数据获取正常。


# Issue 3
---

在第二个问题解决了之后，笔者发现 Qiueer-Template JMX Tomcat With IO-APR.xml 模板中，部分和 session 相关的数据还是无法正常获取。 Item Prototypes 包括：

* [{JVMPORT}][{BIZPORT}] sessionCounter
* [{JVMPORT}][{BIZPORT}] rejectedSessions
* [{JVMPORT}][{BIZPORT}] maxActiveSessions
* [{JVMPORT}][{BIZPORT}] maxActive
* [{JVMPORT}][{BIZPORT}] activeSessions

它们共同的问题在于，对应的 Key 的 ObjectName 都设置为了：

```
Catalina:type=Manager,path=/,host=localhost
```

这与使用 jconsole 观察到的 ObjectName 不一致：

```
Catalina:type=Manager,host=localhost,context=/
```

修改了上述 Item Prototypes Key 之后，重启 zabbix-agent ，数据就能够正常获取到了。

通过 Host -> Item 查看 Application 为 Tomcat 和 Sessions 类别中，只有很少量的 Item 显示为 “Not supported”

![zabbix-host-item-filter-1](https://www.zhuxiaodong.net/static/images/zabbix-host-item-filter-1.png)

在 Monitor -> Latest data 中，Tomcat 相关的 Application 也能够正确地获取到数据

![zabbix-lastest-data-tomcat](https://www.zhuxiaodong.net/static/images/zabbix-lastest-data-tomcat.png)