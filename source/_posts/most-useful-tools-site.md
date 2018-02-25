title: 我收藏的常用工具和网站
date: 2016-08-31 14:08:26
categories: programming
tags:
- tools
---

## 命令行常用快捷键:
---
```
空格键: 预览
cmd + ,: 设置
cmd + -/=: 缩小/放大
ctrl + u: 删除到行首(与zsh冲突, zsh中是删除整行)
ctrl + k: 删除到行尾
ctrl + p/n: 上/下移动一行或者前/后一个命令
ctrl + b/f: 光标前/后移char
esc + b/f: 光标前/后移word(蛋疼不能连续work)
ctrl + a/e: 到行首/行尾
ctrl + h/d: 删前/后字符
ctrl + y: 粘贴
ctrl + w: 删除前一个单词
esc + d: 删后一个单词
ctrl + _: undo
ctrl + r: bck-i-search/reverse-i-search, 输入关键字搜索历史命令
```

## 学习网站:
---
[**Learn X in Y minutes**](https://learnxinyminutes.com/): 快速学习各种相关知识的网站. 强调在几分钟之内对该技术能够有大概的了解和认识.
[**实验楼**](https://www.shiyanlou.com/): 提供学习该项技术的实验环境(虚拟机环境), 部分课程需要收费, 会员也需要收费.
[**慕课网**](http://www.imooc.com/): 课程很丰富, 最大的靓点是全免费.
[**菜鸟网**](http://www.runoob.com/): 文档教学类网站, 优点是涉及的技术知识非常全面, 很适合入门学习.

## 优秀的个人技术Blog:
---
[**后端技术杂谈**](http://www.rowkey.me/): 后端技术Java, 架构.
[**iMysql**](http://imysql.com/): Mysql层面有一些比较深入的文章.
[**火丁笔记**](http://huoding.com/): 有一些非常有技术含量的后端技术文章.

## Linux:
---
[**bash man**](http://man.linuxde.net/): 简洁的Linux bash shell帮助文档, 以命令或者分类的方式搜索, 并提供较为详细的示例.

[**the geek stuff**](http://www.thegeekstuff.com/): linux学习的网站, 示例非常简洁, 清晰. 比较好的文章包括:
* http://www.thegeekstuff.com/2011/12/linux-performance-monitoring-tools/
* http://www.thegeekstuff.com/2010/11/50-linux-commands/
* http://www.thegeekstuff.com/2010/12/50-unix-linux-sysadmin-tutorials/

[**view large file**](https://www.cyberciti.biz/faq/find-large-files-linux/): 查看大文件.

[**du & df**](http://os.51cto.com/art/201012/240726_all.htm): 查看磁盘空间.

[**centos rpm 安装相关**](https://wiki.centos.org/zh/TipsAndTricks/YumAndRPM): 尤其是如下这条, 在服务器上通过yum install安装, 但是镜像源下载rpm包超级慢的情况, 可以找一台网络比较好的PC, 用迅雷等下载软件加速下载, 再copy到服务器上之后进行安装, 此时,可能会使用到:

```
# 用 yum 去安装本地组件，并自动地检查／满足依赖性
yum --nogpgcheck localinstall packagename.arch.rpm
```

[**Linux各个版本的源代码在线阅读**](http://elixir.free-electrons.com/linux/v2.6.39.4/source)

<!--more-->

## Web前端:
---
[**Can I Use**](http://caniuse.com/): Raw browser/feature support data from caniuse.com

## 运维:
---
[运维生存时间](http://www.ttlsa.com/): 运维相关的高质量文章.
[Linux运维笔记](https://blog.linuxeye.com/): 维护的lnmp、lamp、lnmpa一键安装包有一些特点.
[阿里云官方文档](https://help.aliyun.com/knowledge_list/8314850.html?spm=5176.789005859.3.2.DDVZxt): 阿里云官方的FAQ, 有一些比较有价值的和ECS服务器相关的经验总结.

## 常用工具类网站:
---
[**searchcode**](https://searchcode.com/): 一个非常好的搜索源代码的网站，集成了github，google code，BitBucket等知名代码托管平台的代码。

[**grepcode**](http://grepcode.com/): 搜索Java的标准库的代码会更好用，并且支持不同的JDK版本的过滤。

[**observatory**](https://mozilla.github.io/http-observatory-website/): 测试你的网站的安全漏洞, 并给出建议和评分.

[**StatCounter**](http://gs.statcounter.com/): 统计浏览器的市场份额, 数据统计的更新频率很高, 统计筛选的功能很丰富.

[**RGB转16进制工具**](http://c.runoob.com/front-end/55): RGB和16进制的颜色转换.

[**菜鸟工具**](http://c.runoob.com/): 涵盖绝大多数语言的在线运行和编译工具, 以及前端相关的工具: JSON格式化/图片转BASE64/js压缩 etc.

[**thoughtworks radar**](https://www.thoughtworks.com/radar): 评估使用的技术/语言/平台是否靠谱.

[**bugly SDK**](https://dsx.bugly.qq.com/): 提供android studio/XCode/gradle的国内镜像, 提升下载速度.

[**google trends**](https://www.google.com/trends/): 根据google趋势查看各个技术的热度.

[**站长工具**](http://ip.chinaz.com/): 查询IP、tracroute、网站信息查询、SEO查询....

[**Online book convertor**](http://onlineconverter.epubor.com/)

[**gif split**](http://ezgif.com/split):

[**gif maker**](http://loading.io/):

[**二维码生成**](https://cli.im/):

[**字体产生ascii**](http://www.network-science.de/ascii/)

[**ascii码流程图工具**](http://asciiflow.com/)

## windows效率工具:
---
[**listary**](http://www.listary.com/): windows下堪比Alfred的神器.
[**Cmder**](http://www.jeffjade.com/2016/01/13/2016-01-13-windows-software-cmder/)

## Mac效率工具:
https://github.com/jaywcjlove/awesome-mac

## 2的n次幂速算表:

![2-n-table](https://www.zhuxiaodong.net/static/images/2-n-table.png)

## most tech in interview:

![most-userful-tech-stack](https://www.zhuxiaodong.net/static/images/most-userful-tech-stack.png)

## Front-end checklist:

[**https://github.com/thedaviddias/Front-End-Checklist**](https://github.com/thedaviddias/Front-End-Checklist)

## 计算机词汇正确发音:

[**这是**](https://github.com/shimohq/chinese-programmer-wrong-pronunciation): github上一个有趣的项目，帮助大家纠正常见技术词汇的错误发音，本质上是通过调用有道词典的：http://dict.youdao.com/dictvoice?audio={传入的单词}&type=1

## SSL

[https://mozilla.github.io/server-side-tls/ssl-config-generator/](https://mozilla.github.io/server-side-tls/ssl-config-generator/) : Mozilla SSL Configuration Generator 集成了常见的 web 服务器的 SSL 相关配置，包括 nginx apache HAProxy 等。

[https://wiki.mozilla.org/Security/Server_Side_TLS](https://wiki.mozilla.org/Security/Server_Side_TLS) : mozilla 发布的关于 TLS 相关规范建议。
