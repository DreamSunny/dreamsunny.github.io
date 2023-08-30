---
layout:     post
title:      "SystemUI-支持gradle编译"
subtitle:   ""
date:       2022-03-20 12:00:00
author:     "Yaku"
header-img: "img/post-bg-2015.jpg"
tags:
    - Android
    - SystemUI
---



### 1. 导入framework.jar

在源码根目录下执行：make framework

在R以下拷贝 out/target/common/obj/JAVA_LIBRARIES/framework-full_intermediates/classes.jar 重命名为 framework.jar。

在R及以上拷贝 out/target/common/obj/JAVA_LIBRARIES/framework_intermediates/classes.jar 重命名为 framework.jar。

在项目根目录下的 build.gradle 将 framework.jar 设置为核心类库：

```
allprojects {
  gradle.projectsEvaluated {
    tasks.withType(JavaCompile) {
      options.compilerArgs << '-Xbootclasspath/p:libs/framework.jar'
    }
  }
}
```

| 扩展 BootStrap 级别类         | 描述                                                         |
| ----------------------------- | ------------------------------------------------------------ |
| -Xbootclasspath:bootclasspath | 让jvm从指定的路径中加载bootclass，用来替换jdk的rt.jar。一般不会用到。 |
| -Xbootclasspath/a:path        | 被指定的文件追加到默认的bootstrap路径中。                    |
| -Xbootclasspath/p:path        | 让jvm优先于默认的bootstrap去加载path中指定的class。          |

在模块下的 build.gradle 中添加依赖：

```
dependencies {
  compileOnly files('libs/framework.jar')
}
```



### 2.处理 internal 资源

SystemUI项目中有两种使用系统 internal 资源的方式：

- 在java代码中通过 com.android.internal.R.xxx 来引用。

因为 R 文件是在 framework.jar 中的，所以采用 gradle 编译没有影响。



- 在资源文件xml中通过 @*android:xxx/xxx 引用。

aapt2 在编译资源文件时会对引用资源进行转换，分为两种：

1.转换为数值：match_parent 转换 -1，wrap_content 转换为 -2；dimen值转换为 dimension(value左移8位+1)

2.转换为引用：@ref/资源id



因为在编译期间无法关联到 internal 资源id，会将引用的 internal 资源作为新资源生成一个新id，而运行期间找不到这个新id的资源值，从而引发异常。



**解决方案：**

将所有引用到 @*android:xxx/xxx 资源的资源文件拷贝到一个新的目录下，通过脚本将 @*android:xxx/xxx 替换成具体的值。

在 build.gradle 中通过 flavorDimensions + productFlavors 方式将新目录下资源文件覆盖掉项目中引用到 internal 资源的文件。

如果布局文件引用了 @*android:id/ 资源且又在 Java 代码通过 findViewById() 方式查找，这种只能重新声明一个资源id来规避。



### 3.处理资源冲突

SystemUI支持gradle编译要解决的另一大难题是资源冲突。

原生系统中 SystemUI 是依赖 SettingsLib 编译，其中 SystemUI 的代码中就有引用 SettingsLib 的资源。我们期望是通过 make SettingsLib 可以编译出一个包含res资源的aar包，但实际上aar包中是没有res资源的。其原因是 android_library 被设计为 android_app 模块提供依赖，编译产出为 .jar + package-res.apk。



**可选方案如下：**

方案一：SettingsLib支持gradle编译，生成包含res的aar。

这种方案就是成本会高一些，SettingsLib依赖简单，但子模块非常多。


方案二：[资源合并](https://developer.android.com/studio/write/add-resources.html#resource_merging)

如果存在同一资源的两个或多个匹配版本，则只有一个版本会包含在最终 APK 中。构建工具会根据以下优先级顺序（左侧的优先级最高）选择要保留的版本：

构建变体 > build 类型 > 产品变种 > 主源代码集 > 库依赖项

将 SettingsLib 资源文件拷贝到 SystemUI，采用 flavorDimensions + productFlavors 实现优先引用资源：
```
flavorDimensions "overlay"
productFlavors {
  overlay {
    dimension "overlay"
  }
}
sourceSets {
  overlay {
    res.srcDirs = ['res-overlay']
  }
}
```