title: How to debug webview on ios simulator
date: 2017-08-16 21:08:04
categories: programming
tags:
- ios
- webview
- debug
---

# 应用场景：
---

目前大部分native app当中，都会通过webview加载mobisite（移动站点）的方式来实现某些对交互要求不高的功能。而webview的工作方式并不等同于真机设备上浏览器（例如：iOS Safari）的工作方式，明明在真机设备上工作很好的页面，放到native app中使用webview呈现时，总会有这样或者那样的问题。因此，我们需要能够在webview当中去debug这些页面。

<!--more-->

一些说明：

* 关于如何在iOS真机设备的浏览器（Safari）上调试页面，网上的教程非常多，但我总觉得不太实用，设想一下，很少有人会直接打开移动端的浏览器去浏览我们开发的mobisite，入口要么是微信（服务号，朋友圈分享），要么是native app（通过webview加载）。

* 某些方案（例如：[这里](http://www.saitjr.com/ios/ios-user-safari-debug-webview.html)）当中提到的，写一个最简单的iOS native App，这个native App当中就只有一个webview，然后手动修改需要加载的URL地址。个人认为这种方式也无法重现在特定的native app当中webview出现的问题（例如这种方式native app并没有顶部导航条，或者是由于UIWebView的某些参数设置与实际的native app不一致）。

> NOTE: 笔者最开始也是使用的此种方式，有一个小坑需要绕过，默认的情况下，如果访问的非https的URL，会无法正常加载。解决方案是设置一下plist文件：

```
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsArbitraryLoads</key>
    <true/>
</dict>
```

> ref: https://segmentfault.com/a/1190000002933776

* 本文所涉及的方式只适合在mac下进行调试，windows下如何调试还有待研究。

# 如果能够拿到native app的source code：
---

恭喜你，比起无法拿到源代码的方式要少折腾许多。只要在你的机器上编译通过，Command + R就能够在iOS模拟器当中运行。此时无论是使用Safari还是[ios_webkit_debug_proxy](https://github.com/google/ios-webkit-debug-proxy)方便地进行调试。

# 如果由于某种原因，无法拿到native app的source code：
---

**step 1: 请你的iOS开发团队将相关的native app编译好的iOS模拟器包发给你**

在哪里可以找到这个iOS模拟器包？

首先在xcode项目导航的Products目录下，Show in Finder。
![products-show-in-finder](https://www.zhuxiaodong.net/static/images/products-show-in-finder.png)

对应的.app文件我们需要的能够在iOS模拟器上正常运行的包文件.
![debug-simulator](https://www.zhuxiaodong.net/static/images/debug-simulator.png)

> 注意：普通的ipa包虽然也可以通过下面所描述的方式安装到模拟器上，但是一运行就会闪退，原因是由于ipa包编译是使用的ARM架构，而非X86架构。

**step 2: 通过xcrun simctl install命令进行安装。安装之前需要启动模拟器，然后在终端当中运行：**

```
xcrun simctl install booted <完整路径的app文件名>
```

**step 3：如何调试？**

这方面的教程也很多，总结下来，要么使用Safari，要么使用chrome[基于ios_webkit_debug_proxy](https://github.com/google/ios-webkit-debug-proxy)。

使用Safafi的方式：
* http://www.saitjr.com/ios/ios-user-safari-debug-webview.html

使用chrome的方式：
* http://konishilee.com/blog/2017/03/Front-End-about-chrome-inspector.html

* http://liyaogang.com/ios%E8%B0%83%E8%AF%95uiwebview%E5%92%8Cwkwebview%E5%8A%A0%E8%BD%BD%E7%9A%84h5%E9%A1%B5%E9%9D%A2/

# 其它：
---

xcrun simctl是一个很强大的工具，命令行最强大之处就是可以做到批量化和自动化，例如taobao的FED团队介绍的[MDS（Mobile pages Develop & Debug Solution）](http://taobaofed.org/blog/2015/11/13/web-debug-in-ios/)。

强烈推荐阅读这篇关于xcrun simctl的文章：

http://shashikantjagtap.net/simctl-control-ios-simulators-command-line/


参考文章：
http://www.tracenote.com/2017/07/29/simulatorAppInstall/
https://www.careiphone.com/how-to-install-an-application-on-xcode-simulator-of-ios-simulator/
http://www.anexinet.com/blog/install-app-ios-simulator/


