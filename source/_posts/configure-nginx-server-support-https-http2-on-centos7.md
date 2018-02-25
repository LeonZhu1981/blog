title: nginx编译-安装-配置-优化实践总结.
date: 2016-11-25 11:47:23
categories: programming
tags:
- linux
- ssl
- https
- 阿里云
- SLB
- nginx
- http/2
- centos7
---
# 一些更新说明:
---
**2016-12-01:** 
1. 更新了使用nginx_upstream_check_module时, 当backend web server为IIS时, 原有的配置会导致upstream check module无法判断后端server是否alive的问题.
2. nginx configure添加了http_stub_status_module, 用于监控基本的访问量信息.
3. 添加了重新编译nginx之后, 替换原有部署的可执行文件的具体操作步骤.

# 环境介绍:
---
* OS: 阿里云/CentOS 7/3.10.0-123.9.3.el7.x86_64.
* 硬件配置: 2台ECS(web01, web02), 4 core, 16GB memeory, 100GB disk size.
* 原有程序的部署介绍(基于node.js)
	* port 9000: 部署了PC网站, 域名为: *.example.com, www.example.com
	* port 9001: 部署了m站点, 域名为: m.example.com
	* port 9002: 部署了app web api, 域名为: api.example.com


# 总体思路:
---
1. nginx部署在2台ECS server(web01, web02)的443端口.
2. 利用nginx的反向代理, 将请求按照domain name将请求分别转发至对应的node.js application port.
3. nginx upstream配置健康检查, 监测web01和web02上的node.js application是否alive.


# 编译安装nginx:
---

## 重要说明:
nginx以及所有的第三方包的源代码都统一放到/usr/local/src目录.目录结构如下:

```
└── src
    ├── ct-submit-1.0.0
    ├── libbrotli
    ├── nginx-1.11.5
    ├── nginx-ct-1.3.1
    ├── nginx_upstream_check_module
    ├── ngx_brotli
    ├── openssl
    └── sslconfig
```

如果你之前已经安装过nginx的基础上, 需要更新patch或者添加第三方module, 请在make完成了之后**不要执行make install**. 正确的处理方式为: 

* 先将原有的nginx二进制文件备份, 并备份其所有的conf文件. 完成了备份之后, 在将make生成的二进制文件手动copy到/usr/local/nginx/sbin/目录下:

```
sudo cp /usr/local/src/nginx-1.11.5/objs/nginx /usr/local/nginx/sbin/
```

* copy完成了之后, 可能需要调整conf的相关配置, 请记得一定要使用nginx -t测试配置文件的正确性

```
sudo /usr/local/nginx/sbin/nginx -t
```

* 测试成功了之后, 请执行: 

```
sudo /usr/local/nginx/sbin/nginx -s reload
```
<!--more-->

## 安装依赖库:
```
sudo yum install build-essential libpcre3 libpcre3-dev zlib1g-dev unzip prce prce-dev pcre pcre-devel libtool zlib zlib-devel git
```
> 安装git为了从去clone github上相关的nginx插件.

## 获取nginx所依赖的第三方Module:

### nginx-ct: 

用于启用[Certificate Transparency](https://imququ.com/post/certificate-transparency.html)功能.

```
cd /usr/local/src/
sudo wget -O nginx-ct.zip -c https://github.com/grahamedgecombe/nginx-ct/archive/v1.3.1.zip
uzip nginx-ct.zip
```

### ngx_brotli: 
支持google开发的[brotli](https://github.com/google/brotli)压缩算法. 也可以参考[这里](http://www.infoq.com/cn/news/2015/10/Google-Brotli-Zotfli).

让nginx支持brotil, 我们首先需要安装libbrotli:

```
cd /usr/local/src/

git clone https://github.com/bagder/libbrotli
cd libbrotli\

./autogen.sh
./configure
make
make install
```

libbrotli默认安装在/usr/local/lib/libbrotlienc.so.1 目录, 这会导致nginx在启动时报出以下错误:

> nginx: error while loading shared libraries: libbrotlienc.so.1: cannot open shared object file: No such file or directory

解决方案: 建立libbrotlienc.so.1的软链接至/usr/lib/目录

```
# 64 位系统
sudo ln -s /usr/local/lib/libbrotlienc.so.1 /lib64

# 32 位系统
sudo ln -s /usr/local/lib/libbrotlienc.so.1 /lib
```

ref: [启用 Brotli 压缩算法，减少流量](https://wangqiliang.com/qi-yong-brotli-ya-suo-suan-fa-ti-gao-xing-neng/)

获取ngx_brotli源代码:
```
git clone https://github.com/google/ngx_brotli.git
```

### CloudFlare OpenSSL patch:
该库主要包含2部分, 一个部分是[CloudFlare](https://zh.wikipedia.org/wiki/CloudFlare)的nginx [ssl_ciphers](https://github.com/cloudflare/sslconfig/blob/master/conf)配置. 另外一部分是其开发的OpenSSL ChaCha20/Poly1305 patch以及Dynamic TLS Records for Nginx patch.

获取CloudFlare OpenSSL patch:
```
git clone https://github.com/cloudflare/sslconfig.git
```

### OpenSSL:
使用手动编译OpenSSL的方式, 能够更加灵活地控制, 并规避系统yum install OpenSSL版本不够新的问题.

这里采用的是OpenSSL 1.0.2j:

```
cd /usr/local/src/

wget -O openssl.tar.gz -c https://github.com/openssl/openssl/archive/OpenSSL_1_0_2j.tar.gz
tar zxf openssl.tar.gz
mv openssl-OpenSSL_1_0_2j/ openssl

# 接下来为OpenSSL打上CloudFlare的patch
cd openssl/

patch -p1 < ../sslconfig/patches/openssl__chacha20_poly1305_draft_and_rfc_ossl102j.patch 
```

### nginx_upstream_check_module:
nginx_upstream_check_module是专门提供负载均衡器内节点的健康检查的外部模块, 由淘宝的姚伟斌大神开发, 通过它可以用来检测后端server的健康状态. 如果后端 server不可用, 则后面的请求就不会转发到该节点上. 淘宝自己的tengine上是自带了该模块. 
github: https://github.com/yaoweibin/nginx_upstream_check_module.

```
cd /usr/local/src/

git clone https://github.com/yaoweibin/nginx_upstream_check_module.git
```
下载完成了之后, 在后面编译nginx的时候需要使用到.


## 编译并安装nginx:

获取nginx的源代码, 版本采用1.11.5, 并打上CloudFlare Dynamic TLS Records for Nginx patch.

```
cd /usr/local/src/

wget -c https://nginx.org/download/nginx-1.11.5.tar.gz
tar zxf nginx-1.11.5.tar.gz

cd nginx-1.11.5/
patch -p1 < ../sslconfig/patches/nginx__1.11.5_dynamic_tls_records.patch
```

```
cd /usr/local/src/nginx-1.11.5/

# 首先对nginx打上nginx_upstream_check_module的patch
patch -p1 < ../nginx_upstream_check_module/check_1.11.5+.patch

./configure --add-module=../ngx_brotli --add-module=../nginx-ct-1.3.1 --add-module=../nginx_upstream_check_module --with-openssl=../openssl --with-http_v2_module --with-http_ssl_module --with-http_gzip_static_module --with-http_stub_status_module --with-cc-opt='-Wno-deprecated-declarations -DTCP_FASTOPEN=23'

make
sudo make install
```
上述编译安装完成了之后, 默认情况下会把nginx安装到/usr/local/nginx目录.

**发现的一些坑:**
如果不添加--with-cc-opt='-Wno-deprecated-declarations'编译参数, 会出现以下的问题, 最后参考[https://github.com/google/ngx_brotli/issues/39](https://github.com/google/ngx_brotli/issues/39)进行了解决.

```
-o objs/addon/src/ngx_http_brotli_filter_module.o \
                ../ngx_brotli/src/ngx_http_brotli_filter_module.c
../ngx_brotli/src/ngx_http_brotli_filter_module.c: In function ‘ngx_http_brotli_body_filter’:
../ngx_brotli/src/ngx_http_brotli_filter_module.c:272:9: error: ‘BrotliEncoderInputBlockSize’ is deprecated (declared at /usr/local/include/brotli/encode.h:87) [-Werror=deprecated-declarations]
         ctx->brotli_ring = BrotliEncoderInputBlockSize(ctx->encoder);
         ^
../ngx_brotli/src/ngx_http_brotli_filter_module.c: In function ‘ngx_http_brotli_filter_add_data’:
../ngx_brotli/src/ngx_http_brotli_filter_module.c:498:5: error: ‘BrotliEncoderCopyInputToRingBuffer’ is deprecated (declared at /usr/local/include/brotli/encode.h:95) [-Werror=deprecated-declarations]
     BrotliEncoderCopyInputToRingBuffer(ctx->encoder, size, b->pos);
     ^
../ngx_brotli/src/ngx_http_brotli_filter_module.c: In function ‘ngx_http_brotli_filter_process’:
../ngx_brotli/src/ngx_http_brotli_filter_module.c:534:5: error: ‘BrotliEncoderWriteData’ is deprecated (declared at /usr/local/include/brotli/encode.h:109) [-Werror=deprecated-declarations]
     if (!BrotliEncoderWriteData(ctx->encoder, ctx->last, ctx->flush, &size,
     ^
../ngx_brotli/src/ngx_http_brotli_filter_module.c: At top level:
cc1: error: unrecognized command line option "-Wno-c++11-extensions" [-Werror]
cc1: all warnings being treated as errors
make[1]: *** [objs/addon/src/ngx_http_brotli_filter_module.o] Error 1
make[1]: Leaving directory `/svr-setup/nginx-1.11.5'
make: *** [build] Error 2
```

想要启用[TCP Fast Open](http://www.pagefault.info/?p=282), 但在启动nginx时却遇到了以下的错误:

```
nginx: [emerg] invalid parameter "fastopen=3" in ***
```

根据[这篇文章](https://zhangge.net/5098.html)的介绍找到了解决方案, 增加--with-cc-opt='-DTCP_FASTOPEN=23'的编译参数.

## 通过Systemd管理nginx:
centos 7及以上的版本是用[Systemd](http://www.ibm.com/developerworks/cn/linux/1407_liuming_init3/)进行系统初始化的, 它主要的设计目标是克服sysvinit固有的缺点, 提高系统的启动速度.

Systemd服务文件以.service结尾, 比如现在要建立nginx为开机启动, 如果用yum install命令安装的, yum命令会自动创建nginx.service文件, 直接用命令:

```
systemctl enable nginx.service
```

由于我们之前是使用源代码进行的安装, 因此需要手动创建nginx.service服务文件.

首先在系统服务目录里创建nginx.service文件:

```
vim /lib/systemd/system/nginx.service
```

然后copy以下的内容至nginx.service文件中:
```
[Unit] 
Description=nginx 
After=network.target 
 
[Service] 
Type=forking 
ExecStart=/usr/local/nginx/sbin/nginx 
ExecReload=/usr/local/nginx/sbin/nginx -s reload 
ExecStop=/usr/local/nginx/sbin/nginx -s quit
PrivateTmp=true 
 
[Install] 
WantedBy=multi-user.target
```

常用命令:
```
# 启动nginx
systemctl start nginx.service

# 设置开机自启动
systemctl enable nginx.service

# 查看服务当前状态
systemctl status nginx.service

# 重新启动服务
systemctl restart nginx.service

# reload服务
systemctl reload nginx.service

# 查看已经启动的服务
systemctl list-units --type=service
```

更多内容, 请参考[这里](http://www.dohooe.com/2016/03/03/352.html)

## 配置nginx:
完成上述nginx的安装之后, /usr/local/nginx下的目录结构为:

```
├── conf
│   ├── fastcgi.conf
│   ├── fastcgi.conf.default
│   ├── fastcgi_params
│   ├── fastcgi_params.default
│   ├── koi-utf
│   ├── koi-win
│   ├── mime.types
│   ├── mime.types.default
│   ├── nginx.conf
│   ├── nginx.conf.default
│   ├── scgi_params
│   ├── scgi_params.default
│   ├── uwsgi_params
│   ├── uwsgi_params.default
│   └── win-utf
├── html
│   ├── 50x.html
│   └── index.html
├── logs
│   ├── access.log
│   ├── error.log
│   └── nginx.pid
├── sbin
│   └── nginx
```

### SSL证书管理以及配置文件的规划设计:

* nginx.conf作为主配置文件, 用于配置所有公共配置, 包括ssl证书的配置, ssl相关的优化配置, 安全以及部分性能优化的配置.
* 将PC网站/m站点/App web api剥离成3个单独的配置文件, 统一放置conf.d目录下.

### 配置前的准备工作.

创建ssl证书的存放目录. 创建完成了之后, 将你的ssl证书的*.pem文件和*.key copy至此目录下. 以下我们假设SSL证书的文件名为: aliyun_example_com.pem和aliyun_example_com.key

```
sudo mkdir /usr/local/nginx/ssl
```

部署[Diffie-Hellman](http://baike.baidu.com/view/551692.htm)密钥交换协议/算法. 具体设置方式可以参考[这里](https://weakdh.org/sysadmin.html):

```
cd /usr/local/nginx/ssl/

# You will first need to generate a new Diffie-Hellman group, 
# regardless of the server software you use. Modern browsers, 
# including Google Chrome, Mozilla Firefox, and Microsoft Internet 
# Explorer have increased the minimum group size to 1024-bit. We 
# recommend that you generate a 2048-bit group. The simplest way of 
# generating a new group is to use OpenSSL:

openssl dhparam -out dhparams.pem 2048
```

创建[Certificate Transparency](https://imququ.com/post/certificate-transparency.html)生成文件的存放目录:

```
sudo mkdir /usr/local/nginx/sct/
```

获取SCT文件, 需要使用[ct-submit](https://github.com/grahamedgecombe/ct-submit)工具, 此工具使用golang编写, 因此需要使用golang进行编译.

```
sudo yum install golang

cd /usr/local/src/

wget -O ct-submit.zip -c https://github.com/grahamedgecombe/ct-submit/archive/v1.0.0.zip

unzip ct-submit.zip
cd ct-submit-1.0.0

go build
```

go build编译完成了之后, 会生成一个ct-submit-1.0.0可执行文件:

```
-rwxr-xr-x 1 admin admin 8001144 Nov 12 15:57 ct-submit-1.0.0
-rw-r--r-- 1 admin admin    3952 Nov 12  2015 ct-submit.go
-rw-r--r-- 1 admin admin     760 Nov 12  2015 LICENSE
-rw-r--r-- 1 admin admin    2041 Nov 12  2015 README.markdown
```

通过ct-submit-1.0.0工具将你的公匙提交给google:

```
./ct-submit-1.0.0 ct.googleapis.com/aviator </usr/local/nginx/ssl/aliyun_example_com.pem >/usr/local/nginx/sct/aliyun_example_com.sct

./ct-submit-1.0.0 ct1.digicert-ct.com/log </usr/local/nginx/ssl/aliyun_example_com.pem >/usr/local/nginx/sct/aliyun_example_com.sct
```

上面的两个命令访问google api时可能会出现以下timeout的错误, 解决方法是多试几次, 或者通过"科学上网"的方式.
```
panic: Post https://ct.googleapis.com/aviator/ct/v1/add-chain: dial tcp 216.58.197.10:443: i/o timeout
```

创建conf.d目录:

```
sudo mkdir /usr/local/nginx/conf/conf.d
```

### nginx main config.

以下的内容较多, 相关节点增加了配置说明. 笔者主要参考了:
[https://imququ.com/post/my-nginx-conf.html](https://imququ.com/post/my-nginx-conf.html): Jerry Qu大神的站点的配置.
[https://cipherli.st/](https://cipherli.st/): 主要介绍SSL证书的相关安全配置.
[https://gist.github.com/plentz/6737338](https://gist.github.com/plentz/6737338): 同样的是安全相关的配置, 比较好的一点是给出了一些引用说明.
[http://seanlook.com/2015/05/17/nginx-install-and-config/](http://seanlook.com/2015/05/17/nginx-install-and-config/): nginx安装配置的详解, 参考了部分http proxy, gzip等.

下面给出我的nginx.conf的配置内容, 在使用之前, 建议大家结合自己的环境, 搞懂每一个节点的具体作用, 切勿直接copy.

```
user  nobody nobody;
worker_processes  auto;

error_log  /alidata1/logs/nginx/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

pid        logs/nginx.pid;
worker_rlimit_nofile 65535;


events {
    use epoll;
    multi_accept on;
    worker_connections  2048;
}


http {
    include       mime.types;
    default_type  application/octet-stream;
    charset       UTF-8;
    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    tcp_nopush      on;
    tcp_nodelay     on;
        

    #keepalive_timeout  0;
    keepalive_timeout  60;
    
    gzip               on;
    gzip_vary          on;

    gzip_comp_level    6;
    gzip_buffers       16 8k;

    gzip_min_length    1000;
    gzip_proxied       any;
    gzip_disable       "msie6";

    gzip_http_version  1.0;

    gzip_types         text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript application/javascript image/svg+xml;

    brotli             on;
    brotli_comp_level  6;
    brotli_types       text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript application/javascript image/svg+xml;
    
    proxy_buffering on;
    proxy_buffers 64 32k;
    proxy_buffer_size 16k;
    proxy_temp_file_write_size 128k;
    proxy_max_temp_file_size 50m;

    proxy_connect_timeout 30;
    proxy_read_timeout 300;
    proxy_send_timeout 120;
    proxy_set_header X-Forwarded-Proto  $scheme;

    # don't send the nginx version number in error pages and Server header
    server_tokens      off;
    
    # config to don't allow the browser to render the page inside an frame or iframe
    # and avoid clickjacking http://en.wikipedia.org/wiki/Clickjacking
    # if you need to allow [i]frames, you can use SAMEORIGIN or even set an uri with ALLOW-FROM uri
    # https://developer.mozilla.org/en-US/docs/HTTP/X-Frame-Options
    add_header X-Frame-Options SAMEORIGIN;

    # when serving user-supplied content, include a X-Content-Type-Options: nosniff header along with the Content-Type: header,
    # to disable content-type sniffing on some browsers.
    # https://www.owasp.org/index.php/List_of_useful_HTTP_headers
    # currently suppoorted in IE > 8 http://blogs.msdn.com/b/ie/archive/2008/09/02/ie8-security-part-vi-beta-2-update.aspx
    # http://msdn.microsoft.com/en-us/library/ie/gg622941(v=vs.85).aspx
    # 'soon' on Firefox https://bugzilla.mozilla.org/show_bug.cgi?id=471020
    add_header X-Content-Type-Options nosniff;
    
    # This header enables the Cross-site scripting (XSS) filter built into most recent web browsers.
    # It's usually enabled by default anyway, so the role of this header is to re-enable the filter for 
    # this particular website if it was disabled by the user.
    # https://www.owasp.org/index.php/List_of_useful_HTTP_headers
    add_header X-XSS-Protection "1; mode=block";    

    # 由于部署了2台nginx, 此http header是为了识别是那台机器在处理请求, 从而帮助我们更好分析问题可能出现的原因.
    add_header x-server-id "02";

    ssl_ct               on;
    ssl_ct_static_scts   /usr/local/nginx/sct;

    ssl_certificate      /usr/local/nginx/ssl/aliyun_example_com.pem;
    ssl_certificate_key  /usr/local/nginx/ssl/aliyun_example_com.key;

    # Diffie-Hellman parameter for DHE ciphersuites, recommended 2048 bits
    # Please refer https://weakdh.org/sysadmin.html
    ssl_dhparam          /usr/local/nginx/ssl/dhparams.pem;

    # enables server-side protection from BEAST attacks
    # http://blog.ivanristic.com/2013/09/is-beast-still-a-threat.html
    ssl_prefer_server_ciphers on;

    # disable SSLv3(enabled by default since nginx 0.8.19) since it's less secure then TLS http://en.wikipedia.org/wiki/Secure_Sockets_Layer#SSL_3.0
    ssl_protocols        TLSv1 TLSv1.1 TLSv1.2;

    # https://github.com/cloudflare/sslconfig/blob/master/conf
    ssl_ciphers          EECDH+CHACHA20:EECDH+CHACHA20-draft:EECDH+AES128:RSA+AES128:EECDH+AES256:RSA+AES256:EECDH+3DES:RSA+3DES:!MD5;

    # enable session resumption to improve https performance
    # http://vincent.bernat.im/en/blog/2011-ssl-session-reuse-rfc5077.html
    ssl_session_cache    shared:SSL:50m;
    ssl_session_timeout  60m;

    # enable ocsp stapling (mechanism by which a site can convey certificate revocation information to visitors in a privacy-preserving, scalable manner)
    # http://blog.mozilla.org/security/2013/07/29/ocsp-stapling-in-firefox/
    ssl_stapling               on;
    ssl_stapling_verify        on;
    ssl_trusted_certificate    /usr/local/nginx/ssl/aliyun_example_com.pem;
    resolver                   223.5.5.5 223.6.6.6 valid=300s;
    resolver_timeout           10s;

    # config to enable HSTS(HTTP Strict Transport Security) https://developer.mozilla.org/en-US/docs/Security/HTTP_Strict_Transport_Security
    # to avoid ssl stripping https://en.wikipedia.org/wiki/SSL_stripping#SSL_stripping
    add_header Strict-Transport-Security "max-age=31536000; includeSubdomains; preload";

    # 其余所有的配置, 都存放到conf.d目录下.
    include /usr/local/nginx/conf/conf.d/*.conf;
}
```

### nginx server config for each application.
我们将web01和web02上部署的3个node.js application拆分为3个不同的server configuration:
	* port 9000: 部署了PC网站, 域名为: *.example.com, www.example.com
	* port 9001: 部署了m站点, 域名为: m.example.com
	* port 9002: 部署了app web api, 域名为: api.example.com

每一个server使用upstream_check_module做健康检查.

创建PC网站的server配置文件.

```
cd /usr/local/nginx/conf/conf.d/

sudo touch customeized_website_https_server.conf
```

配置文件的内容为:
```
upstream website {
    server      web01:9000 weight=1 max_fails=2 fail_timeout=15s;
    server      web02:9000 weight=1 max_fails=2 fail_timeout=15s;
    
    # upstream_check_module的相关配置, 请参考:
    # http://seanlook.com/2015/06/02/nginx-cache-check/
    check interval=5000 rise=2 fall=3 timeout=1000 type=http;

    # watchdog.html只是简单的response一个字符串, 当然也可以根据
    # 你的应用场景定制, 比如检测后端的Slave DB, 关键核心服务等.
    check_http_send "HEAD /watchdog.html HTTP/1.1\r\nConnection: keep-alive\r\nHost: web01.com\r\n\r\n";
    check_http_expect_alive http_2xx http_3xx;
}

server {
# 开启http2, 并且设置default_server. 
# 更多请参考:
# http://nginx.org/en/docs/http/configuring_https_servers.html#single_http_https_server
# http://nginx.org/en/docs/http/ngx_http_core_module.html#listen
    listen               443 ssl http2 default_server;
    server_name          example.com www.example.com;
    access_log           /alidata1/logs/nginx/example_com.log;

    location /{
        proxy_http_version   1.1;

        # 出于安全层面的考虑, 禁止输出web server相关的信息.
        proxy_hide_header    Vary;
        proxy_hide_header    X-Powered-By;

        proxy_set_header     Host             $host;
        proxy_set_header     X-Real_IP        $remote_addr;
        proxy_set_header     X-Forwarded-For  $proxy_add_x_forwarded_for;

        proxy_next_upstream  http_502 http_504 http_404 error timeout invalid_header;
        proxy_pass           http://website;
    }
    
    location /access-status {
        stub_status on;
        access_log  off;
    }

    location /check-status {
        check_status;
        access_log  off;
    }
}
```

**update 2016-12-01:**
> 1. 最开始的时候, 我将upstream check module配置为如下的方式:
```
check_http_send "GET /watchdog.html HTTP/1.1\r\n\r\n";
```
>    当后端server为node.js没有任何问题, 但是当后端server为IIS的时候, upstream check module会认为/watchdog.html无法被正常监控到. telnet对应的后端server端口和curl请求/watchdog.html都没有问题. 查看nginx的error.log, 发现有这样的一些错误日志:
```
[error] 27595#0: check protocol http error with peer: xx.xx.xxx.xxx:9000
```
>    于是google之后, 发现有很多人遇到相同的[问题](https://github.com/alibaba/tengine/issues/384).

>    最终找到了2种解决方案:
>    第一种方案: 将请求协议修改成HTTP/1.0
```
GET /watchdog.html HTTP/1.0\r\n\r\n
```

>    第二种方案: 不改变HTTP/1.1协议请求, 增加显式地指定Connection: keep-alive. 上述配置使用的是该方案
```
# 由GET调整为HEAD只是为了减少一些返回的请求字节数, 与此"坑"无关.
check_http_send "GET /watchdog.html HTTP/1.1\r\nConnection: keep-alive\r\nHost: web01.com\r\n\r\n";
```

> 2. 开启http_stub_status module, 用于监控基础的请求状况信息. 详细配置参考[官方文档](https://nginx.org/en/docs/http/ngx_http_stub_status_module.html).
```
location /access-status {
    stub_status on;
    access_log  off;
}
```
>    访问/access-status页面, 能看到如下的一些基本监控信息
![nginx-stub-module](https://www.zhuxiaodong.net/static/images/nginx-stub-module.png)

> 3. upstream check module提供了后端server健康状态监控页面
```
location /check-status {
    check_status;
    access_log  off;
}
```
>    访问/check-status页面
![upstream-check-module](https://www.zhuxiaodong.net/static/images/upstream-check-module.png)

创建M网站的server配置文件.

```
cd /usr/local/nginx/conf/conf.d/

sudo touch customeized_mobilesite_https_server.conf
```

配置文件的内容为:
```
upstream mobilesite {
    server      web01:9001 weight=1 max_fails=2 fail_timeout=15s;
    server      web02:9001 weight=1 max_fails=2 fail_timeout=15s;

    check interval=5000 rise=2 fall=3 timeout=1000 type=http;
    check_http_send "HEAD /watchdog.html HTTP/1.1\r\nConnection: keep-alive\r\nHost: web01.com\r\n\r\n";
}

server {
    listen               443 ssl http2;
    server_name          example.com www.example.com;
    access_log           /alidata1/logs/nginx/example_com.log;

    location /{
        proxy_http_version   1.1;

        proxy_hide_header    Vary;
        proxy_hide_header    X-Powered-By;

        proxy_set_header     Host             $host;
        proxy_set_header     X-Real_IP        $remote_addr;
        proxy_set_header     X-Forwarded-For  $proxy_add_x_forwarded_for;

        proxy_next_upstream  http_502 http_504 http_404 error timeout invalid_header;
        proxy_pass           http://mobilesite;
    }
    
    location /access-status {
        stub_status on;
        access_log  off;
    }

    location /check-status {
        check_status;
        access_log  off;
    }  
}
```

创建App web api的server配置文件.

```
cd /usr/local/nginx/conf/conf.d/

sudo touch customeized_webapi_https_server.conf
```

配置文件的内容为:
```
upstream webapi {
    server      web01:9002 weight=1 max_fails=2 fail_timeout=15s;
    server      web02:9002 weight=1 max_fails=2 fail_timeout=15s;
    
    check interval=5000 rise=2 fall=3 timeout=1000 type=http;
    check_http_send "HEAD /watchdog.html HTTP/1.1\r\nConnection: keep-alive\r\nHost: web01.com\r\n\r\n";
    check_http_expect_alive http_2xx http_3xx;
}

server {
    listen               443 ssl http2;
    server_name          example.com www.example.com;
    access_log           /alidata1/logs/nginx/example_com.log;

    location /{
        proxy_http_version   1.1;

        proxy_hide_header    Vary;
        proxy_hide_header    X-Powered-By;

        proxy_set_header     Host             $host;
        proxy_set_header     X-Real_IP        $remote_addr;
        proxy_set_header     X-Forwarded-For  $proxy_add_x_forwarded_for;

        proxy_next_upstream  http_502 http_504 http_404 error timeout invalid_header;
        proxy_pass           http://webapi;
    }
    
    location /access-status {
        stub_status on;
        access_log  off;
    }

    location /check-status {
        check_status;
        access_log  off;
    }  
}
```

最后, 我们需要将所有访问80端口的请求, 重定向给https的443端口, 以确保用户第一次访问时, 在浏览器地址栏内输入www.example.com时, 将http redirect to https:

```
cd /usr/local/nginx/conf/conf.d/

sudo touch customeized_all_http_server.conf
```

customeized_all_http_server.conf的配置为:
```
# redirect all http traffic to https
server {
    listen          80;
    server_name     .example.com;
    return 301      https://$host$request_uri;
}
```

上述步骤完成了之后, 使用nginx -t检测配置文件的正确性.

```
sudo /usr/local/nginx/sbin/nginx -t
```

如果得到以下的输出, 证明配置文件是正确的, 否则需要根据错误提示进行配置文件调整:
```
nginx: the configuration file /usr/local/nginx/conf/nginx.conf syntax is ok
nginx: configuration file /usr/local/nginx/conf/nginx.conf test is successful
```

最后, start or reload nginx service:
```
sudo systemctl start nginx.service

#or

sudo systemctl reload nginx.service
```

查看nginx的启动状态:
```
sudo systemctl status nginx.service

nginx.service - nginx
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled)
   Active: active (running) since Mon 2016-11-14 10:34:59 CST; 1 weeks 6 days ago
  Process: 32466 ExecStop=/usr/local/nginx/sbin/nginx -s quit (code=exited, status=0/SUCCESS)
  Process: 11489 ExecReload=/usr/local/nginx/sbin/nginx -s reload (code=exited, status=0/SUCCESS)
  Process: 32471 ExecStart=/usr/local/nginx/sbin/nginx (code=exited, status=0/SUCCESS)
 Main PID: 32472 (nginx)
   CGroup: /system.slice/nginx.service
           ├─11490 nginx: worker process
           ├─11491 nginx: worker process
           ├─11492 nginx: worker process
           ├─11493 nginx: worker process
           └─32472 nginx: master process /usr/local/nginx/sbin/nginx

Nov 14 10:48:59 systemd[1]: Reloading nginx.
Nov 14 10:48:59 systemd[1]: Reloaded nginx.
Nov 14 10:51:42 systemd[1]: Reloading nginx.
Nov 14 10:51:42 systemd[1]: Reloaded nginx.
Nov 14 11:19:04 systemd[1]: Reloading nginx.
Nov 14 11:19:04 systemd[1]: Reloaded nginx.
Nov 14 16:16:16 systemd[1]: Reloading nginx.
Nov 14 16:16:16 systemd[1]: Reloaded nginx.
Nov 14 21:04:09 systemd[1]: Reloading nginx.
Nov 14 21:04:09 systemd[1]: Reloaded nginx.
```

**update 2016-12-01:**
> 如果我们是重新编译nginx, 需要替换掉线上正在运行的可执行文件, 建议采取如下方式:

> 建立备份文件夹, 并将相关的配置文件和可执行文件备份
```
cd /usr/local/nginx/

mkdir bak/ && mkdir bak/conf && mkdir bak/sbin
# 这里假设只修改了nginx.conf主配置文件.
cp -R /usr/local/nginx/conf/nginx.conf /usr/local/nginx/bak/conf/.
```

> 以下假设我们为nginx新添加了module, 或者打了补丁, 需要重新编译. **注意, 不要执行make install**
```
cd /usr/local/src/nginx-1.11.5/
./configure ...
make
```

> 测试一下新编译出的nginx, 使用现有的配置文件是否正确. **使用objs/下的nginx做测试**

```
/usr/local/src/nginx-1.11.5/objs/nginx -t
```

> 测试通过了之后, 替换掉老的nginx可执行文件.

```
cp /usr/local/src/nginx-1.11.5/objs/nginx /usr/local/nginx/sbin/nginx 
```

> 大多数场景下, 可能遇到如下的错误:
```
cp:cannot create regular file `/usr/local/nginx/sbin/nginx': Text file busy
```

> 需要为cp命令添加-rfp参数:
```
cp -rfp /usr/local/src/nginx-1.11.5/objs/nginx /usr/local/nginx/sbin/nginx
```

> 最后, restart nginx. **注意: 不是reload**
```
systemctl restart nginx.service
```
