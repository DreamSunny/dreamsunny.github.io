---
layout:     post
title:      "Broadcast介绍"
subtitle:   ""
date:       2021-09-20 12:00:00
author:     "Yaku"
header-img: "img/post-bg-2015.jpg"
tags:
    - Android
    - Broadcast
---

官方文档：[广播概览](https://developer.android.com/guide/components/broadcasts?hl=zh-cn)

# 1.注册广播：registerReceiver

![img](/img/broadcast/register.png)

涉及的知识点： 

显示广播与隐式广播 

两种注册方式：静态注册与动态注册的区别。 

安全：权限限制。 

AMS：广播与AMS的关联。 

设计模式：观察者模式 

# 2.发送广播：sendBroadcast

![img](/img/broadcast/send.png)

涉及的知识点： 

两种发送方式：有序广播和无序广播的区别。 

前台广播和后台广播。

粘性广播：在 Android 5.0（API 级别 21）中已废弃

本地广播：LocalBroadcastManager.sendBroadcast 