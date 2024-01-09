---
layout: post
title: "自签名证书浏览器调试"
description: "自签名证书浏览器调试"
date: 2023-12-28
tags: [Chrome, Certificate]
---


[来源](https://blog.csdn.net/XWdongbei/article/details/114536816)

问题描述
在使用google浏览器时，有时访问某些网址时会显示网址不安全，查看后发现网络证书无效

解决方法
1、桌面找到google浏览器图标，右键，选择属性

2、在''目标''后空一格添加--ignore-certificate-errors --allow-running-insecure-content 

3、重启即可

然而，有时还会出现问题，这是，需要在出现问题的网址上盲打  thisisunsafe
