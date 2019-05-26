---
title: "deepin系统启动流程"
date: 2019-05-25T16:54:55+08:00
---

deepin系统整个的启动流程到底是怎么样子的？以前曾被同事缠问过类似的问题。遇到这种宏大而又不着边际的问题，我的回复往往是“你还太嫩，现在我告诉你还是会忘掉的，等你干上个两年，不用我说你就知道了”。我边敲着键盘，边佩服自己的聪明才智。

<center>两年后……</center>

这个小伙子长大了，并且坚定地又问了同样的问题。

我一愣神，脑海中不停浮现出一个声音“出来混，迟早是要还的”。想想也罢，是时候该把压箱底儿的货拿出来了，毕竟自封了“半吊子系统工程师”的title（虽然我自封的title还有很多：半吊子客服、半吊子产品经理、半吊子研发项目经理等等），不给你们露两手看看看来还真不行……



## 概览

deepin系统启动，从整体上看主要分为了硬件上电、内核引导、内核启动、系统初始化、图形界面等几个阶段。如果将这几个阶段分为两个部分，那么第一部分的硬件上电、内核引导、内核启动主要是“引导（boot）”，更偏向让内核可以启动；而第二部分的系统初始化、图形界面两个阶段主要的任务则是“初始化（initialize）”了，因为对于一个系统来说仅仅有内核跑起来是不行的，还要有各种各样的服务对系统的软硬件进行管理，这也是平常大家说发行版跟纯粹的GNU/Linux内核不是一个概念的原因之一。



下面我从一个软件开发者的角度说一下我对每个阶段的理解以及一些调试的方法。



## 硬件上电

既然说了是软件开发者的角度，这个部分对我来说基本上相当于黑盒子了。但是大体上我们仍然知道这个部分主要是：

1. 硬件上电
2. `BIOS`/`UEFI`
3. `bootloader`

当你按下电源的那一刻起，电流就会“滋滋滋”的流向主板，启动`BIOS（Basic Input Output System）`系统。`BIOS`系统，顾名思义就是最直接跟硬件打交道的系统，因为有标准规定，所以输入输出设备的基本功能都是可以使用的，一些硬件的开关配置也可以在`BIOS`中进行操作。除此之外，`BIOS`还有两个重要的功能，一个是硬件自检；另外一个是加载引导。硬件自检这个跟作为一个”半吊子系统工程师“没什么关系，自不多说。加载引导的过程其实就是大家耳熟能详的`MBR`、小蝌蚪找妈…哦不…`MBR`中找`bootloader`了。

跟`BIOS`对应的`UEFI`，要说它们之间的区别，除了加载引导的方式不大一样以外。对我来说可能就是界面能够用鼠标点点点了吧，嗯……哈哈哈。

这里讲个段子，之前15.7搞启动时间优化的时候，测试的同学测试系统启动时间的优化情况，老是说效果不理想，我去看看吧，原来是他们测试系统启动是从硬件启动算起的，我说你们直接从内核引导开始计算，他还问我为什么。优化时间/(`BIOS`时间+`GRUB`时间+内核时间+图形时间) 跟 优化时间/(内核时间+图形时间)哪个大哪个小？我只能说这个测试同学的数学不大好……😂



## 内核引导 

`BIOS`在`MBR`中（或者`UEFI`在主板专有的存储设备中）找到`bootloader`并加载后，`bootloader`就会开始加载Linux内核并启动了。

### GRUB引导

deepin系统默认的`bootloader`是`GRUB（GRand Unified Bootloader）`。其实我一直觉得这个名字挺恶心的，大神们果然都是重口味……`GRUB`并不需要按照什么规则去硬盘中找系统，而是根据`/boot/grub/grub.cfg`中的启动项加载内核、启动系统，而这个配置文件则是在系统安装或者手动执行`update-grub`这个命令的时候生成和更新。

`update-grub`这个命令其实是对`grub-mkconfig`的一个包装，在非Debian系的发行版上是没有的。`grub-mkconfig`会执行的动作主要是：

1. 加载`/etc/default/grub`中的一些配置项。比如`GRUB_CMDLINE_LINUX_DEFAULT`配置项会控制Linux的[boot param](https://wiki.archlinux.org/index.php/kernel_parameters)。
2. 挨个执行`/etc/grub.d/`目录中的脚本，用来生成最终的`grub.cfg`文件。比如我们平常看到`update-grub`命令执行时输出的哪些启动项，其实就是`/etc/grub.d/03_os-prober`这个脚本里面执行`os-prober`这个工具产生的。

在`GRUB`界面选择启动项，按`e`编辑启动项。除了普通的上下左右键移动光标，还可以使用基本的`Emacs`快捷键：

- `Ctrl+N` 下一行
- `Ctrl+P` 上一行
- `Ctrl+B` 左移一个字符
- `Ctrl+F` 右移一个字符
- `Ctrl+A` 移动光标到行首
- `Ctrl+E` 移动光标到行尾

编辑完成后按`Ctrl+X`按照编辑后的结果启动系统，但是编辑的结果不会保存，也就是说如果需要永久修改某个启动项，就要修改`grub.cfg`文件或者会影响`grub.cfg`生成的`/etc/default/grub`以及`/etc/grub.d/`中的脚本文件了。

对于`GRUB`，我们一般需要知道的就这么多，关于`GRUB`其他一些用法和知识，可以参考[GRUB与系统引导](https://blog.nanpuyue.com/2017/037.html)这篇文章。

### UEFI直接引导

在`UEFI`模式下，除了使用`GRUB`来引导内核以外，还可以通过`UEFI`直接引导内核（需要内核开启了` EFI Stub`支持），具体的配置方式见[Debian Wiki EFI Stub](https://wiki.debian.org/EFIStub)。需要注意的一点是在使用`efibootmgr`创建启动项的时候，可能需要`-d`参数指定设备，否则可能会导致创建启动项失败。



##  内核启动

内核启动部分其实主要是想说`initrd`。

`initrd`是一个小型的`rootfs`，这个`rootfs`保证了内核启动过程中所需要的内核模块和用户态工具。同时，它还需要为下一个阶段“系统初始化”做准备，也就是为init程序准备好真正的文件系统，并且启动`init`程序。

内核使用`initrd`启动的过程主要是：

1. 执行`init`脚本（这个不是上面说得`init`程序，而是生成好的`initramfs`中的`/init`这个文件。后面的步骤其实都发生在这个脚本的执行过程中）；
2. 解析内核启动参数，识别关键的如`debug`、`boot`、`break`等；
3. 执行`/scripts/init-top/`中的脚本；
4. 加载内核模块；
5. 执行`/scripts/init-premount/`中的脚本；
6. 执行`/scripts/$BOOT`脚本中的`mountroot`函数，其中的`$BOOT`参数就是第2步中识别到的`boot`参数指定的，自带的可选项有`local`（本地启动，默认）、`nfs`（比如PXE启动）；
7. 执行`/scripts/init-bottom/`中的脚本；
8. 执行`init`程序。



### live系统

上面第6步提到`initramfs-tools`包自带有两种boot类型：`local`和`nfs`，我们使用live系统的时候的boot类型其实`live`。这个boot类型主要是`live-boot`这个包支撑起来的。

在启动live系统的时候，`/scripts/live`中的`mountroot`会调用`/lib/live/boot/`目录中的脚本设置根文件系统，包括挂在ISO和设置overlay等。



### 调试

`initrd`启动阶段支持几个特殊的启动参数辅助调试：

- `debug` 会开启initrd启动脚本中的调试模式；
- `break`  可以讲启动停止在某个阶段，例如`break=premount`就会在真正的根文件系统挂载前停掉启动流程，并给你一个`busybox`环境；可选`break`的阶段可以在`/scripts/init`脚本中看到，就是哪些使用`maybe_break`的行。



##  系统初始化

从这个阶段开始，启动流程就算是进入我们真实的的系统中了，`init`程序会启动各种服务对系统进行配置，保证软硬件环境可以正常使用。

deepin系统下默认的`init`程序是`systemd`，这个庞然大物太过复杂，或许将来开个坑写一个系列才能说好……这里只简单说明一下跟系统启动流程相关的内容。一个是 `man bootup`中的一个图：

```
           local-fs-pre.target
                    |
                    v
           (various mounts and   (various swap   (various cryptsetup
            fsck services...)     devices...)        devices...)       (various low-level   (various low-level
                    |                  |                  |             services: udevd,     API VFS mounts:
                    v                  v                  v             tmpfiles, random     mqueue, configfs,
             local-fs.target      swap.target     cryptsetup.target    seed, sysctl, ...)      debugfs, ...)
                    |                  |                  |                    |                    |
                    \__________________|_________________ | ___________________|____________________/
                                                         \|/
                                                          v
                                                   sysinit.target
                                                          |
                     ____________________________________/|\________________________________________
                    /                  |                  |                    |                    \
                    |                  |                  |                    |                    |
                    v                  v                  |                    v                    v
                (various           (various               |                (various          rescue.service
               timers...)          paths...)              |               sockets...)               |
                    |                  |                  |                    |                    v
                    v                  v                  |                    v              rescue.target
              timers.target      paths.target             |             sockets.target
                    |                  |                  |                    |
                    v                  \_________________ | ___________________/
                                                         \|/
                                                          v
                                                    basic.target
                                                          |
                     ____________________________________/|                                 emergency.service
                    /                  |                  |                                         |
                    |                  |                  |                                         v
                    v                  v                  v                                 emergency.target
                display-        (various system    (various system
            manager.service         services           services)
                    |             required for            |
                    |            graphical UIs)           v
                    |                  |           multi-user.target
                    |                  |                  |
                    \_________________ | _________________/
                                      \|/
                                       v
                             graphical.target
```
基本涵盖了由`systemd`接管系统以后，`systemd`所管理的服务之间的一个依赖和基本先后顺序。

另外一个是`systemd-analyze`命令，通过`systemd-analyze plog > bootup.svg`可以更加直观的看到`systemd`各个服务在启动过程中启动的时间和所存在的时间。



## 图形界面

图形界面阶段是系统到达图形环境后的过程，这个部分主要包括：

1. `display-manager.service` 启动`lightdm`；
2. `lightdm`启动`lightdm-deepin-greeter`；
3. 用户输入密码后进入图形会话；
4. `lightdm`执行`/usr/sbin/deepin-session`；
5. `/usr/sbin/deepin-session`执行`/etc/X11/Xsession.d`中的脚本，并最终启动`/usr/bin/startdde`；
6. `startdde`启动桌面环境的组件，系统启动完成。



## 结尾

哦，对了，文章开头的故事纯属虚构，请勿对号入座 :P
