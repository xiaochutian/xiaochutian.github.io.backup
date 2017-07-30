---
layout:     post
title:      "Mac配置Charles代理环境"
subtitle:   ""
# date:       2016-06-20 10:23:36
author:     "K.I.S.S."
#header-img: ""
catalog:    true
tags:
    - Mac
    - Proxy
    - HTTPS
    - iOS
    - Android
---

## 0 一篇一句

> 当你的才华还撑不起你的野心的时候，你就应该静下心来学习；    
当你的能力还驾驭不了你的目标时，就应该沉下心来，历练。    
                                    ——by 莫言

## 1 概述

#### 1.1 Charles是什么

Charles is an HTTP proxy / HTTP monitor / Reverse Proxy that enables a developer to view all of the HTTP and SSL / HTTPS traffic between their machine and the Internet. This includes requests, responses and the HTTP headers (which contain the cookies and caching information).

Charles是一款HTTP代理，HTTP监听器，反向代理。它可以让开发者看到『机器』和『网络』之间的HTTP和HTTPS通信（包括：requests, response和headers）

#### 1.2 安装环境

- macOS Sierra 10.12.5
- Charles 4.0.1
- iOS 10.3.2
- Android 4.3

#### 1.3 需求

需要能在『iOS』和『Android』设备上监听HTTP和HTTPS请求

## 2 安装过程

#### 2.1 Charles

###### 2.1.1 下载安装
链接: [https://pan.baidu.com/s/1qYyRrnm](https://pan.baidu.com/s/1qYyRrnm) 密码: z3f8   

###### 2.1.2 配置HTTPS

打开SSL开关：『Proxy --> SSL Proxy Settings --> 勾选 Enable SSL Proxying』

![CharlesSslSettings](/img/in-post/post-setting-up-charles-in-mac-to-proxy-ios-and-android-device/charles-ssl-settings.jpeg)

###### 2.1.3 查看本机的IP和Port {#address}

- IP：『Help --> Local IP Address』
- Port：『Proxy --> Proxy Settins --> Proxies』（默认8888）

#### 2.2 iOS

###### 2.2.1 连上Mac中Charles的IP和Port {#connect}

wifi连上无线网，地址见[2.1.3节](#address)中查看到的IP和Port。

###### 2.2.2 安装证书 {#install_cert}

手机Safari中输入 http://www.charlesproxy.com/getssl 地址    
获取并安装 Charles 信任证书

> 注：建议使用Safari，Chrome可能加载不出来

###### 2.2.3 信任证书 {#trust_cert}

iOS 10.3需要手动信任安装的证书    
『设置 --> 通用 --> 关于本机 --> 证书信任设置 --> 启用设置』    
『Settings --> General --> About --> Certificate Trust Settings --> Enable』

###### 2.3 Android

Android的安装过程跟iOS类似，参见[2.2.1节](#connect)和[2.2.2节](#install_cert)

> 注：安卓安装过程中，如果你的手机没有密码，它会提示你先设置PIN码。

## 3 注意事项

1. 在Charles的SSL设置中，启用SSL
2. iOS 10.3需要对证书，手动设置信任。详见[2.2.3节](#trust_cert)
3. 安卓，如果没有设置密码，它会提示必须要有PIN码

## 4 参考

[【yanglei3kyou的专栏】Mac Charles 4.0+ 初步探讨(HTTP + HTTPS相关配置)](http://blog.csdn.net/yanglei3kyou/article/details/52571554)    
[【唐巧的博客】Charles 从入门到精通](http://blog.devtang.com/2015/11/14/charles-introduction/)    
[【简书】iOS 10.3下解决Charles抓包ssl证书信任问题](http://www.jianshu.com/p/6ad09374053b)
