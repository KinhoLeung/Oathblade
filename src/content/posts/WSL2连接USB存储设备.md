---
title: WSL2连接USB存储设备
published: 2024-08-19
description: '开发环境搭建'
image: ''
tags: [WSL2, USB]
category: 'Blog'
draft: false 
---

最近在使用WSL2作为我的开发环境，不过在我尝试将bin文件烧录到SD卡（这里我是通过USB读卡器连接到电脑上）上时，发现SD没有被挂载上，去网上找了下原因。发现主要是以下2个原因导致的：

* **WSL2本质是一个虚拟机，无法直接访问宿主主机的USB设备。**
* **WSL2的内核没有加入USB存储设备的驱动。**

> WSL2：WSL2本质上是一个虚拟化技术，它在Windows操作系统上创建了一个轻量级的虚拟机，其中运行了一个完整的Linux内核。这个Linux内核与Windows内核相互隔离，但可以通过WSL2与Windows系统进行通信和交互。WSL2使用了Linux内核虚拟机（VM）技术，通过Hyper-V虚拟化平台来实现，在Windows系统上运行Linux应用程序时提供了更好的性能和兼容性。虽然WSL2在本质上是一个虚拟化技术，但它与传统的虚拟机不同，它并不是运行完整的Linux发行版，它更加轻量级且无需额外的资源分配，提供了一个与Linux兼容的运行时环境。用户可以在WSL2中安装并运行各种Linux发行版，如Ubuntu、Debian、Fedora等。WSL2和Linux发行版之间的关系是，WSL2提供了一个运行Linux发行版的虚拟化环境，用户可以在WSL2中安装和运行各种Linux发行版，以获得Linux环境和运行Linux应用程序的能力。
>

------

**知道了原因就好办了，先来解决第一个问题：让WSL2连接USB设备。这里用到了usbipd-win这个项目。**

> usbipd-win：USB/IP项目旨在开发一种通用的通过IP网络共享USB设备的系统。为了让计算机之间能够共享USB设备并完全发挥它们的功能，USB/IP将“USB I/O消息”封装到TCP/IP数据包中，并在计算机之间传输它们。说白了就是把USB封装在TCP中，通过网络传送。
>

先在windows中打开powershell使用以下命令来安装usbipd工具：

```
winget install --interactive --exact dorssel.usbipd-win
```

下载完后点击安装就好，安装完后还会要求你重启电脑。

![15C1C188-1974-483b-8192-3CE70740A5C5](https://kinho-image.oss-cn-shenzhen.aliyuncs.com/image/15C1C188-1974-483b-8192-3CE70740A5C5.png)

![A840BABB-F957-47d6-8663-E80DFE4C8B55](https://kinho-image.oss-cn-shenzhen.aliyuncs.com/image/A840BABB-F957-47d6-8663-E80DFE4C8B55.png)

再到WSL2中使用使用以下命令来安装usbipd工具：

```
sudo apt install linux-tools-generic hwdata
```

```
sudo update-alternatives --install /usr/local/bin/usbip usbip /usr/lib/linux-tools/*-generic/usbip 20
```

![D18EBF35-3667-41fd-85AD-A0896A14C6EC](https://kinho-image.oss-cn-shenzhen.aliyuncs.com/image/D18EBF35-3667-41fd-85AD-A0896A14C6EC.png)

到这里第一个问题就解决了，别忘了重启下电脑。

**然后再来解决第二个问题：重新编译WSL2的内核，加入 USB 存储设备支持。**

首先我们需要到[WSL2的内核仓库](https://github.com/microsoft/WSL2-Linux-Kernel)中clone一份内核源码：

```
git clone https://github.com/microsoft/WSL2-Linux-Kernel.git
```

再把源码解压出来：

```
unzip WSL2-Linux-Kernel-linux-msft-wsl-5.15.y
```

![image-20231013180736390](https://kinho-image.oss-cn-shenzhen.aliyuncs.com/image/image-20231013180736390.png)

然后再安装libncurses-dev：

```
sudo apt install libncurses-dev
```

> libncurses-dev：libncurses-dev是一个开发库，用于在Linux系统上开发基于终端的用户界面（TUI）应用程序。它是ncurses库的开发版本，提供了编译和链接TUI应用程序所需的头文件和静态库文件。使用libncurses-dev，开发人员可以利用ncurses库的功能创建具有交互性和可视化效果的终端应用程序。
>

使用以下命令来编辑内核配置文件：

```
make menuconfig KCONFIG_CONFIG=Microsoft/config-wsl
```

进入 `Device Drivers` -> `USB support` -> `Support for Host-side USB` ，选中 `USB Mass Storage support`（ `*` 号是直接编译进内核，`M` 是编译为内核模块，内核模块需要手动加载），把下面弹出来的一堆驱动都选上，保存完之后就可以退出了。

![5B2688B6-6ED1-4527-959D-4954D9179CF3](https://kinho-image.oss-cn-shenzhen.aliyuncs.com/image/5B2688B6-6ED1-4527-959D-4954D9179CF3.png)

在开始编译之前还需要安装`build-essential`、`flex`、`bison`、`libssl-dev`、`libelf-dev`、`dwarves`，这些工具是编译内核所需的常见工具和库。

```bash
sudo apt install build-essential flex bison libssl-dev libelf-dev dwarves
```

![AC3394FD-F04C-4fc5-8E1A-7805B9E03462](https://kinho-image.oss-cn-shenzhen.aliyuncs.com/image/AC3394FD-F04C-4fc5-8E1A-7805B9E03462.png)

这里我已经安装过了就不重新安装了，接下来就可以编译内核了，进入源码目录WSL2-Linux-Kernel-linux-msft-wsl-5.15.y，用下面这个命令将刚才的那些驱动全部编进内核中：

```
cd WSL2-Linux-Kernel-linux-msft-wsl-5.15.y
```

```
make -j$(nproc) bzImage KCONFIG_CONFIG=Microsoft/config-wsl
```
编译完成的内核是bzImage文件，和我们编译时指定的名称一致，文件在arch/x86/boot/文件夹下。

![image-20231013182814747](https://kinho-image.oss-cn-shenzhen.aliyuncs.com/image/image-20231013182814747.png)

> 大家不嫌弃的话可以使用我编译好的这个：
>
> 链接：https://pan.baidu.com/s/1dfBIGh_a2nbN9QiXUPQtXg?pwd=WSL2 
> 提取码：WSL2

把编译好的内核复制出来，放在 Windows 的用户目录（默认是 `C:\Users\{username}`）下创建一个名为 `.wslconfig` 的文件，内容根据 [微软官方文档](https://docs.microsoft.com/zh-cn/windows/wsl/wsl-config#options-for-wslconfig) 来：

```
[wsl2]
kernel=path\\to\\kernel
```

以我的为例：

![9070468D-3F61-4cb3-BD14-B947B9317D5B](https://kinho-image.oss-cn-shenzhen.aliyuncs.com/image/9070468D-3F61-4cb3-BD14-B947B9317D5B.png)

到这里第二个问题也解决完了。

**最后就是验证成果了**

打开powershell使用以下命令来查看内核版本号。

```
uname -r
```

![0944BD45-E23A-4b9f-BAC8-A96E95CF021C](https://kinho-image.oss-cn-shenzhen.aliyuncs.com/image/0944BD45-E23A-4b9f-BAC8-A96E95CF021C.png)

可以看到这是更换内核前的版本号是5.15.90.1，然后使用下面这个命令来关闭WSL2，再重新打开WSL2。

```
wsl --shutdown
```

![97AAF45F-B70A-4aa9-A0F0-38D9A7495B68](https://kinho-image.oss-cn-shenzhen.aliyuncs.com/image/97AAF45F-B70A-4aa9-A0F0-38D9A7495B68.png)

可以看到版本已经变成5.15.133.1了，说明内核更换成功，接下来使用usbipd-win来连接，usbipd-win的使用方法参考也是参考的[微软官方文档](https://learn.microsoft.com/zh-cn/windows/wsl/connect-usb)，首先打开powershell，使用下面的第一个命令列出所有连接到 Windows 的 USB 设备，找到USB大容量存储设备对应的BUSID，然后使用这个BUSID替换下面第二个命令中的<busid>来使USB设备连接到WSL2，以我的为例：

```
usbipd wsl list
```

```
usbipd wsl attach --busid <busid>
```

![6C1F5365-AAF2-4740-A840-3B6E7DC1A73E](https://kinho-image.oss-cn-shenzhen.aliyuncs.com/image/6C1F5365-AAF2-4740-A840-3B6E7DC1A73E.png)

最后到WSL2中使用下面使用下面这2个命令来查看USB设备和SD卡的挂载情况：

```
lsusb
```

```
ls /dev/sd*
```

![9AB1E88D-E644-4d4f-9187-9376A13B3462](https://kinho-image.oss-cn-shenzhen.aliyuncs.com/image/9AB1E88D-E644-4d4f-9187-9376A13B3462.png)

可以看到WSL2已经成功地连接上了USB设备，SD卡也挂载成功✌️。

PS：有哪个地方写得不对的，欢迎各位指正。

------

**参考连接**

[在WSL2中连接USB设备](https://www.littleqiu.net/access-usb-storage-in-wsl2/)

[HyperV和WSL2的USB直通](https://yadom.in/archives/usb-passthrough-hyper-v-and-wsl2.html)

[WSL2嵌入式开发随笔（2）——使用自己编译的WSL2系统内核](https://zhuanlan.zhihu.com/p/609431551)

[wsl系列内容：WSL2编译和使用自定义内核的方法](https://blog.csdn.net/chubbykkk/article/details/125216332)

[WSL support](https://github.com/dorssel/usbipd-win/wiki/WSL-support#usbip-client-tools)

[连接USB设备](https://learn.microsoft.com/zh-cn/windows/wsl/connect-usb)
