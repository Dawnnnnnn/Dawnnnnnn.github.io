---
title: 用MacBookPro打造一个私有nas音乐库
date: 2025/07/21 19:30
updated: 2025/07/21 19:45
tags: [nas,Navidrome,MusicTagWeb,折腾,音流]
categories: 技术
description: 用MacBookPro打造一个私有nas音乐库
cover: https://api.vvhan.com/api/wallpaper/acg
---


## 背景

最近QQ音乐会员要过期了，经过深思熟虑后决定不续费了，主要有以下几个痛点:

1. 版权虽然比网易云音乐多，但也有部分歌是网易云独占，尤其是B站UP主的歌，导致这部分歌在QQ音乐上听不到
2. QQ音乐豪华绿钻已经限制设备数了，不太够用
3. QQ音乐For Mac拿我的电脑当PCDN用，在公司经常能看到它占用大量上传带宽

再考虑到我听的歌比较固定，平时也不用每日推荐功能，于是我萌生了一个打造私有nas音乐库的想法，正巧我有一台半坏不坏的MacBook Pro 2017款，只是WIFI出了点问题，插网线还是能用。所以就拿它当家庭nas吧

## 歌曲源获取

QQ音乐的比较好说，因为我目前还有会员，所以可以直接下载，下载后用解密工具批量解密就可以了，这部分收集了700+歌曲

网易云音乐因为没会员了，之前因版权问题从网易云迁移到QQ的时候同步迁移了一下歌单，但还是很多没迁移成功，所以这部分借助了一个公开的网易云歌曲下载API写脚本实现了对歌单的下载，这部分有200+歌曲


## 歌曲元数据刮削

上面两个渠道获取的音乐都是只有音频文件，缺少了封面、歌词等元数据，这部分我推荐使用[MusicTag](https://www.cnblogs.com/vinlxc/p/11347744.html)做一次存量的刮削，这个软件基本能解决95%的元数据缺失问题


## MacBookPro配置

1. 禁用自动休眠与节能设置
```bash
sudo systemsetup -setcomputersleep Never
sudo pmset -b sleep 0; sudo pmset -b disablesleep 1
```
2. 进入 系统设置 → 锁定屏幕 → 全部设为 “永不”
3. 启用自动登录（用户与群组 → 自动登录）
4. 基础软件安装
```bash
xcode-select --install
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
brew install orbstack ffmpeg
docker volume create portainer_data
docker run -d -p 8000:8000 -p 9443:9443 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:lts
```
5. 启用 SSH（系统设置 → 共享 → 远程登录）。设置仅允许公钥登录

vim /etc/ssh/sshd_config

```bash
PubkeyAuthentication yes
PasswordAuthentication no
ChallengeResponseAuthentication no
UsePAM no
```
```bash
sudo launchctl stop com.openssh.sshd
sudo launchctl start com.openssh.sshd
```
6. docker安装软件

因为我们前面已经安装了portainer，所以这里可以在网页上配置了，我配置了三个应用

- [heimdall](https://github.com/linuxserver/Heimdall): 用于进行内部导航，防止我记不住映射的端口号

![](https://static.dawnnnnnn.com/2025/07/81b14c6cd76e515437bf6b9d35f6f13f.png)
![](https://static.dawnnnnnn.com/2025/07/914231ad413a87dce290ca417a89d6ad.png)
![](https://static.dawnnnnnn.com/2025/07/e96aadc1673b1423254aef6e97a1512a.png)

- [Navidrome](https://github.com/navidrome/navidrome/): 用于进行音乐库管理

![](https://static.dawnnnnnn.com/2025/07/d5f2f52ee39575f92ad768a3d60b1c43.png)
![](https://static.dawnnnnnn.com/2025/07/2d1cf95fcf8d2b6950d18aa362638954.png)


- [music-tag-web](https://github.com/xhongc/music-tag-web): 用于对新进库的音乐进行自动刮削

![](https://static.dawnnnnnn.com/2025/07/8782ae79e21a162a12608d1b8de996e2.png)
![](https://static.dawnnnnnn.com/2025/07/d56549da8c355eaa0accfca17eccb8cc.png)


7. 启用 VNC (系统设置 → 共享 → 远程管理)，设置一个复杂的密码


8. 路由器配置

    1. 装一个DDNS-GO用于关联动态公网IP和域名
    ![](https://static.dawnnnnnn.com/2025/07/e67a567d56393c039142959841c91594.png)

    2. 路由器配置静态IP绑定
    ![](https://static.dawnnnnnn.com/2025/07/4950e1e03091062ec9e7c8868af87f42.png)

    3. 打开路由的UPNP功能
    ![](https://static.dawnnnnnn.com/2025/07/a922b4203f6afc53f94ea6659975395a.png)

    4. 进行端口转发
    ![](https://static.dawnnnnnn.com/2025/07/bc182380c17361be1bc0d352cdec48ec.png)


## 给BBDown打造一个WebUI

前面我们已经配置好了基础工作，存量的歌曲已经能听了，那么还需要解决一个增量的问题。由于B站是一个非常好的音乐社区，曲库大而全甚至还有翻唱资源。所以我直接使用BBDown作为音乐下载器就能覆盖90%的增量需求了

那么现在就需要解决 如何随时随地用手机/电脑调用BBDown 的问题，用Cursor半小时撸了一个，大概是这样

![](https://static.dawnnnnnn.com/2025/07/663a330ffffe441eb6164fea9c7d0948.png)

![](https://static.dawnnnnnn.com/2025/07/036cea4f273caea8b5adac3d9e5c9bdd.png)

![](https://static.dawnnnnnn.com/2025/07/ee2bb57fd8acfe93b1263a646eaed90a.png)

![](https://static.dawnnnnnn.com/2025/07/c804d22ff3e098ecdb44b0f46555d532.png)

在网页提交后会自动把音频下载到音乐库中


## 跨平台播放器

音流是一个跨平台的NAS音乐播放器，好用


![](https://static.dawnnnnnn.com/2025/07/abc432a9671f76edd7f02d1f46e82fc9.png)

