title: 基于阿里云上实现全站https的正确姿势(二)
date: 2016-11-24 22:40:51
categories: programming
tags:
- linux
- ssl
- https
- 阿里云
- SLB
- nginx
- http/2
---

[上一篇文章](http://www.zhuxiaodong.net/2016/how-to-switch-your-website-to-https-on-aliyun-part1/)当中, 我们介绍了SSL证书的类型, 如何选择和购买等. 本篇文章我们开始进入实践环节, 讨论如何部署SSL证书.

# 我们应用程序的整体架构:
---
![arch](https://www.zhuxiaodong.net/static/images/zhuanzhu-arch.png)

> 上述架构只列出了和讨论https相关的部分. 

# SSL证书放在哪里进行管理?
---
对于静态资源请求, 由于存储在阿里云OSS上, 并在最前端使用了阿里云CDN, 因此将SSL证书的相关信息在阿里云CDN上配置一下就好了.

对于动态资源的请求, 最前端是使用SLB, 理论上来说, 处理方式可以和上述CDN的方式保持一致, 即将SSL证书放在SLB上进行管理,  但是这样做有以下的缺点:

* 我们需要支持将http请求强制跳转至https, 毕竟用户在浏览器当中输入地址时, 几乎没有用户会主动在浏览器地址栏当中输入: https://. 目前阿里云的SLB并不支持redirect强制跳转的功能,因此我们需要在SLB背后在架设一个nginx, 通过配置server listen 80来做强制跳转.

```
server {
    listen          80;
    server_name     .example.com;
    return 301      https://$host$request_uri;
}
```

* SLB无法做更多基于https安全以及性能层面的优化. 例如支持[HSTS](https://imququ.com/post/sth-about-switch-to-https.html#toc-2), [Content Security Policy](https://imququ.com/post/content-security-policy-level-2.html), [OCSP Stapling](https://raymii.org/s/tutorials/OCSP_Stapling_on_nginx.html)

同时, nginx还可以很方便做很多扩展功能, 例如http/2的支持, TCP优化等.

基于上述原因, 我们决定在SLB背后的web server上部署nginx, 并将SSL证书放在nginx上进行管理. 

调整之后的架构如下:
![arch2](https://www.zhuxiaodong.net/static/images/zhuanzhu-arch2.png)

> Q: 上述架构中nginx是否会有单点失败的问题?
> A: 不会, 基于成本的考虑, 我们并没有在使用单独的ECS Server部署nginx, 而是将nginx直接部署到了原来的那2台web server上, SLB上的健康检查也调整为check对应的nginx端口是否alive. 具体的设置方式会在下文当中进行介绍.

<!--more-->

# 实践环节
---
接下来我们会阿里云CDN配置, nginx编译/部署/配置, 阿里云SLB配置, 以及相关的应用程序改动等方面, 逐一讲解完整的https部署过程.

## Step 1: 从阿里云portal上下载SSL证书.
![download-ssl](https://www.zhuxiaodong.net/static/images/downloadssl.png)

这里下载的是for nginx格式的证书. 原因是由于在阿里云CDN设置证书时需要使用, 此外我们会使用nginx来管理SSL证书.

下载解压了之后, 会得到2个文件:
xxx.pem
xxx.key

关于证书的编码格式和不同的扩展名, 请参考[这里](http://www.cnblogs.com/guogangj/p/4118605.html)

## Step 2: 在阿里云CDN上设置开启https支持.
* 在阿里云CDN的基本信息中, 找到https安全加速, 点击编辑按钮.
![aliyun-cdn-config1](https://www.zhuxiaodong.net/static/images/aliyun-cdn-config1.png)

* 使用文本编辑器打开之前下载xxx.pem文件, 将文件的内容copy到证书内容文本框内.(**注意去掉-----END CERTIFICATE-----与-----BEGIN CERTIFICATE-----之间的换行符**)

* 使用文本编辑器打开之前下载xxx.key文件, 将文件的内容copy到私钥文本框内
![aliyun-cdn-config2](https://www.zhuxiaodong.net/static/images/aliyun-cdn-config2.png)

* 建议跳转类型设置为默认, 这样保证在程序完全切换完成之前兼容性.

* 保存了之后, 使用https去访问任意的一个静态资源, 测试看是否正常.

## Step 3: 源代码编译部署nginx.
内容较多, 参考[这里](http://www.zhuxiaodong.net/2016/configure-nginx-server-support-https-http2-on-centos7/)

## Step 4: 调整阿里云SLB的监听设置.
上一个步骤中, 我们已经部署好了nginx, 接下来需要调整SLB, 将原有SLB的监听端口由访问后端web application调整为访问nginx.

在阿里云后台portal中, 添加SLB监听设置: SLB TCP:443 --> Nginx TCP:443

![aliyun-slb-step1.png](https://www.zhuxiaodong.net/static/images/aliyun-slb-step1.png)

设置SLB的健康检查:
![aliyun-slb-step2.png](https://www.zhuxiaodong.net/static/images/aliyun-slb-step2.png)

> NOTE: 
> 1. 原有SLB http:80 --> backend server http:port 的监听在完全切换为https之前建议保留, 以确保应用程序的兼容性.
> 2. 完整切换完成之后, 将SLB http:80的监听调整为: SLB TCP:80 --> backend server tcp:443(即nginx的端口)

## Step 5: 修改代码中内部以及第三方访问资源.
正如前文提到的, 由于浏览器的[Mixed Content](https://imququ.com/post/sth-about-switch-to-https.html#toc-0)规范, 我们需要将application当中的使用到的内部访问URL以及第三方组件, 调整为使用https协议进行访问.

在升级的过程当中, 我们难免会出现遗漏的地方, 有两种方式来解决这个问题:

1. 加载资源时, 去掉显式指定访问协议的部分, 这样浏览器就能根据主页面协议类型来自动选择HTTP/HTTPS资源 例如:
```
<img src="http://static.example.com/banner.jpg" />

# 调整为:

<img src="//static.example.com/banner.jpg" />
```

2. 针对于现代浏览器, 可以通过upgrade-insecure-requests这个CSP 指令, 强制让页面上所有的http访问转换为https访问, 设置方式参考[这里](http://www.cnblogs.com/hustskyking/p/upgrade-insecure-requests.html)


下面我们列出一些在升级第三方组件的过程当中, 踩过的一些“坑”:

### 百度地图javascript api.

按照百度地图javascript api的[官方文档](http://lbsyun.baidu.com/index.php?title=jspopular), 只有javascript V2.0版本才支持https, 参考下图:

![baidumap-api-summary](https://www.zhuxiaodong.net/static/images/baidumap-api-summary.png)

对于javascript API, 比较关键的是添加query string: **s=1**, 参考[这里](http://lbsyun.baidu.com/index.php?title=jspopular/guide/introduction#Https_.E8.AF.B4.E6.98.8E)

```
https://api.map.baidu.com/api?v=2.0&ak=你的密钥&s=1；
```

我们在将原有程序使用V1.4版本升级到V2.0的过程当中, 还发现了api的不兼容问题, 最后调整了代码才解决这个问题:

```
var local = new BMap.LocalSearch(map, {
    onSearchComplete: function (rs) {
        searchResult = rs;
    }
});

function bindTraffic() {
    var html = [];
    var str = buildTemplate(searchResult[0]);
    if (str != "") {
        // your code logic here
    }
}

function buildTemplate(ud, tmpl) {
    var context = {
        title: ud.keyword,
        items: []
    };

    $.each(ud, function (index1, u) {
    	//u对象下V1.4版本的属性名叫ki, V2.0版本调整为了wr
        $.each(u.wr, function (index2, ki) {
            // your code logic here
        })
    });
}
```

### CNZZ (用于网站的数据收集统计,类似于GA或百度统计) 
不需要升级, 官方的js脚本已经考虑了https的使用场景.

### OneAPM (用于网站性能监控和统计)
不需要升级, 官方的js脚本已经考虑了https的使用场景.

### 百度站长统计
js代码需要调整为:

```
<script type="text/javascript">
    (function(){
        var bp = document.createElement('script');
        bp.src = '//push.zhanzhang.baidu.com/push.js';
        var curProtocol = window.location.protocol.split(':')[0];
        if (curProtocol === 'https') {
            bp.src = 'https://zz.bdstatic.com/linksubmit/push.js';
        }
        else {
            bp.src = 'http://push.zhanzhang.baidu.com/push.js';
        }
        var s = document.getElementsByTagName("script")[0];
        s.parentNode.insertBefore(bp, s);
    })();
</script>
```

# 对网站进行https测试.
---

## 使用SSL Lab's进行评测

访问https://www.ssllabs.com/ssltest/index.html, 以下是我们按照上述方式调整之后的得分:
![ssllab-score](https://www.zhuxiaodong.net/static/images/ssllab-score.png)

## 使用HTTP Security Report进行评测
访问https://httpsecurityreport.com/, 以下是我们按照上述方式调整之后的得分:
![http-security-report](https://www.zhuxiaodong.net/static/images/http-security-report.png)

得分并不高, 剩下的几个点需要参考[HTTP Security Best Practice](https://httpsecurityreport.com/best_practice.html)继续优化.