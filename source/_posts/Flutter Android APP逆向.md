---
title: Flutter Android APP逆向系列(一)
date: 2024/06/26 17:56
updated: 2024/06/26 17:56
tags: [逆向,Android,Flutter,安全技术,渗透测试]
categories: 安全技术
description: 一些经验积累
cover: https://api.vvhan.com/api/wallpaper/acg
---

##  简介

因为内容太多所以分三篇讲，一篇讲一个案例


## Flutter APP逆向现状

1. Flutter APP网络请求默认不会走系统代理
2. Flutter APP网络请求默认没有SSLPinning，但大多数应用都会配置SSLPinning
3. 代码逻辑基本在`libapp.so`中，无法直接反编译出源码。应用依赖`libflutter.so`引擎工作，常规Android Java层逆向手段无效

## reflutter工具

[reflutter](https://github.com/Impact-I/reFlutter)作者有[一篇文章](https://swarm.ptsecurity.com/fork-bomb-for-flutter/)，写的非常详细，我就挑重点说一下

reflutter有两种模式，抓包(Traffic monitoring and interception)和dump函数偏移(Display absolute code offset for functions)

这两种模式本质上都是git clone flutter enigne后对关键内容进行patch，再重新编译出一个`libflutter.so`

比如抓包模式实际上是对`src/third_party/dart/runtime/bin/socket.cc`中的`FUNCTION_NAME(Socket_CreateConnect)(Dart_NativeArguments args)`进行patch，指定该应用流量通过`192.168.133.104:8083`代理发送出去。除此之外还处理了`src/third_party/boringssl/src/ssl/ssl_x509.cc`中的`ssl_crypto_x509_session_verify_cert_chain`，用于去掉证书校验

## blutter工具

[blutter](https://github.com/worawit/blutter)作者有[一篇PPT](https://conference.hitb.org/hitbsecconf2023hkt/materials/D2%20COMMSEC%20-%20B(l)utter%20%E2%80%93%20Reversing%20Flutter%20Applications%20by%20using%20Dart%20Runtime%20-%20Worawit%20Wangwarunyoo.pdf)，写的非常详细，我总结一下

blutter是基于官方Dart Runtime，并调用其内部API来静态获取快照等信息的 

## 实战案例
目标: 抓包并获取目标APP API的加密方式

已知：目标APP有360加固、没有SSLPinning

### 解决抓包问题
先reflutter一把

![](https://static.dawnnnnnn.com/2024/06/b62754115e4f3815274fa5ebb0a76aac.png)

然后安装

![](https://static.dawnnnnnn.com/2024/06/76615f216b18b94870f90a64171fc077.png)

不出意外的失败了，这是因为还没给apk签名。可以通过各种方式给这个apk签名，我比较常用的是MT管理器

签名完成再安装就可以成功了

![](https://static.dawnnnnnn.com/2024/06/0287276ef8b195aa61d0e06ce8076af2.png)

然后我们启动一下，不出意外的闪退了，因为这个APP是有加固的，reflutter的重打包不太够用了

![](https://static.dawnnnnnn.com/2024/06/acbece9665b4ffbd7e1285fa45368f47.png)

上面我们已经简述了reflutter的原理，实际上生效的就是那个重编译的libflutter.so而已，那么我们能不能直接用这个so呢，就能绕过重打包的过程。那结论肯定是可以的

我们先通过lsof -p pid找到目标APP的libflutter.so位置

![](https://static.dawnnnnnn.com/2024/06/437e33acbad09012ff4aaea9b3706c7f.png)

然后解压reflutter生成的release.RE.apk文件，找到它的libflutter.so，push到模拟器内，然后mv，chmod 777

![](https://static.dawnnnnnn.com/2024/06/9e9cfc3c8fc94a176d4ddb0d0a8b9406.png)

然后我们给模拟器配置代理

![](https://static.dawnnnnnn.com/2024/06/a68aafceb04e03c98a26ba5efd03770a.png)

然后开始抓包

![](https://static.dawnnnnnn.com/2024/06/1b74d589dc06bff8405155c6f1a73028.png)

可以看到现在所有的包都可以抓到了

### 寻找sign值算法

从上面的抓包结果中可以看到header中有一个sign值，我们现在需要找到它的生成方式

现在我们使用reflutter和blutter应该效果是一样的，都能dump出相应的函数名。但是我习惯使用blutter，因为它们的运行方式不同，reflutter是动态dump，blutter是静态的分析。blutter还能输出IDA可用的脚本，所以我们先跑一下blutter看看效果

![](https://static.dawnnnnnn.com/2024/06/0dbfb3c411e925900420af17e150d5f4.png)

![](https://static.dawnnnnnn.com/2024/06/9f364c172479ab777970d7ba6a3b7c71.png)

我们先用IDA看一下libapp.so，在这里导入blutter生成的IDA脚本还原函数符号

![](https://static.dawnnnnnn.com/2024/06/cd3503995743f3405575b39beb733d7e.png)

等待分析完后搜索sign

![](https://static.dawnnnnnn.com/2024/06/c6c6f7c1ed6f169bdeae17e7f69e75b0.png)

大概就能定位到sign是在这些函数里生成的了

然后我们开始frida进行动态分析

先hook看看这个getSign函数是干嘛的，在IDA里可以看到它的返回值是一个md5

![](https://static.dawnnnnnn.com/2024/06/7763deca7950549cdb237afcb7f775ad.png)

我稍微改了一下blutter_frida.js这个文件，大概像这样，0xcaf2a0是getSign函数的地址，然后我加了一个打印返回值的功能dumpArgs

![](https://static.dawnnnnnn.com/2024/06/97126b8cb08699f7f235af5a58b1a8d5.png)

跑一下frida

![](https://static.dawnnnnnn.com/2024/06/2216bd3e312684f840aab022f3c2f577.png)

我们可以看到这个函数的入参是 String@780038e211 = "gqaPpGrN1MCK3fVV4Vfoym65YkwO4vJL"，返回值是 9CEBBC69BC8D43991968209577DBF64B

在抓包软件里确认一下

![](https://static.dawnnnnnn.com/2024/06/df7d2d6b7cd6fb06c8f82982625c4635.png)

完全能对上，这个getSign_caf2a0函数的返回值就是我们要的sign

我们再看看这个函数的入参究竟是什么，在IDA里面跟一下

![](https://static.dawnnnnnn.com/2024/06/1610fbf4f89990b708dab4bd7e4e07ea.png)

是一个signKey


我们再看一下generateMd5_caf550的逻辑

![](https://static.dawnnnnnn.com/2024/06/4daee3572cc4f0cb5a1ec62e7672b92a.png)

这个函数很简单，一眼就能看出convert_c7706c是md5加密前的最后一步，那hook这里也许就能拿到被加密的字符串，试一下

![](https://static.dawnnnnnn.com/2024/06/40b1e0313c989151323e03b562dc51cf.png)

那么直接推测sign的算法是 md5(f"path{path}timestamp{timestamp}version1.0.0{signKey}")

## 总结

实际的逆向过程并没有这么简单，hook的函数并不止以上提到的这些。以上只是我精炼的最速通关思路，逆向走弯路是难免的，多熟悉熟悉就好了