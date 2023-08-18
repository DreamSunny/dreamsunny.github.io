---
layout:     post
title:      "Recovery机制介绍"
subtitle:   ""
date:       2020-5-29 12:00:00
author:     "Yaku"
header-img: "img/post-bg-2015.jpg"
tags:
    - Android
    - SystemUI
    - Recovery
---



2020年三星手机发生了一场严重事故，中国三星手机用户在5月23日这一天凌晨突然无限重启，事故原因是因SystemUI发生异常重复Crash导致手机进入Recovery界面，用户只能选择恢复出厂设置导致用户数据永久丢失。

[怎么看待三星大量手机在今天（5.23）凌晨系统崩溃并数据丢失？]( https://www.zhihu.com/question/396666758/answer/1245994988)



知乎上已经有大佬详细分析了事故原因，本文重点介绍下为什么SystemUI频繁Crash会进入Recovery界面。



# **1. ResureParty功能初始化**

ResureParty是一套系统自救机制，当监控到系统核心程序出现循环崩溃时，通过尝试一系列措施达到恢复设备正常运行的目的，直到所有措施都无济于事，则会进入Recovery界面让用户通过擦除数据的方式进行设备恢复。

 其初始化流程如下图：

![r1](/img/Recovery/r1.png)

在SystemServer启动时，会初始化ResureParty机制。核心逻辑在PackageWatchDog，当应用发生异常退出时，PackageWatchDog会记录相关异常信息。

​                                      

# **2. SystemUI发生Crash被系统重新拉起**

SystemUI是常驻进程，当其发生异常退出时，系统会尝试重新启动，其流程图如下：

![r2](/img/Recovery/r2.png)

ActivityManagerService中收到应用死亡的回调后，会判断应用是否为系统常驻进程。如果应用是系统常驻进程，则会重新启动该进程。

# **3. SystemUI频繁重启触发ResureParty机制**                                  

当SystemUI发生异常退出时，系统会重新启动SystemUI。但如果SystemUI的异常状态没有恢复，即重启后依然发生异常退出，则会陷入到异常退出-重启-异常退出-重启...这种循环崩溃的状态，此时就会触发ResureParty机制。

 触发ResureParty机制的流程图如下：

![r3](/img/Recovery/r3.png)

当应用异常退出后，会上报到PackageWatchDog，其onPackageFailureLocked方法中会判断当前应用是否为系统常驻进程，如果应用是系统常驻进程，则会更新mitigationCount，其对应的严重等级如下表：                                      

| mitigationCount | 严重等级                                |
| --------------- | --------------------------------------- |
| 1               | LEVEL_RESET_SETTINGS_UNTRUSTED_DEFAULTS |
| 2               | LEVEL_RESET_SETTINGS_UNTRUSTED_CHANGES  |
| 3               | LEVEL_RESET_SETTINGS_TRUSTED_DEFAULTS   |
| 4               | LEVEL_WARM_REBOOT                       |
| 5               | LEVEL_FACTORY_RESET                     |

当达到严重等级达到LEVEL_FACTORY_RESET级别后，系统就会进入Recovery界面2