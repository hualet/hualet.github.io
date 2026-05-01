+++
draft = false
date = 2026-05-01T22:34:23+08:00
title = "最近 vibe coding 的几个小项目"
description = ""
slug = ""
authors = []
tags = ["AI","随想"]
categories = []
externalLink = ""
series = []

+++

26年春节前后，Coding Agent 的能力又大涨了一波，以至于我也按耐不住，春节后到现在陆陆续续 Vibe Coding 了些小项目，探索 Coding Agent 新一波的能力、Harness Engineering 之余，还能满足一些自己的小需求。

不得不说这一波不管是从模型能力（Claude Opus 4.6 、GPT 5.4/5.5、GLM 5.1、Kimi K2.5/2.6、DeepSeek V4 pro），还是 Coding Agent 的工具能力（Claude code、Opencode、codex 、kimi cli）都又飞涨了一大截。相较于去年6月份体验AI编程（Cursor），现在 Vibe Coding 一次性满足需求的准确率提升了不止一丢丢，基本上在单点需求上99%不用返工。

除了这个，“蒸馏”了软件工程最佳实践的一些插件和skills，也进一步提升了AI辅助软件工程的靠谱程度，其中典型的 omo（oh-my-opencode）、superpowers、openspec 等等。以及面向驾驭多 Agent的 clawteams ，在提升并行开发效率方面也给碳基程序员打开了不少脑洞。

不过随着Coding Agent 本身的飞速进化（一周2-3个版本），很多能力已经被官方纳入，比如 plan、ralph loop、multi-agent、btw 等等，我个人现在倾向于 codex/opencode + superpowers 这套极简组合方案。



## 项目列表



### dde-shell-weather-plugin 



![dde-shell-weather-plugin-project-card](/img/2026/05/dde-shell-weather-plugin-project-card.png)



[dde-shell-weather-plugin](https://github.com/hualet/dde-shell-weather-plugin)  算是年后第一个正儿八经 vibe coding 的项目，为 deepin V25 新的 dde-shell 提供了天气插件，目前已经上架应用商店，迭代了有6、7个版本。



主要功能：

- 任务栏上展示位置和天气信息

  ![dde-shell](/img/2026/05/dde-shell.png)

- 显示未来整点时间天气

  ![dde-shell-weather-plugin](/img/2026/05/dde-shell-forcast.png)

- 支持动态图标，天气刷新的时候图标会有动态效果

- 支持亮色、暗色主题自适应

- 支持任务栏居中/居左自适应

- 支持自动定位（使用免费服务、不太稳定）和手动定位



这个项目最好玩的地方在于：

1）实实在在地让我感受到了范式变化。 以前我如果要做一个插件，最高效的方式是先去熟悉一下它的插件开发文档、实现demo、逐步增加功能、边调试边修改……而这个项目的工作方式是，我直接丢给 codex 了 dde-shell 的项目代码，告诉它我要做一个天气插件，让它帮我完成一个原型，我在此基础之上逐步完善产品功能设想、流水线、打包上架等。顺带手，我还让 codex 给 dde-shell 提了个 PR，补充了插件相关的文档和示例 😜；

2）AI 可以大幅拓展我的能力边界。 动态天气图标本来只是个提案，并没有报以太大的期望。但是在我告诉 codex 以后，没想到它真的帮我找到了实现路径并且实现。要不是受限于 25 系统中 Qt 的6.8版本老了一丢丢，在 Qt 6.10 上可以使用 Animated Vector Graphics 按道理可以更好实现；

3）AI 不完全是遵循，也可以跟你辩论了。 在给项目调整 API 选型的时候，我让它给我使用XX API 服务，它居然反过来跟我说建议我不要这么做，因为 XX API 明确了免费接口不能商用。 在我跟它强调我这个项目将来会完全开源以后，它才悻悻作罢 😂。





### DBusLens



![DBusLens-project-card](/img/2026/05/DBusLens-project-card.png)



[DBusLens](https://github.com/hualet/DBusLens)  项目是一个分析 DBus 流量的工具，缘起于我总感觉我们系统的  DBus 被滥用，而 bustle 没有我想要的分析维度，所以想做一个监控分析工具。 目前已经推送到 pypi 软件包仓库。

主要功能：

- 按照 Sender 维度聚合分析：

  ![img](https://raw.githubusercontent.com/hualet/DBusLens/main/docs/attachments/senders.svg)

- 按照 Members 维度聚合分析：

  ![img](https://raw.githubusercontent.com/hualet/DBusLens/main/docs/attachments/members.svg)

- 分析主要报错，以及报错的调用方

  ![img](https://raw.githubusercontent.com/hualet/DBusLens/main/docs/attachments/errors.svg)

- 从 Latency 角度聚合：

  ![img](https://raw.githubusercontent.com/hualet/DBusLens/main/docs/attachments/latency.svg)

- 以图表形式展示调用的依赖关系：

  ![img](https://raw.githubusercontent.com/hualet/DBusLens/main/docs/attachments/plot.svg)



虽然最终没有观测到明显的问题，没能证明我认为系统DBus存在“滥用”的猜想，但是这何尝不是一件好事呢。



### qqmail-cli

![qqmail-cli-project-card](/img/2026/05/qqmail-cli-project-card.png)



[qqmail-cli](https://github.com/hualet/qqmail-cli)  是个袖珍项目，主要是在让 openclaw/Hermes Agent 辅助我查阅邮件的过程中，发现腾讯邮箱的搜索接口实现得不标准，以至于使用 himalaya 等 cli 邮件客户端的时候，搜索邮件的效率异常地低，所以算是为腾讯邮箱做了个特化的 cli 绕过这个问题。

有兴趣的可以看项目介绍，这里就不展开了。



### deepin-terminal-ghostty 



![deepin-terminal-ghostty-project-card](/img/2026/05/deepin-terminal-ghostty-project-card.png)



[deepin-terminal-ghostty](https://github.com/hualet/deepin-terminal-ghostty) 算是这两个月 vibe coding 得最大的项目了，这个项目的诞生源于我希望 deepin-terminal 支持垂直标签页和更多现代化终端协议（比如 kitty image protocol）的需求。

但是，原版 deepin-terminal 大量修改了 vendor 的 qtermwidget ，导致在遇到上游 BUG /功能缺失，无法快速跟进上游项目，我对此是“深恶痛绝”。同时，又因为我对 ghostty 这个网红终端心向往之。所以，在选择基于 deepin-terminal 做开发、和基于原生 qtermwidget 做开发两者中，我选择了基于 ghostty 🤣 。

说是基于  ghostty，但是 ghostty 现在并没有提供等同于 qtermwidget 的开发库，只提供了 libghostty-vt 处理 vt 的基础协议实现。所以 deepin-terminal-ghostty 目前还没有等同于 ghostty  的 GPU 渲染能力，仍然是基于 QPainter 绘制，未来是否自研 GPU 渲染还是等上游，还要看上游拆分 lib 库的速度。



目前已经实现的功能：

- 绝大部分原版本 deepin-terminal 的特性：多主题、远程管理、多标签页、横竖分屏等

- 新增垂直标签页，可以展示标签页和内部分屏
- 初步的 shell 集成，支持：
  - Coding Agent  识别和徽章展示
  - 命令执行的状态提醒

![Show Case](https://github.com/hualet/deepin-terminal-ghostty/raw/main/resources/showcase.png)



这个项目因为规模稍大，好玩的地方多了一些，对于 harness engineering 的要求和可玩性也增加，比如 单元测试、自动化测试、各类兼容性测试、性能测试 以及 崩溃问题处理。 计划单独写一篇文章，或者内部做个分享，回头再补充。



##  Vibe Coding 配置



模型：

- 主力：GPT 5.4/5.5
- 辅助：Kimi K2.5/2.6、GLM 5.1
- 次级辅助：DeepSeek V4 pro



Coding Agent：

- 主力：codex + superpowers
- 辅助：
  - opencode + superpowers
  - kimi cli



没有太多远程 Coding 的需求，所以虽然配了 Paseo ，但是并没有大量使用。



## 综合感受



1）vibe coding 上瘾。久违了的创造和快速学习的快感，通过 vibe coding 可以得到快速释放，经常从 00：30 拖到 1：30或者2:00 才睡觉，导致我的早睡计划一直无法实现；

2）有了 AI 以后，以前需要代理给其他人做的事情，现在可以方便地交给 AI。比如我看到 rtk 号称能节省不少 token，本来是安排同事去做调研的，后来转念一想可以交给 AI 啊，结果 AI 做得又快又好，也减少了对同事的干扰。

3）以前受限于“想象力”做得慢或者不会做的事情，现在也可以让 AI 做。比如 给GUI程序补充单元测试，这是以前我觉得没法做得，但是在 deepin-terminal-ghostty 项目中 AI 给我啪啪打脸……在 dde-shell-weather-plugin  项目中， AI 帮我把 CSS-Animated SVGs 转换成  SMIL-Animated SVGs，参见 [gist](https://gist.github.com/hualet/0b43abc9c4e7f56d465f0879b5b4e6dd)。

4）vibe coding 的同时也要 vibe learning。deepin-terminal-ghostty 开发过程中，需要不停地补充终端底层的知识，比如终端是怎么跟shell通信的、shell 集成是不是真集成了个 shell、各类测试工具带来的问题等等。

5）harness engineering 的本质还是软件工程。

6）AI 时代最重要的是 taste中，taste 这个词应该有更广泛的含义。不只是界面设计，技术、工程、工具都考验眼界和审美，你的眼界和审美决定了你项目能达成的高度。
