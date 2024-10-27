+++
draft = false
date = 2024-09-26T15:00:07+08:00
title = "Windows 版本特性史"
description = ""
slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++

最近突发兴致，想系统性研究一下操作系统老大哥微软，本来想去考古微软的历史，但想想工作量太大了，先从Windows的历史开始吧。

参考了很多内容 [参考链接](##参考链接) ，跟这些文章/资料不同的是，本文不关注非桌面版、更关注产品功能和技术点。

文章会持续更新，欢迎评论补充！



## Windows 98

- 界面方面也进行了改进
    - 任务栏上点击最小化窗口的功能
    - 新的快速启动工具栏，可将某些快捷方式固定到任务栏
    - Windows 资源管理器（Windows Explorer）添加了后退和前进按钮以及地址栏
    - 多显示器支持
- Web TV的支持
- 文件系统
    - 对FAT32文件系统的支持
- 融入了更多基于 Web 的元素
    - 附带的 Internet Explorer，还有应用程序如 Outlook Express、FrontPage Express 和 Microsoft Chat
- 引入了 Windows 驱动程序模型（WDM）来帮助硬件支持

## Windows 2000 & Windows Me

- 辅助功能
    - 如屏幕键盘和朗读器
- 多种语言支持
- 计算机管理控制台
    - 包括磁盘管理、设备管理器和磁盘碎片整理等工具
- Windows Shell
    - 支持透明度和阴影等特效
    - 任务栏增加了气泡通知，以吸引用户的注意力
    - Windows 资源管理器获得了可自定义的工具栏和自动完成支持
- 文件系统
    - 增加了对 NTFS 3.0 的支持
    - 文件系统加密
- 多媒体（Me版本）
    - Windows Movie Maker 视频编辑器
    -  Windows Media Player 和 Windows DVD Player
- 系统工具（Me版本）
    - 新增了系统还原、自动更新
    - 屏幕键盘支持

## Windows XP

- 重新设计的用户界面 “Luna”
- 任务栏
    - 支持任务分组
- 支持远程桌面（最开始在Windows Server中引入）

## Windows Vista

- 引入 Windows Aero 新界面设计
    - 系统主题支持
    - 桌面小工具支持
- 系统安全
    - 用户账户控制（UAC）
    - Windows Defender
    - 防火墙
    - 硬盘加密 BitLocker
- 升级系统应用
    - Windows Media Player 11
    - Internet Explorer 7
    - Windows Search
    - Windows Mail
- 多点触控支持

> 因为过多的性能消耗，以及UAC导致的过多安全提示而饱受诟病，用户满意度极差。
> hualet: 不过从技术的角度看，Vista确实是改进非常大的版本，尤其是在安全方面

## Windows 7

- 优化了 Windows Aero 的效果和性能
    - 优化了触屏支持和手写识别效果
    - Aero Snap: 拖动窗口触碰屏幕边缘触发最大化
    - Aero Peek: 显示桌面
- 性能优化
    - 改善多核心（Multi-Core）处理器的运行效率
    - 提升开机速度
- 内核
    - 支持虚拟硬盘（VHD）
    - 支持 SSD
- 首次预装 XPS & Windows PowerShell
- 控制台
    - ClearType文字調整工具
    - 顯示器色彩校正精靈
    - 桌面小工具（Desktop Gadget）
    - 系統還原（System Restore）
    - 疑難排解
    - 工作空間中心（Workspaces Center）
    - 認證管理員
    - 系統圖示和顯示
- Windows XP 模式：支持通过虚拟化技术调用虚拟机中的 XP，实现软件的兼容 （有点像彩虹）

## Windows 8

- 更适合触摸的界面（基于磁贴和 Metro 的用户界面）
- 开始菜单被替换为开始屏幕
- Windows 商店
- 首次集成了 OneDrive

> 由于新的用户界面非常难以学习，对大多数 PC 用户来说成为了一个不受欢迎的更新。

## Windows 10

- 首次引入多任务视图+虚拟桌面
- 首次引入 UWP
- 生物认证支持：指纹、人脸、虹膜

## Windows 11

- 界面功能
    - 平板模式融合
    - 全新的Auto HDR功能
    - 交互式锁屏天气：锁定屏幕上更丰富的天气体验，包括动态的交互式天气更新
- 安卓应用支持
- 预装 Teams 应用
- 安全增强：需要在支持安全启动（UEFI Secure Boot）与TPM 2.0的计算机上才能启动与运行。
- AI 大模型
    - Copilot
    - AI 应用增强，包括画图、截图

- Windows 11 23H2 [系统之家新闻，看起来是个质量版本](https://www.xitongzhijia.net/win11/234531.html)

- Windows 11 24H2  [系统之家新闻](https://www.xitongzhijia.net/win11/299294.html)
    - Wi-Fi7支持：提升连接体验。
    - 辅助助听设备支持的蓝牙LE音频增强功能。
    - 系统托盘&任务栏功能增强功能。（）
    - 更简化文件资源管理器。
    - 适用于电脑的智能电源管理。
    - 使用 QR 码加入和共享Wi-Fi网络。
    - 用于 Wi-Fi 网络访问的增强隐私控制。
    - Microsoft TEAMS 中的轻松账户管理和通知。
    - 跨设备扩展语音Clarity（一个网站优化工具，不是重点，微软怎么啥玩意儿都做……）。



## 参考链接

- [Microsoft Windows的历史](https://zh.wikipedia.org/zh-cn/Microsoft_Windows%E7%9A%84%E5%8E%86%E5%8F%B2)
- [Windows 版本历史回顾：从 1.0 到 11 的惊艳历程](https://www.sysgeek.cn/windows-version-history/)
- [Windows系统历史版本简介](https://blog.csdn.net/qq_44794321/article/details/127279796)