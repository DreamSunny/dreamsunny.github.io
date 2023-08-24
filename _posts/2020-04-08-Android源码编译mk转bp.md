---
layout:     post
title:      "Android源码编译mk转bp"
subtitle:   ""
date:       2020-04-08 12:00:00
author:     "Yaku"
header-img: "img/post-bg-2015.jpg"
tags:
    - Android
---



### 1. 基本概念介绍

自Android N开始，Google开始用Ninja来替代Makefile编译系统。编译时，会先把Android.mk通过kati转换成.ninja文件，然后使用ninja命令进行编译。

Android.bp是替代Android.mk的配置文件，在Android O默认开启相关支持。编译时，由Soong解析转换成.ninja文件。

Soong是为Android编译设计的工具，Blueprint和Soong都是由Golang写的项目，Blueprint解析文件的格式，而Soong解释内容的含义，bp即Blueprint的缩写。

与Android.mk不同的是，Android.bp是纯粹的配置文件，不包含分支、循环等流程控制，也不能做算数、逻辑运算，需使用Golang编写控制逻辑。

注意：Android.mk文件中的模块可以依赖Android.bp文件中的模块，但不能相反。

### 2. Android.bp文件格式

Android.bp文件的语法和语义与Bazel BUILD文件类似，参考：[bazel](https://docs.bazel.build/versions/master/be/overview.html)

模块：Android.bp文件中的模块以一个模块类型开始，后面跟着一组属性，以名值对(name: value)表示。每个模块必须有一个name属性，并且在所有的Android.bp文件中必须是唯一的。

Globs：用于获取文件列表，可以包含普通的Unix通配符\*，如"\*.java"，还可以包含单个\*\*通配符作为路径元素，它将匹配零个或多个路径元素。

变量：Android.bp文件可以包含顶级变量并赋值。变量的范围被限定为它们声明的文件的剩余部分，以及任何子 blueprint 文件。一个例外是变量不可变，能够被 += 进行附加赋值，而且只能在被引用之前。

    gzip_srcs = ["src/test/minigzip.c"],
     
    cc_binary {
        name: "gzip",
        srcs: gzip_srcs,
    }

注释：Android.bp文件能包含C风格的多行/\* \*/注释和C++风格的单行注释//。

类型：变量和属性是强类型的，基于第一个赋值动态变量，以及模块类型静态属性。他们支持的类型有：布尔型（bool) 整型（int）字符串（string）字符串列表（"string1", "string2"）Maps ({key1: "value1", key2: \["value2"]})。

Map 可以包含任意类型的值，包括嵌套的maps。列表和 maps 允许在最后一个值之后有逗号。

操作符：字符串、字符串列表、和maps能使用+操作符进行附加。整型可以用+操作符来总结。附加的maps将生成两个map中键的并集，并附加的两个map中存在任何键的值。

缺省模块：缺省模块可用于在多个模块中重复相同的属性。

    cc_defaults {
        name: "gzip_defaults",
    }
     
    cc_binary {
        name: "gzip",
        defaults: ["gzip_defaults"],
    }

名称解析：Soong为不同目录中的模块提供了指定相同名称的能力。只要每个模块在一个单独的名称空间中声明。 命名空间可以这样声明：

    soong_namespace {
        imports: ["path/to/otherNamespace1", "path/to/otherNamespace2"],
    }

每个Soong模块都会根据其在树中的位置分配一个名称空间，除非找不到soong\_namespace模块，否则每个Soong模块都被认为位于当前目录或最接近的父目录中的Android.bp中的soong\_namespace所定义的名称空间中。在这种情况下，该模块被认为处于隐式根目录命名空间。

格式化：Soong 包含了一个 blueprint 文件的格式化器，类似于 gofmt。使用以下命令来递归格式化当前目录中的所有 Android.bp 文件：

    bpfmt -w .

标准格式包括 4 个空格的缩进，包含多个元素的列表中，每个元素之后的换行符，并且始终包括列表和 maps中的逗号。

### 3. mk与bp对照

参考：[android.go](https://android.googlesource.com/platform/build/soong/+/a930003/androidmk/cmd/androidmk/android.go)

### 4. go语言

声明变量：var identifier type ，简短形式使用 := 赋值操作符

声明常量：const identifier \[type] = value

声明数组：var variable\_name \[SIZE] variable\_type

指针：var var\_name \*var-type

type关键字：定义结构体，定义接口，类型定义，类型别名，类型查询

结构体：结构体是由一系列具有相同类型或不同类型的数据构成的数据集合。也可以不包含任何字段，称为空结构体。

    type struct_variable_type struct {
       member definition;
       member definition;
       ...
       member definition;
    }

接口：把所有的具有共性的方法定义在一起，任何其他类型只要实现了这些方法就是实现了这个接口。也可以不带任何方法，成为空接口。

    type interface_name interface {
       method_name1 [return_type]
       method_name2 [return_type]
       method_name3 [return_type]
       ...
       method_namen [return_type]
    }

切片(Slice)：是对数组的抽象，实现“动态数组”。append()用来将元素添加到切片末尾并返回结果。

范围(Range)：用于 for 循环中迭代数组(array)、切片(slice)、通道(channel)或集合(map)的元素。在数组和切片中它返回元素的索引和索引对应的值，在集合中返回 key-value 对的 key 值。

Map(集合)：可以使用内建函数 make 也可以使用 map 关键字来定义 Map

    /* 声明变量，默认 map 是 nil */
    var map_variable map[key_data_type]value_data_type
     
    /* 使用 make 函数 */
    map_variable := make(map[key_data_type]value_data_type)

函数定义：不属于任何结构体、类型的方法。

    func function_name([parameter list]) [return_types] {
       函数体
    }

方法定义：与函数区别的是，方法在定义的时候，会在func和方法名之间增加一个参数，这个参数就是接收者，这样我们定义的这个方法就和接收者绑定在了一起，称之为这个接收者的方法。

init()函数：没有输入参数和返回值，用于包(package)的初始化，先于main()函数执行，不被其他函数调用，主要用于程序运行前的注册。

### 5. 添加逻辑控制

1.  在bp中添加bluetoothext\_defaults模块并指定go文件：

    bootstrap_go_package {
        name: "soong-xxx",
        pkgPath: "android/soong/xxx",
        deps: [
        ],
        srcs: [
            "bluetoothext.go",
        ],
        pluginFor: ["soong_build"],
    }
    
    // 定义模块
    bluetoothext_defaults {
        name: "bluetoothext_defaults",
    }

这里需要注意的是，bluetoothext\_defaults是自定义的模块，bluetoothext.go配置这个模块包含了哪些内容，编译时需要在Android.bp中引用这个模块。

2.编写go文件，定制bluetoothext\_defaults模块的编译内容：

    package bluetoothext
     
    import (
        "android/soong/android"
        "android/soong/cc"
    )
     
    func init() {
        // bluetoothext_defaults在Android.bp中定义
        android.RegisterModuleType("bluetoothext_defaults", bluetoothextDefaultsFactory)
    }
     
    ...
     
    func bluetoothextDefaultsFactory() android.Module {
        module := cc.DefaultsFactory()
        android.AddLoadHook(module, bluetoothextDefaults)
     
        return module
    }
     
    func bluetoothextDefaults(ctx android.LoadHookContext) {
        type props struct {
        //参考mk与bp对照
        }
     
        p := &props{}
        //定制模块内容
     
        ctx.AppendProperties(p)
    }

3.引用自定义的bluetoothext\_defaults模块：

    // 编译模块中添加定义的模块
    cc_library_shared {
        name: "libbluetooth_jni",
        defaults: ["bluetoothext_defaults"]
        ...
    }

### 6. config常用方法

参考：[config.go](https://android.googlesource.com/platform/build/soong/+/master/android/config.go)

    // 获取编译常量
    func (c *config) Getenv(key string) string {}
    func (c *config) GetenvWithDefault(key string, defaultValue string) string {}
     
    // 判断常量值是否为true
    func (c *config) IsEnvTrue(key string) bool {}
     
    // 判断常量值是否为false
    func (c *config) IsEnvFalse(key string) bool {}
     
    // 获取机器名
    func (c *config) DeviceName() string {}
     
    // 获取Android版本名
    func (c *config) PlatformVersionName() string {}
     
    // 获取Android版本int值
    func (c *config) PlatformSdkVersionInt() int {}
     
    // 获取Android版本String值
    func (c *config) PlatformSdkVersion() string {}

### 参考资料

[soong](https://android.googlesource.com/platform/build/soong)  
[Android soong 代码分析](https://www.jianshu.com/p/699cfa0f8a74)  
[Android中的Ninja简介](https://note.qidong.name/2017/08/android-ninja/)  
[Android编译系统中的Android.bp、Blueprint与Soong](https://note.qidong.name/2017/08/android-blueprint/)  