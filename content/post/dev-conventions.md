---
title: "deepin 开发者规范"
date: 2019-07-30T11:03:54+08:00
---

为了保证研发项目的规范化，避免不一致造成的效率低下，特在此规范中对项目目录、README、代码风格、版权信息和版本控制等方面进行了一定约束，公司内部及合作项目均需按此规范进行。



## 项目目录

一般情况下，项目目录结构按照下表说明进行创建和管理，对于复杂项目由开发经理按需以此为基础调整项目目录结构：

| 目录名    | 说明                                                         |
| --------- | ------------------------------------------------------------ |
| src/      | 存放项目源代码文件。                                         |
| 3rdparty/ | 存放本项目引用的第三方项目源代码或者二进制。如非顶级依赖，可以适当调整放入src目录相应的模块中。 |
| tests/    | 单元测试存放目录。                                           |
| docs/     | 用于存放帮助手册、man等文件。                                |
| tools/    | 项目用到的工具和脚本。                                       |
| po/       | 存放项目用到的翻译文件，也可以为translations、i18n等明显可以看出是翻译文件存放目录的名称。 |
| misc/     | 项目需要使用到的数据文件或者不属于前面任一类型的文件。       |
| .desktop  | 项目的desktop文件，如有多个可以放在misc目录中。              |
| README    | 项目的说明文档，用于说明项目的概要情况。                     |
| LICENSE   | 项目采用的授权协议。                                         |
| Makefile  | 构建脚本，同类型的还有pro文件CMakeLists.txt文件等。          |



## README

每个项目必需包含该README文件，README文件内需包含如下内容：

- 项目描述
- 依赖（包括编译依赖和运行时依赖）
- 构建和安装
- 项目目录说明
- 获取帮助（开发项目要求）
- 参与开发（开源项目要求）
- 授权协议说明

参考模板如下：

```markdown
# Project Name

Project description.

## Dependencies

### Build dependencies

- Qt >= 5.6

### Runtime dependencies

- git 

## Installation

### Build

​````
$ mkdir build
$ cd build
$ qmake ..
$ make
​````

### Install

​```
$ sudo make install
​```

3. Install:

​````
$ sudo make install
​````

## Getting help

Any usage issues can ask for help via

* [Gitter](https://gitter.im/orgs/linuxdeepin/rooms)
* [IRC channel](https://webchat.freenode.net/?channels=deepin)
* [Forum](https://bbs.deepin.org)
* [WiKi](https://wiki.deepin.org/)

## Getting involved

We encourage you to report issues and contribute changes

* [Contribution guide for developers](https://github.com/linuxdeepin/developer-center/wiki/Contribution-Guidelines-for-Developers-en). (English)
* [开发者代码贡献指南](https://github.com/linuxdeepin/developer-center/wiki/Contribution-Guidelines-for-Developers) (中文)

## License

[Project name] is licensed under [GPLv3](LICENSE).

```



## 代码风格

Qt/C++ 代码风格规范详见：[Qt/C++代码风格指南](https://docs-blog.wh-redirect.deepin.cn/?p=2788)；

Golang 代码风格按官方风格，使用go fmt进行格式化，此处不再详述。



## 版权信息

开源项目（DDE和应用）除开发库开源协议需具体安排外，其余项目均优先采用GPLv3协议进行开源；

协议头模板：

```
/*
 * Copyright (C) 2019 ~ %YEAR% Deepin Technology Co., Ltd.
 *
 * Author:     %USER%
 *
 * This program is free software: you can redistribute it and/or modify
 * it under the terms of the GNU General Public License as published by
 * the Free Software Foundation, either version 3 of the License, or
 * any later version.
 *
 * This program is distributed in the hope that it will be useful,
 * but WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
 * GNU General Public License for more details.
 *
 * You should have received a copy of the GNU General Public License
 * along with this program.  If not, see <http://www.gnu.org/licenses/>.
 */



```

私有项目（网站项目等）无需LICENSE文件，在README文件的版权信息部分指明版权信息为公司所有：

```
Copyright (C) 2019 Deepin Technology Co., Ltd.
```



## 版本管理

项目版本管理如无特殊要求，使用`git`作为项目版本管理软件。

### 代码提交

代码提交过程需遵循以下要求：

1. 每个需尽量保证短小完整，如在一个提交内只修复一个问题、只完成一个功能，而不是一个提交内既有问题修复又有功能开发；

2. 提交信息格式为：

   ```
   提交类型(模块)：提交短描述
   空行
   提交长描述
   ```

   其中，提交类型主要包括fix（问题修复）、feat（功能开发）、build（构建修改）、docs（文档）、chore（其他）；模块部分可省略。
   
3. 代码提交后，需由项目组中其他开发人员审核加分后才能合并；

### 分支命名规范

#### 开发分支

- 默认的开发为 `master` 分支，同时作为社区版的维护分支，所以要保证一定的稳定性，不能太过于激进；
- 影响比较大的功能，开新分支进行开发，分支命名规则为 `dev/功能代号` ，如 `dev/hidpi` (一个项目的开发分支定期清理一下，不要超过5个)。

#### 维护分支

- 社区版因为滚动更新的原因，维护分支为master分支；
- 专业版维护分支命名规则为 `maintain/pro/专业版版本号`，各个项目在同一个专业版上使用同样的维护分支，如 `maintain/pro/15.5`。

### 其他要求

- 推送至服务器的提交、tag等禁止修改，以避免协作者处出现不必要的问题。

