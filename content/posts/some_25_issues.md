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

最近在 deepin 25 上遇到几个问题，处理方案记录一下，以备未来自己或者其他人需要。

### 01 待机睡死问题

有一段时间 25 没有预装 TLP，导致系统功耗升高，有时电脑发烫明显。发现后，一方面通知系统研发做集成更新，另一方面自己先装上 TLP 做验证。

装上 TLP 系统功耗是降下来了，但不出意料引入了稳定性问题：**偶发待机睡死**，概率还不低。找内核同事帮忙看过，受限于出差远程调试不便，也没有立刻跑出有效线索，于是自己做了几轮尝试。

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

因为明确知道是装了 TLP 后出现的问题，把现象发给 ChatGPT 后得到一些方向。

第一次尝试把 `amdgpu` 和 `nvme` 加入 `RUNTIME_PM_DRIVER_DENYLIST`：

```
RUNTIME_PM_DRIVER_DENYLIST="nvme amdgpu mei_me nouveau radeon"
```

效果一般。于是**先“一刀切”关闭 RUNTIME_PM** 验证：

```
RUNTIME_PM_ON_AC=off
RUNTIME_PM_ON_BAT=off
```

果然有效，基本未再复现待机睡死。

> 注：该方案过于激进，**适合个人兜底，不宜主线默认**，后续仍需结合驱动/固件继续收敛问题面。

------

### 02 网络不稳定问题

最初表现是**邮箱收发常失败**。一开始怀疑是客户端缺乏重试，但结合场景（家/酒店更易复现）以及上面 TLP 带来的不稳定，进一步追问 ChatGPT，得到提示：

```
某些驱动/网卡会因省电引入短时 TX/队列停顿，可能被 AP 的 band-steering 频繁“踢/引流”，造成频段来回跳。
```

于是尝试**禁用 Wi-Fi 省电**。可选方式：

1. **驱动层面禁用**

   ```
   # 为特定网卡添加配置，禁用 powersave
   options iwlwifi power_save=0
   ```

2. **NetworkManager 层面禁用**

   ```
   # /etc/NetworkManager/conf.d/wifi-powersave.conf
   [connection]
   wifi.powersave = 2     # 2=禁用，3=启用
   ```

3. **针对特定网络禁用**

   ```
   nmcli connection modify WIFI名字 802-11-wireless.powersave 2
   ```

我偏向系统策略，因此只是在**家/公司多 AP**的环境，禁用特定 WiFi的 省电。之后问题**明显减少**。

------

### 03 邮箱收发邮件问题

即便网络不稳的问题得到缓解，**邮箱收发邮件仍出现稳定失败的情况**。给客户端增加重试机制，没有解决。

这时看到同事讨论商店访问不稳，怀疑和 **IPv6 优先**有关：25 默认跟随上游策略走 IPv6，但链路并不稳定。试探性验证：

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

果然优先解析到 IPv6。两种调整思路：

1. **修改 `gai.conf` 提升 IPv4 优先级**

   ```
   # /etc/gai.conf
   precedence ::ffff:0:0/96  100
   ```

2. **内核层面直接禁用 IPv6（更激进）**

   ```
   sudo sysctl -w net.ipv6.conf.default.disable_ipv6=1
   sudo sysctl -w net.ipv6.conf.all.disable_ipv6=1
   ```

我选择方案一。调整后再次验证：

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

可以看到**默认走到 IPv4** 了。**邮件重试机制 + IPv4 优先**后，问题基本解决。

问了下 ChatGPT 为什么 IPv6 不稳，说了一堆我也没看懂的玩意儿：

```
主要成因（命中频率靠前）：

ICMPv6 被阻断 → PMTU 黑洞
IPv6 依赖 ICMPv6 Packet Too Big 做路径 MTU 发现；被拦后大包/握手卡顿、间歇超时。
对策：防火墙放行 ICMPv6 常用类型：1,2,3,4,128,129,133-136。

MTU/PPPoE/隧道不匹配
家庭宽带经 PPPoE（1492）或上游隧道，若设备错误设置 MSS，可能丢包重传。
对策：
临时调低接口 MTU：sudo ip link set wlo1 mtu 1480；
开启 TCP MSS clamp（nftables）：tcp flags syn / tcp option maxseg size set clamp-to-pmtu。

CDN/GeoDNS 对 IPv6 定位不准
AAAA 记录被分配到较远 PoP，RTT 抖动。
对策：对比 traceroute -6/-4；必要时固定 IPv4 访问关键服务或与运营商/服务商修正 GeoIP。

VPN 仅隧道 IPv4
应用解析出 AAAA 后绕过 VPN 直连外网 IPv6，受策略/互联影响表现为不稳。
对策：让 VPN 承载 IPv6（WG AllowedIPs=::/0 / OpenVPN tun-ipv6 + redirect-gateway ipv6），或连 VPN 期间禁用 IPv6/优先 IPv4。

CPE/驱动/省电 Bug
老旧路由的 RA/NDP、DHCPv6-PD 抖动；Linux 无线驱动省电影响稳定性。
对策：更新路由固件与网卡驱动；关闭无线省电（如：iw dev wlo1 set power_save off；TLP 中关闭 Wi-Fi 省电）。
```

哎，坑爹的网络……
