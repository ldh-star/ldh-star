<div align=center>
    <h1>Android执行Shell命令</h1>
</div>

本文介绍了三种安卓执行 shell 命令的方式：

* root 下执行
* adb shell
* 利用 Shizuku

# Shell大致介绍

## 为何能执行shell？

Android操作系统是基于Linux内核构建的，在Android上也可以使用标准的shell工具，这意味着Android设备也可以执行许多与Linux相同的Shell命令。

当一个安卓应用程序执行shell命令时，系统会新启动一个独立的shell进程，该进程会运行指定的shell解释器（通常是 /system/bin/sh）

在这个过程中，进程间通信（IPC）是通过标准输入（stdin）、标准输出（stdout）和标准错误（stderr）流进行的。这些流提供了父进程（应用进程）与子进程（Shell进程）之间通信的途径，这种通信依赖的是传统的UNIX管道和流机制。

## 可以执行哪些shell命令？

Android设备基于Linux内核，每个应用程序在系统中都有一个唯一的用户ID（UID）。这个UID用于标识应用程序，并限制其权限和访问范围。

没有root权限的普通应用受到Android权限模型和SELinux策略的严格限制。SELinux是一种强制访问控制（MAC）机制，它定义了哪些进程可以执行哪些操作。普通应用只能执行受限范围内的命令，这些命令通常不会涉及到系统的核心部分或敏感数据。

* 普通应用：只能执行基本的文件操作、系统信息查询和非特权的网络和进程命令，受限于用户权限和SELinux策略。
* root应用：具有超级用户权限，可以执行几乎所有命令，包括系统管理、用户和权限管理、高级文件操作、网络和进程管理以及设备和硬件管理命令

# 执行Shell实践

上文中提到，普通应用只有一些基本的普通命令的权限。

然而该特性无法满足很多对系统进行高权限操作的场景（比如自动化测试脚本需要模拟对屏幕进行点击等等）。

本文总结了三种办法来解决这个问题，让安卓机也能执行一些高权限的shell命令：

* 设备Root
* 使用 adb shell 运行 .sh 脚本文件
* 接入Shizuku

## 设备Root

设备root之后，可以为指定应用授予root权限，root用户拥有Linux级别最高权限。

### 设备如何Root

目前主流的方式有如下几种：

1. 开虚拟机或者模拟器
    1. 一些模拟器可以直接开启root权限管理器
    2. AndroidStudio官方模拟器也可以刷入下面的方案
2. Magisk  https://magiskcn.com/
    1. 对boot镜像修补，实现了自己的su二进制文件
    2. 官网有不同安卓品牌设备安装Magisk的方式
    3. 还有个人开发者的分支MagiskDelta，功能更强大
3. KernelSU  https://kernelsu.org/
    1. 对Linux内核打补丁，高效管理root，难以被检测
    2. 支持通用内核镜像和可加载内核镜像两种修补模式

### 一般的Root流程

#### 1、解除Bootloader锁

目前主流的方式都是对系统boot镜像（有些设备是init\_boot）进行补丁后刷入系统。

但是许多厂商不允许用户随意对系统进行刷机修改，Bootloader都是锁定的。

仅部分商家的安卓设备允许解锁，并且还带有限制条件：

| 厂商     | 系统       | 是否支持解锁 |
| ---------- | ------------ | -------------- |
| 小米     | MIUI       | ✅✅ 门槛低  |
| HyperOS  | ✅ 门槛高  |
| 三星     | OneUI      | ✅✅✅ 无门槛 |
| vivo     | FuntouchOS | ❌ |
| OriginOS | ❌         |
| Google   | Pixel      | ✅✅✅ 无门槛 |
| Oppo     | ColorOS    | ✅ 门槛高 |
| 一加     | ColorOS    | ✅ 门槛高 |

#### 2、修补系统镜像

具体规则参考Magisk和KernelSU官网教程。

[安装 | KernelSU](https://kernelsu.org/zh_CN/guide/installation.html)

[Magisk安装教程](https://magiskcn.com/)

#### 3、刷入修补后的镜像

adb reboot bootloader

fastboot flash boot 修补后的镜像.img

fastboot flash init\_boot 修补后的镜像.img

### 总结

好处：通过刷机，可以对系统进行修改，能够授予第三方应用root权限，启动的 shell 进程可以以 root 用户的身份运行，具有完全的系统权限。

坏处：由于大部分厂商都会对Bootloader锁进行限制，解锁门槛高，并且对于一些野系统，很难找到官方原版的系统镜像文件，因此root方案是门槛非常高的，并且只支持部分厂商的机型。

## 使用 adb shell 运行 .sh 脚本文件

在安卓中，adb shell 命令启动的进程能够以 shell 用户的身份运行，这比普通用户拥有更多的权限，不过还是不如root用户拥有所有权限。

因此，将 shell 命令集写入脚本文件，再通过 adb shell 来启动后台执行，也可以实现自动化脱机运行脚本，参考：

[shell脚本自动化测试\_am start -s-CSDN博客](https://blog.csdn.net/qq_40148262/article/details/131190445)

好处：对于一些简单的脚本，可以用 adb shell 很方便的实现。无需 root，拥有adb shell 级别的权限。

坏处：无法适应复杂的脚本场景

## 接入Shizuku

### Shizuku介绍

Github：https://github.com/RikkaApps/Shizuku

通过使用 ADB 接口，提供一种在不获取 root 权限的情况下，执行某些需要高权限操作的方法。它通过运行在 ADB shell 环境中的服务进程来提供这些功能。

### Shizuku工作原理

#### 1、启动高权限服务进程

首先在设备上安装Shizuku应用，然后通过adb启动Shizuku服务：

```Shell
adb shell sh /sdcard/Android/data/moe.shizuku.privileged.api/start.sh
```

该脚本会创建一个守护进程，并执行字节码文件，开启loop循环在后台运行。

这个服务进程在 adb shell 环境中运行，具有较高的权限。

#### 2、在第三方应用中接入ShizukuAPI

在自己开发的应用中接入ShizukuAPI，运行时授予该应用的Shizuku权限。

具体接入规则见开发文档：https://github.com/RikkaApps/Shizuku-API

#### 3、通过Binder与Shizuku服务进程通信

ShizukuAPI提供了一系列的高权限接口，当第三方应用被授权后可以调用这些接口。

比如，通过ShizukuAPI和Shizuku服务进行IPC通信可以让Shizuku的服务进程执行用户应用指定的shell命令。

Shizuku服务进程会启动具有 adb shell 权限的 shell 进程来执行这些命令。

具体源码分析可以参考：

https://blog.csdn.net/weixin\_45427063/article/details/137238732

[Android Shizuku源码分析](https://moonlab.top/posts/2020/android-shizuku-theory/)

[Android Shizuku源码分析 第二篇](https://moonlab.top/posts/2020/android-shizuku-theory2/)

### Shizuku特点

无需root、具有 adb shell 级别权限、支持复杂的业务场景、接入成本不高

# 总结

本文系统地介绍了在Android平台上执行Shell命令的几种方法及其优缺点：

设备Root虽然提供了最高的权限，但复杂且存在较高的风险；

adb shell方法适用于较简单的脚本操作；

而Shizuku提供了一种无需root权限但仍具有高权限的方法，支持复杂的业务场景，且接入成本不高。通过这些方法，开发者可以在不同的场景下选择合适的解决方案，以实现对Android设备的高权限操作。
