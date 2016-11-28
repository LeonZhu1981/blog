title: 基于阿里云上实现全站https的正确姿势(一)
date: 2016-11-21 21:34:00
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

# 一些重要的互联网资源参考:
---
强烈建议通读一下Jerry Qu的关于https, http/2, nginx的blog, 这是我目前发现的在国内的技术文章中, 关于上述的几个知识点讲解的最为全面透彻的文章.

传送门: [https://imququ.com/](https://imququ.com/)

# 为什么我们需要实现全站https?
---
目前主流大厂的网站和服务都已经实现了全站https, 例如: baidu, taobao, jd等. 
关于这方面的好处和优势, 互联网上太多文章在进行介绍. 例如: [为什么我们应该尽快升级到 HTTPS？](https://imququ.com/post/moving-to-https-asap.html)
对于一般的创业型公司, 迫切需要实现全站https的理由: 
1. [**苹果要求所有iOS应用在年底前默认使用HTTPS连接**](http://techcrunch.cn/2016/06/17/apple-will-require-https-connections-for-ios-apps-by-the-end-of-2016/)
2. [**微信小程序的开发, 要求必须使用https通信**](https://mp.weixin.qq.com/debug/wxadoc/dev/api/network-request.html?t=20161122)
3. [**HTTP/2的支持**](https://imququ.com/post/http2-resource.html) 

> HTTP2.0其实可以支持非HTTPS的，但是现在主流的浏览器chrome，firefox表示还是只支持基于TLS部署的HTTP2.0协议，所以要想升级成HTTP2.0还是先升级HTTPS为好.


# 升级之前的准备工作.
---
### SSL的证书类型?
* DV (Domain Validation): 面向个体用户，安全体系相对较弱，验证方式就是向 whois 信息中的邮箱发送邮件，按照邮件内容进行验证即可通过.
* OV (Organization Validation): 面向企业用户，证书在DV证书验证的基础上，还需要公司的授权，CA通过拨打信息库中公司的电话来确认.
* EV (Extended Validation): 在浏览器URL地址栏当中，展示了注册公司的信息，这会让用户产生更大的信任，这类证书的申请除了以上两个确认外，还需要公司提供金融机构的开户许可证，要求十分严格.

**需要强调的是，不论是 DV、OV 还是 EV 证书，其加密效果都是一样的！**

**重要的区别:**
1. DV证书的审核速度较快, 由程序完成审核, 一般申请了之后马上就能够完成证书颁发; OV和EV证书审核较慢, 由人工审核, 一般需要数天的时间.
2. EV证书会在浏览器当中显示公司的信息, 通常被称为绿色地址栏, 以增加用户的信任感. 参考如下访问github时, chrome浏览器的显示方式:
![EV证书示例](http://static.zhuxiaodong.net/blog/static/images/ev-cert.png)
3. EV证书**不支持单个泛域名(*.example.com)或者多个泛域名(*.example1.com, *.example2.com)**, OV和DV证书则支持.

**如何选择?**
通常的情况下, 一个处于初创阶段的互联网公司有多少个子系统是很难在前期就界定出来的. 因此, 如果能够无法明确规划出未来有多少个子系统, 需要使用多少个二级域名, 最好购买支持单个或者多个泛域名的证书. 至于是DV还是OV, 我个人觉得不是那么重要.

**一些经验总结:**
HTTPS网页中加载的HTTP资源被称之为[Mixed Content](https://imququ.com/post/sth-about-switch-to-https.html#toc-0), 不同的浏览器有不同的block http资源的策略, 因此最好提前整理出你的程序依赖的内部http资源, 和外部http资源(统计资源[GA, CNZZ, 百度统计], 第三方分享, 第三方地图服务等). 尤其是外部的http资源, 需要评估是否提供了https的服务. 

<!--more-->

### 到哪里获取https证书?

**免费资源:**
* [Let's Encrypt](https://letsencrypt.org/): 目前个人站点使用的非常普遍的证书.
特点/限制: 
	* 如果你在使用云服务提供商的CDN或者负载均衡服务(例如阿里云的CDN或者SLB), Let's 
Encrypt就不太适合了, 阿里云的CDN或者SLB要求在云平台上上传证书的公钥和私钥, 而Let's Encrypt强制90天过期, 虽然我们可以通过自动化的方式renew证书, 但我目前还没有找到比较好的解决方案.
	* 只支持单个域名.


* [阿里云Symantec免费DV证书](https://common-buy.aliyun.com/?spm=5176.2020520163.cas.1.ukbIQV&commodityCode=cas#/buy): 
特点/限制:
	* 赛门铁克-顶级证书颁发机构.
	* 1年过期.
	* 只支持单个域名.


* [腾讯云GeoTrust免费DV证书](http://www.qcloud.com/blog/?p=1237): 
特点/限制:
	* GeoTrust-顶级证书颁发机构.
	* 1年过期.
	* 只支持单个域名.

**强烈不推荐**:
1. [StartSSL.com](https://www.startssl.com): 详见官网的以下描述:
![startssl申明](http://static.zhuxiaodong.net/blog/static/images/startssl.png)

2. [Wosign(沃通)](https://www.wosign.com): 
[聊聊“沃通/WoSign”的那些破事儿](https://program-think.blogspot.com/2016/09/About-WoSign.html)
![wosign](http://static.zhuxiaodong.net/blog/static/images/wosign.png)
[Mozilla发布的wosign-issue](https://wiki.mozilla.org/CA:WoSign_Issues)
[新浪的报道](http://k.sina.cn/article_1772191555_69a17f430190016i6.html?vt=4)

> PS: 
> 1. 阿里云在之前的一段时间也能够购买wosign的证书, 目前也已经下线了.
> 2. wosign的SEO做得不错, 搜索**SSL证书**, google和baidu都排在了第一页, 导致很多不明真相的群众被忽悠了.

**付费资源:**
* [阿里云](https://common-buy.aliyun.com/?spm=5176.2020520163.cas.1.ukbIQV&commodityCode=cas#/buy): 提供了赛门铁克, GeoTrust, CFCA三个颁发的证书.

> PS: 其它都没有用过, 就不推荐了.

### 最终我选了什么?
阿里云上的GeoTrust DV SSL: 价格便宜, 支持泛域名, 颁发迅速(30 min之内), 最重要的是统一使用了阿里云进行管理.
![aliyunssl](http://static.zhuxiaodong.net/blog/static/images/aliyunssl.png)

后续内容, 请继续参考下一篇[文章](http://www.zhuxiaodong.net/2016/how-to-switch-your-website-to-https-on-aliyun-part2/)
