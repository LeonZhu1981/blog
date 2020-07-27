title: maven学习笔记
date: 2017-03-13 11:17:56
categories: programming
tags: 
- maven
- java
---

## maven安装:

mac下请使用brew进行安装:

```
brew install maven
```

验证maven是否安装成功:

```
mvn -v
```

其中显示出的maven安装的根目录会对于今后修改配置非常有用:

```
Maven home: /usr/local/Cellar/maven/3.3.9/libexec
```

## maven约定的目录结构:

```
src
  --main
   --java
    --package
  --test
   --java
    --package
resouces
```

## 核心命令:

* mvn compile: 编译
* mvn test:    执行测试
* mvn clean:   清除target文件夹
* mvn install: 将编译出的jar包安装到本地maven仓库中（与将编译出的jar包放到CLASSPATH目录下的作用类似.）

## 使用阿里云的maven镜像, 提升相关资源下载速度.

修改maven根目录下的conf文件夹中的setting.xml文件，内容如下：

```
<mirrors>
	<mirror>
	  <id>alimaven</id>
	  <name>aliyun maven</name>
	  <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
	  <mirrorOf>central</mirrorOf>        
	</mirror>
</mirrors>
```
<!--more-->
## 自动创建目录结构: 带参数的方式

```
mvn archetype:generate -DgroupId=com.sotao.mavendemo04 -DartifactId=maven04-model -Dversion=1.0.0-SNAPSHOT -Dpackage=com.sotao.mavendemo04.service
```
参数说明:
* -DgroupId=组织名, 公司网址的反写+项目名
* -Dversion=版本号
* -Dpackage=代码的包名

## 自动创建目录结构: 不带参数的方式

```
mvn archetype:generate
```

## 如果想要让mvn package打包的jar包能够直接被java -jar xxxx.jar执行, 需要在pom.xml文件当中添加:

```
<build>
	<plugins>
	  <plugin>
	    <!-- Build an executable JAR -->
	    <groupId>org.apache.maven.plugins</groupId>
	    <artifactId>maven-jar-plugin</artifactId>
	    <version>3.0.2</version>
	    <configuration>
	      <archive>
	        <manifest>
	          <addClasspath>true</addClasspath>
	          <classpathPrefix>lib/</classpathPrefix>
	          <mainClass>com.sotao.mavendemo04.service.App</mainClass>
	        </manifest>
	      </archive>
	    </configuration>
	  </plugin>
	</plugins>
</build>
```

## 远程中央仓库的默认地址:

可以通过查看如下jar包当中的pom.xml文件得到:

```
/usr/local/Cellar/maven/3.3.9/libexec/lib/maven-model-builder-3.3.9.jar
```

```
<repositories>
    <repository>
      <id>central</id>
      <name>Central Repository</name>
      <url>https://repo.maven.apache.org/maven2</url>
      <layout>default</layout>
      <snapshots>
        <enabled>false</enabled>
      </snapshots>
    </repository>
</repositories>
```

## 本地默认仓库的地址:

```
cd /Users/leonzhu/.m2/
```

修改本地默认仓库的地址:

```
vim /usr/local/Cellar/maven/3.3.9/libexec/conf/settings.xml
```

修改节点:
```
<localRepository>/path/to/local/repo</localRepository>
```

## maven生命周期:

**clean:**   清理项目
* pre-clean: 执行清理之间的工作.
* clean: 清理上一次构建生成的文件.
* post clean: 执行清理后的文件.

**default:** 构建项目(最核心的周期)
* compile: 编译
* test: 运行单元测试
* package: 打包jar
* install: 安装jar至本地仓库

**site:**    生成项目站点
* pre-site: 在生成项目站点前要完成的工作.
* site: 生成项目的站点文档.
* post-site: 在生成项目站点后要完成的工作.
* site-deploy: 发布生成的站点至目标服务器.

使用相关的plugins插件, 可以在指定的目标周期, 运行对应的插件完成对应的工作:

```
<build>
	<plugins>
	  <plugin>
	    <groupId>org.apache.maven.plugins</groupId>
	    <artifactId>maven-source-plugin</artifactId>
	    <version>2.4</version>
	    <executions>
	      <execution>
	        <phase>package</phase>
	        <goals>
	          <goal>
	            jar-no-fork
	          </goal>
	        </goals>
	      </execution>
	    </executions>
	  </plugin>
	</plugins>
</build>
```

## POM.xml文件详解:

```
<project xmlns="http://maven.apache.org/POM/4.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                      http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <!--maven的版本号-->                  
  <modelVersion>4.0.0</modelVersion>
  
  <!--反写的公司网址 + 项目名-->
  <groupId>com.sotao.mavendemo</groupId>
  <!--项目名 + 模块名-->
  <artifactId>mavendemo-model</artifactId>
  <!--
  第一个数字: 大版本号
  第二个数字: 分支版本号
  第三个数字: 小版本号
  snapshot: 快照版本
  alpha: 内测版本
  beta: 公测版本
  release: 稳定版本
  GA: 正式发布版本
  -->
  <version>0.0.1SNAPSHOT</version>
  <!--打包方式
  	默认是jar包的方式
  	还可以是war zip pom(该方式的指定, 通常在聚合或者继承的方式时使用)
  -->
  <packaging></packaging>

  <name></name>
  <url></url>
  <description></description>
  <developers></developers>
  <licenses></licenses>
  <organization></organization>

  <dependencies>
  	<dependency>
  	</dependency>
  </dependencies>

</project>
```

## 依赖范围(Dependency Scope):

三种class path: 编译/运行/测试

* compile: 默认的依赖范围.在编译/运行/测试都有效.
* provided: 在测试/编译的时候有效, 运行时不包含. 比如开发、测试阶段需要使用相关的servlet api, 而运行阶段web容器已经包含了相关的servlet api.
* runtime: 在测试/运行的时候有效, 编译时不包含. 比如编译时只需要包含接口的代码, 并不需要包含其实现的代码.
* test: 在测试的时候有效, 比如单元测试的junit.
* system: 在测试/编译的时候有效, 但可移植太差, 与本机系统相关联.
* import: This scope is only supported on a dependency of type pom in the <dependencyManagement> section. It indicates the dependency to be replaced with the effective list of dependencies in the specified POM's <dependencyManagement> section. Since they are replaced, dependencies with a scope of import do not actually participate in limiting the transitivity of a dependency.

## 依赖传递:

A->B
C->A

那么C自动依赖于B: C->B

如果想要C不依赖于B, 可以使用exclusions来排除依赖：

```
<exclusions>
    <exclusion>
        
    </exclusion>
</exclusions>
```

## 依赖冲突:

依赖传递的原则:
* 短路径优先:
A->B->C->D
A->X->D

最终导入的D的相关依赖包为短路径的那个.

* 依赖路径相同的情况下, 谁先声明谁优先:
pom.xml文件中的依赖, 谁先声明就依赖对应的包.

## 聚合和继承:

* 聚合: 如果想要让多个项目批量来执行相关的maven命令(clean, install, package等), 可以使用聚合的特性. 
	* step 1: 首先将<packaging>pom</packaging> 修改为pom方式.
	* step 2: 设置<modules><module>maven项目的名字</module></modules>

* 继承: 相关的项目都一起依赖于某个库, 比如junit, 就可以使用pom继承的方式来在聚合项目当中设置.
	* step 1: 首先将<packaging>pom</packaging> 修改为pom方式.
	* step 2: 在聚合项目当中设置<dependencyManagement></dependencyManagement>, 将共同的依赖放置到该节点下.
	* step 3: 可以自定义扩展properties属性来定义一些公共的变量.
	* step 4: 设置子项目的<parent></parent>属性.

## 分析pom.xml当中的依赖关系:

```
mvn dependency:tree
```

## 相关的参考资料:

[官方文档](http://maven.apache.org/guides/index.html)
[中央仓库查询](https://mvnrepository.com/)
[imooc上不错的教学视频](http://www.imooc.com/learn/443)