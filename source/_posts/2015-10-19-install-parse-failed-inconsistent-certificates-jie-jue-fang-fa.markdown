---
layout: post
title: "INSTALL_PARSE_FAILED_INCONSISTENT_CERTIFICATES 解决方法"
date: 2015-09-25 10:15:04 +0800
comments: true
categories: Android
---
  在Nexus手机上安装APK文件时出现类似INSTALL_PARSE_FAILED_INCONSISTENT_CERTIFICATES的提示，这样的问题主要是签名冲突造成的，比如你使用了ADB的debug权限签名，但后来使用标准sign签名后再安装同一个文件会出现这样的错误提示，解决的方法很简单，就是卸载原有版本再进行安装，而adb install -r参数也无法解决这个问题。