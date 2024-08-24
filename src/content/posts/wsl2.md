好的,我根据建议对文章进行了优化。以下是优化后的版本:

---
title: WSL2连接USB存储设备：完整指南
published: 2024-08-19
description: '本文详细介绍了如何在WSL2环境中连接USB存储设备,解决了WSL2无法直接访问宿主机USB设备和内核缺少USB存储设备驱动的问题。通过使用usbipd-win工具和重新编译WSL2内核,实现了USB设备在WSL2中的成功挂载。'
image: ''
tags: [开发环境, WSL2, USB]
category: '软件开发'
draft: false 
---

## 目录
1. 问题背景
2. 解决方案概述
3. 步骤1: 安装usbipd-win
4. 步骤2: 重新编译WSL2内核
5. 步骤3: 验证和使用
6. 常见问题解答(FAQ)
7. 总结

## 1. 问题背景

最近在WSL2上搭建开发环境时,尝试将bin文件烧录到SD卡(通过USB读卡器连接)上,发现SD卡没有被挂载。经过研究,发现主要有以下两个原因:

- **WSL2本质是一个虚拟机,无法直接访问宿主主机的USB设备。**
- **WSL2的内核没有加入USB存储设备的驱动。**

> WSL2简介: WSL2是Windows Subsystem for Linux 2的缩写,它是一种轻量级的虚拟化技术,在Windows上创建了一个完整的Linux内核环境。它提供了比WSL1更好的性能和兼容性,允许用户在Windows上运行各种Linux发行版和应用程序。

## 2. 解决方案概述

为了解决这些问题,我们需要:
1. 使用usbipd-win工具让WSL2连接USB设备
2. 重新编译WSL2内核,加入USB存储设备支持

> 安全提示: 在进行以下操作前,请确保备份重要数据,特别是在修改系统内核时要格外小心。

## 3. 步骤1: 安装usbipd-win

### 在Windows上安装usbipd-win

1. 打开PowerShell,运行以下命令:
   ```powershell
   winget install --interactive --exact dorssel.usbipd-win
   ```
2. 按照安装向导完成安装,可能需要重启电脑。

### 在WSL2中安装usbipd工具

1. 在WSL2终端中运行:
   ```bash
   sudo apt install linux-tools-generic hwdata
   sudo update-alternatives --install /usr/local/bin/usbip usbip /usr/lib/linux-tools/*-generic/usbip 20
   ```

完成后,重启电脑以确保更改生效。

## 4. 步骤2: 重新编译WSL2内核

### 准备工作

1. 克隆WSL2内核源码:
   ```bash
   git clone https://github.com/microsoft/WSL2-Linux-Kernel.git
   unzip WSL2-Linux-Kernel-linux-msft-wsl-5.15.y
   ```

2. 安装必要的工具:
   ```bash
   sudo apt install libncurses-dev build-essential flex bison libssl-dev libelf-dev dwarves
   ```

### 编辑内核配置

1. 运行以下命令:
   ```bash
   make menuconfig KCONFIG_CONFIG=Microsoft/config-wsl
   ```

2. 在配置界面中:
   - 进入 `Device Drivers` -> `USB support` -> `Support for Host-side USB`
   - 选中 `USB Mass Storage support` (使用 `*` 直接编译进内核)
   - 选择所有相关的USB驱动
   - 保存并退出

### 编译内核

> 注意: 编译过程可能需要30分钟到1小时,取决于您的硬件配置。

1. 进入源码目录并开始编译:
   ```bash
   cd WSL2-Linux-Kernel-linux-msft-wsl-5.15.y
   make -j$(nproc) bzImage KCONFIG_CONFIG=Microsoft/config-wsl
   ```

2. 编译完成后,在 `arch/x86/boot/` 目录下找到 `bzImage` 文件。

### 配置新内核

1. 将编译好的内核复制到Windows用户目录。
2. 在用户目录下创建 `.wslconfig` 文件,内容如下:
   ```
   [wsl2]
   kernel=C:\\Users\\YourUsername\\path\\to\\bzImage
   ```

## 5. 步骤3: 验证和使用

1. 重启WSL2:
   ```powershell
   wsl --shutdown
   ```

2. 检查内核版本:
   ```bash
   uname -r
   ```

3. 使用usbipd-win连接USB设备:
   ```powershell
   usbipd wsl list
   usbipd wsl attach --busid <busid>
   ```

4. 在WSL2中验证:
   ```bash
   lsusb
   ls /dev/sd*
   ```

## 6. 常见问题解答(FAQ)

Q: 编译内核失败怎么办?
A: 确保已安装所有必要的依赖,并检查错误信息。可能需要更新系统或安装额外的开发工具。

Q: USB设备连接后在WSL2中不可见?
A: 确保usbipd-win正确安装并运行,检查Windows防火墙设置是否阻止了连接。

Q: 新内核无法加载?
A: 检查 `.wslconfig` 文件路径是否正确,确保使用双反斜杠 `\\`。

## 7. 总结

通过以上步骤,我们成功解决了WSL2连接USB存储设备的问题。这个过程涉及到了安装usbipd-win工具和重新编译WSL2内核,虽然步骤较多,但能够显著提升WSL2的功能性。

希望这篇教程对你有所帮助。如果你在操作过程中遇到任何问题,或者有其他经验分享,欢迎在评论区留言讨论。

最后更新日期: 2024-08-19
我们会定期更新本文内容,以确保与最新的WSL2版本兼容。

---

参考链接:
- [在WSL2中连接USB设备](https://www.littleqiu.net/access-usb-storage-in-wsl2/)
- [HyperV和WSL2的USB直通](https://yadom.in/archives/usb-passthrough-hyper-v-and-wsl2.html)
- [WSL2嵌入式开发随笔（2）——使用自己编译的WSL2系统内核](https://zhuanlan.zhihu.com/p/609431551)
- [wsl系列内容：WSL2编译和使用自定义内核的方法](https://blog.csdn.net/chubbykkk/article/details/125216332)
- [WSL support](https://github.com/dorssel/usbipd-win/wiki/WSL-support#usbip-client-tools)
- [连接USB设备](https://learn.microsoft.com/zh-cn/windows/wsl/connect-usb)