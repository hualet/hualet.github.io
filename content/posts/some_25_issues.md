+++
draft = false
date = 2025-08-03T14:17:04+08:00
title = "最近25上遇到的几个问题和处理方案"
description = ""
slug = ""
authors = []
tags = ["25", "网络"]
categories = ["技术"]
externalLink = ""
series = []

+++

最近在deepin 25上遇到几个问题，处理方案记录一下，以备未来或者其他人需要。

### 01 待机睡死问题

有一段时间 25 没有预装 tlp ，导致系统功耗升高，有时候电脑发烫严重。发现这个问题以后一方面是通知系统研发进行系统更新、集成，另外一方面我自己也先装上测试。

结果虽然电脑功耗下来了，但是不出意料引入了系统稳定性问题，具体的现象就是偶尔待机睡死，概率还比较高。找了内核的同事帮忙调试了一下，没有很好的调试结果，再加上我经常出差、远程调试也不方便，所以干脆自己做了一些尝试。

```
处理器 : AMD Ryzen 7 6800H with Radeon Graphics (八核 / 十六逻辑处理器)
主板 : LNVNB161216
内存 : 4GB(MT62F1G32D4DR-031 WT  6400 MT/s)/4GB(MT62F1G32D4DR-031 WT  6400 MT/s)/4GB(MT62F1G32D4DR-031 WT  6400 MT/s)/4GB(MT62F1G32D4DR-031 WT  6400 MT/s)
显示适配器 : Rembrandt [Radeon 680M]
音频适配器 : Radeon High Definition Audio Controller [Rembrandt/Strix]/Family 17h/19h/1ah HD Audio Controller
存储设备 : SAMSUNG MZVL2512HCJQ-00BL2 (512 GB)
蓝牙 : Bluetooth Radio
网络适配器 : RTL8852BE PCIe 802.11ax Wireless Network Controller/RTL8111/8168/8211/8411 PCI Express Gigabit Ethernet Controller
鼠标 : M185 compact wireless mouse (Logitech M185 compact wireless mouse)/ELAN0662:00 04F3:317C Mouse (ELAN0662:00 04F3:317C Mouse)/ELAN0662:00 04F3:317C Touchpad (ELAN0662:00 04F3:317C Touchpad)
键盘 : AT Translated Set 2 keyboard (AT Translated Set 2 keyboard)
显示设备 : CSOT T3 LCD Monitor(14.0 英寸 (302x188 mm))
图像设备 : Integrated Camera (Chicony Electronics Integrated Camera)
```

因为明确知道是装了 tlp 导致的，所以把现象报给ChatGPT 以后，很快得到一些可以尝试的方向。

第一次尝试把 `amdgpu` 和 `nvme` 加入到 `RUNTIME_PM_DRIVER_DENYLIST` 中:

```
RUNTIME_PM_DRIVER_DENYLIST="nvme amdgpu mei_me nouveau radeon"
```

试了一下，效果不是特别明显。遂来了个一刀切, 直接把 `RUNTIME_PM` 给关掉了:

```
RUNTIME_PM_ON_AC=off
RUNTIME_PM_ON_BAT=off
```

果然有效果了，基本上没有再出现过待机睡死的问题。 不过这个处理方案太一刀切，个人使用可以、但不适合上主线，未来还得继续跟进。




### 02 网络不稳定问题

网络不稳定最开始的表现是邮箱收发邮件经常失败，最开始我还觉得是我们邮箱客户端有问题、没有重试机制，跟同事反馈问题的过程中，又揣摩了出现问题的场景：一般是在家里、在酒店容易出现问题，结合上面提到tlp导致的一些不稳定问题，问了一下 ChatGPT：

```
某些驱动/网卡还会因省电引入短时 TX/队列停顿，反而被 AP 的 band-steering 频繁“踢/引流”，造成来回跳。
```

遂尝试把WiFi省电方案给禁掉，有几种方式：

1) 在驱动层面禁用

   ```
   # 为特定网卡添加配置，禁用 powersave
   options iwlwifi power_save=0
   ```

2) 在 NetworkManager 层面禁用

   ```
   # /etc/NetworkManager/conf.d/wifi-powersave.conf 中添加或修改
   [connection]
   wifi.powersave = 2     # 2=禁用，3=启用
   ```

3) 在某些特定网络禁用

   ```
   nmcli connection modify WIFI名字 802-11-wireless.powersave 2
   ```

因为我一般倾向于系统策略，所以选择了在家里、公司的网络有多AP的情况给WiFi禁用了powersave。

貌似后面遇到问题确实少了。



### 03 邮箱收发邮件问题

虽然上面网络不稳定的问题解决了，但是邮箱收发邮件还是经常容易出现收不到的问题。

让同事给邮箱加了重试机制，还是不行。

正纳闷呢，突然看见有的工作群在讨论商店访问不稳定可能是因为商店支持 IPv6，25默认走 IPv6 但是 IPv6 链路又不稳导致额 。抱着试一试的态度：


```
➜  ~ getent ahosts exmail.qq.com
240e:97c:2f:1::5f STREAM exmail.qq.com
240e:97c:2f:1::5f DGRAM  
240e:97c:2f:1::5f RAW    
240e:97c:2f:1::87 STREAM 
240e:97c:2f:1::87 DGRAM  
240e:97c:2f:1::87 RAW    
183.2.144.113   STREAM 
183.2.144.113   DGRAM  
183.2.144.113   RAW    
183.47.114.101  STREAM 
183.47.114.101  DGRAM  
183.47.114.101  RAW    
183.2.144.108   STREAM 
183.2.144.108   DGRAM  
183.2.144.108   RAW    
```

果然优先解析到了 IPv6 ，然后查了一下应该是 25 跟随上游策略，默认 IPv6 优先了。

查到两种修改策略：

1.  修改 gai.conf 配置文件

   ```
   # /etc/gai.conf
   precedence ::ffff:0:0/96  100
   ```

2.  内核层面直接禁用 ipv6

   ```
   sudo sysctl -w net.ipv6.conf.default.disable_ipv6=1
   sudo sysctl -w net.ipv6.conf.all.disable_ipv6=1
   ```

我选择了方案一，修改以后再试：


```
➜  ~ getent ahosts exmail.qq.com
183.2.144.113   STREAM exmail.qq.com
183.2.144.113   DGRAM  
183.2.144.113   RAW    
183.2.144.108   STREAM 
183.2.144.108   DGRAM  
183.2.144.108   RAW    
183.47.114.101  STREAM 
183.47.114.101  DGRAM  
183.47.114.101  RAW    
240e:97c:2f:1::5f STREAM 
240e:97c:2f:1::5f DGRAM  
240e:97c:2f:1::5f RAW    
240e:97c:2f:1::87 STREAM 
240e:97c:2f:1::87 DGRAM  
240e:97c:2f:1::87 RAW    
```

默认走到 IPv4 上了，修改完以后果然立竿见影。 邮箱重试机制+默认走 IPv6 ，问题全面解决。

问了下 ChatGPT 为什么 IPv6 不稳定，说了一大堆：

```
主要成因（按命中频率）

ICMPv6 被阻断 → PMTU 黑洞
IPv6 依赖 ICMPv6 Packet Too Big 做路径 MTU 发现；被拦后大包/握手卡顿、间歇超时。
对策：防火墙放行 ICMPv6 常用类型：1,2,3,4,128,129,133-136。

MTU/PPPoE/隧道不匹配
家庭宽带经 PPPoE（1492）或上游隧道，再叠加设备错误 MSS，导致丢包重传。
对策：

临时把接口 MTU 调低试验：sudo ip link set wlo1 mtu 1480；

开启 TCP MSS clamp（nftables）：tcp flags syn / tcp option maxseg size set clamp-to-pmtu。

CDN/GeoDNS 对 IPv6 定位不准
AAAA 记录被分配到较远 PoP，RTT 抖动。
对策：对比 traceroute -6/-4，必要时固定 IPv4 访问关键服务（如企业邮箱）或与运营商/服务商工单修正 GeoIP。

VPN 仅隧道 IPv4（你的现状）
应用解析出 AAAA 后绕过 VPN 直连外网 IPv6，被策略/互联影响，表现为不稳。
对策：让 VPN 承载 IPv6（WG AllowedIPs=::/0 / OpenVPN tun-ipv6 + redirect-gateway ipv6），或在连 VPN 期间禁用 IPv6/优先 IPv4。

CPE/驱动/省电 Bug
老旧路由器的 RA/NDP、DHCPv6-PD 抖动；Linux 无线驱动省电（如 rtw89）导致瞬断。
对策：更新路由固件与网卡驱动；关闭无线省电：iw dev wlo1 set power_save off；TLP 中关闭 Wi-Fi 省电。
```

也没完全看懂。

哎，坑爹的网络……
