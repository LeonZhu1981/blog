title: tomcat jvm调优实践
date: 2017-10-11 09:31:51
categories: programming
tags:
- java
- jvm
- tomcat
toc: true
---

# Tomcat和JVM参数调优

> 1. 注意, 所有需要源代码安装的库, 都统一放在 /usr/local/src目录
> 2. 主要参考资料:
> https://github.com/apache/cassandra/blob/trunk/conf/jvm.options
> https://github.com/judasn/Linux-Tutorial/blob/master/Tomcat-Install-And-Settings.md
> https://netkiller.github.io/journal/tomcat.html
> http://blog.chopmoon.com/favorites/231.html
> https://yq.aliyun.com/articles/112705
> http://blog.csdn.net/xyang81/article/details/51502766

<!--more-->

## 安装APR

### 安装APR和APR Util

```
cd /usr/local/src

wget http://mirrors.aliyun.com/apache/apr/apr-1.5.2.tar.gzx

tar -zxvf apr-1.5.2.tar.gz

cd apr-1.5.2/

./configure
make
make install
```

```
cd /usr/local/src

wget http://mirrors.aliyun.com/apache/apr/apr-util-1.5.4.tar.gz

tar -zxvf apr-util-1.5.4.tar.gz

cd apr-util-1.5.4/

./configure --with-apr=/usr/local/apr
make
make install
```

### 安装最新的Open SSL库:

由于大部分Linux发行版本的open ssl的版本较低, 因此需要先下载最新的版本进行安装.

```
cd /usr/local/src

wget https://www.openssl.org/source/openssl-1.0.2-latest.tar.gz

tar -zxvf openssl-1.0.2l.tar.gz

cd openssl-1.0.2l
#必须要设置-fPIC编译参数, 否则编译tomcat-native时会报错.
./config --prefix=/usr/local/openssl -fPIC  
make
make install
```

替换掉本地已安装open ssl版本

```
mv /usr/bin/openssl /usr/bin/openssl.bak
mv /usr/include/openssl /usr/include/openssl.bak
ln -s /usr/local/ssl/bin/openssl /usr/bin/openssl
ln -s /usr/local/ssl/include/openssl /usr/include/openssl
echo “/usr/local/ssl/lib” >> /etc/ld.so.conf
ldconfig -v
openssl version -a
```

### 安装tomcat-native库

tomcat的安装包当中已经包含了tomcat-native的源代码，具体位置在tomcat的bin目录下: tomcat-native.tar.gz

```
cp /opt/apache-tomcat-8.5.14/bin/tomcat-native.tar.gz /usr/local/src/

cd /usr/local/src/
tar -zxvf tomcat-native.tar.gz
cd /usr/local/src/tomcat-native/native/

./configure --with-apr=/usr/local/apr --with-java-home=/opt/jdk/
make
make install
```

## 配置Tomcat

### JVM相关配置
vim bin/catalina.sh 

```
# JVM performance
JAVA_OPTS="$JAVA_OPTS -XX:-UseBiasedLocking -XX:AutoBoxCacheMax=20000 -Djava.security.egd=file:/dev/./urandom”

# JVM GC
JAVA_OPTS="$JAVA_OPTS -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=75 -XX:+UseCMSInitiatingOccupancyOnly -XX:MaxTenuringThreshold=2 -XX:+ExplicitGCInvokesConcurrent -XX:+ParallelRefProcEnabled”

# JVM Memory Setting
JAVA_OPTS="$JAVA_OPTS -Xms2048M -Xmx2048M -Xmn1024M -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=512m"

# JVM GC Log/Log/Monitor Setting
JAVA_OPTS="$JAVA_OPTS -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:$CATALINA_BASE/logs/debug/heap_trace.txt -XX:+PrintGCApplicationStoppedTime -XX:+PrintCommandLineFlags -XX:-OmitStackTraceInFastThrow -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=$CATALINA_BASE/logs/debug/dump -XX:ErrorFile=$CATALINA_BASE/logs/debug/error/hs_err_%p.log"

# APR I/O Mode Setting
JAVA_OPTS="$JAVA_OPTS -Djava.library.path=/usr/local/apr/lib"

# JVM JMX Setting 
CATALINA_OPTS="$CATALINA_OPTS -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.port=8040 -Djava.rmi.server.hostname=10.9.1.177"
```

> * 注意JMX相关配置中, 需要将Djava.rmi.server.hostname设置为zabbix监控的hostname
> * JMX监控配置好了之后, 可以使用cmdline-jmxclient-0.10.3.jar这个客户端工具来测试是否能从JMX获取到数据.
> 
> ```
> java -jar cmdline-jmxclient-0.10.3.jar controlRole:tomcat localhost:9000 java.lang:type=Memory NonHeapMemoryUsage 
> ```
> * JMX相关的参数必须放在CATALINA_OPTS下， 而非JAVA_OPTS，否则会导致关闭tomcat时，出现jms端口号被占用的异常.
 


### Tomcat server.xml

vim conf/server.xml

* 找到Executor节点, 添加如下的内容:

```
<Executor name="tomcatThreadPool"
          namePrefix="catalina-hms-8000-exec-"
          maxThreads="500"
          minSpareThreads="30"
          maxSpareThreads="75"
          maxIdleTime="60000"
          prestartminSpareThreads = "true"
          maxQueueSize = "100"/>
```
> 注意namePrefix请设置为能够标识当前应用程序的名字.

* 找到Connector节点, 添加如下的内容:

```
<Connector 
   executor="tomcatThreadPool"
   port="8000" 
   protocol="org.apache.coyote.http11.Http11AprProtocol" 
   connectionTimeout="20000" 
   maxConnections="10000" 
   enableLookups="false" 
   acceptCount="300" 
   maxPostSize="10485760" 
   maxHttpHeaderSize="8192" 
   compression="on" 
   disableUploadTimeout="false" 
   compressionMinSize="2048" 
   acceptorThreadCount="2" 
   compressableMimeType="text/html,text/xml,text/plain,text/css,text/javascript,application/javascript" 
   URIEncoding="UTF-8"/>
```

* 由于我们不会AJP的方式, 因此需要注释掉默认被打开AJP的配置:

```
<!--<Connector port="8040" protocol="AJP/1.3" redirectPort="8443" />-->
```

### Tomcat context.xml

为了解决tomcat启动时, 如下的缓存不足的警告信息, 需要做如下的设置:
org.apache.catalina.webresources.Cache.getResource Unable to add the resource at o the cache for web application [] because there was insufficient free space available after evicting expired cache entries - consider increasing the maximum size of the cache

设置方式:

vim conf/context.xml
添加 Resouces 节点

```
<Resources cachingAllowed="true"
           cacheMaxSize="102400"
           cacheObjectMaxSize="2048"/>
```

参考: 
https://stackoverflow.com/questions/26893297/tomcat-8-throwing-org-apache-catalina-webresources-cache-getresource-unable-to

https://github.com/jhipster/generator-jhipster/issues/3995