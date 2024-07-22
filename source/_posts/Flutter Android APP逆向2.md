---
title: Flutter Android APP逆向系列(二)
date: 2024/07/22 16:56
updated: 2024/07/22 16:56
tags: [逆向,Android,Flutter,安全技术,渗透测试]
categories: 安全技术
description: 一些经验积累
cover: https://api.vvhan.com/api/wallpaper/acg
---


## 实战案例
目标: 抓包目标APP

已知：未加固

## 解决抓包问题

首先按照上一篇的方式尝试直接替换libflutter.so文件，先根据pid定位文件位置

![](https://static.dawnnnnnn.com/2024/07/a67cb979babb2a3a24d59640f634b3aa.png)

这时候我们发现找不到这个so文件，这就比较离奇了，因为我们直接解压是可以看到这个文件的

![](https://static.dawnnnnnn.com/2024/07/f6b4003f31ef0fb8037394dc48bb2b84.png)

这种情况下，我们可以考虑替换这个文件并重打包，也就是回到最原始的使用方法。

当然，经过我的测试，这种方法还是毫无意义的失败了，失败原因在于apktool文件对apk的资源文件解包有问题，一些xml文件会被重命名，这就导致在apktool重打包后，app启动时会找不到对应的xml资源文件从而导致应用闪退

经过我的不断尝试，使用MT管理器直接替换apk中的.so文件，可以成功

![](https://static.dawnnnnnn.com/2024/07/ce2ec2d5f15e1ba03bfe1ce54e58cb98.png)


![](https://static.dawnnnnnn.com/2024/07/95f82a4da29d9edcf47a3ee4242b0d23.png)


![](https://static.dawnnnnnn.com/2024/07/52424c83c0756f66a199e271c0dcd414.png)

剩下的就是常规IDA逆向+Frida动态调试的过程了，有手就行