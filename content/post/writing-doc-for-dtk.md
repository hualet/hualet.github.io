---
title: "如何给 DTK 添加文档"
date: 2018-09-26T17:10:14+08:00
---

## 背景

经过几年的摸爬滚打，DTK 作为构建 deepin 全家桶的基石，一直被说的主要有两个毛病：

1. 内部人员觉得它不够稳；
2. 外部人员觉得它无从下手；

第一个问题主要原因是 DTK 在最开始的时候缺失的东西太多，各种剧烈的变化同时进行，但是又没有比较好的版本控制和接口管理，所以经常是这个应用要用新版本，新版本删除了一个废弃的接口，用了旧接口的应用就崩了。由于最近 DTK 的变化日趋平稳，以及有了基本的版本控制，第一个问题渐渐淡出了人们的视野。

第二个问题外部人员无从下手的原因是 DTK 在开发过程中并没有留下接口文档，所以目前想使用 DTK，内部靠得是口口相传，外部靠的是阅读源码，完全的石器时代……甚至连社区的小伙伴都看不下去了，自己做了一个[《Deepin开发指南》](http://deepin.lolimay.cn/) 。



是时候给DTK添加一下文档了！



## 工具选择

其实， DTK 之前是有写文档的，但是写得并不多，主要问题就是英文文档不好写。我也尝试写过，每个类、每个函数都只能憋出一句英文，这种文档跟没有是差不多的。所以，这里需要做一个艰难的决定：文档用中文来写，英文靠爱好者来翻译。

用什么文档工具呢？ DTK 是基于 Qt 之上的开发库，最自然的想法是使用 QDoc，但是据使用 QDoc 搭建了 [dde-file-manager](https://github.com/linuxdeepin/dde-file-manager) 的 [Github Pages](https://linuxdeepin.github.io/dde-file-manager/) 的 @BLumia 同学说 QDoc 对国际化的支持不是特别好，我们按道理是要支持中英双语文档的，遂放弃。除了 QDoc 以外，文档生成工具就是 Doxygen、还有一个KDE 项目自己的文档工具 KApiDox ，它们三者之间是什么关系呢？Doxygen 从 QDoc 上 fork 出来的，KApiDox 是基于 Doxygen 的封装……我没有怎么看 KApiDox ，所以就选择了 Doxygen 。



## 如何写

下面就是规则部分了。

### 1. 文档注释位置

为了保持头文件的清爽和干净，所有文档注释都要写在源文件中。

### 2. 多语言

按照 Doxygen 多语言处理的方式，需要在中文文档前添加 `\~chinese` 命令，在英文文档前添加 `\~english` 命令，这样设置 Doxyfile 中 `OUTPUT_LANGUAGE=Chinese`之后，就只会包含中文文档，而不会夹杂英文了，如：

```c++
/*!
 * \~english \brief DApplication::loadTranslator loads translate file form
 * \~english system or application data path;
 * \~english \param localeFallback, a list of fallback locale you want load.
 * \~english \return load success
 *
 * \~chinese \brief DApplication::loadTranslator 加载程序的翻译文件。
 * \~chinese 使用这个函数需要保证翻译文件必须正确命名: 例如程序名叫 dde-dock，
 * \~chinese 那么翻译文件在中文locale下的名称必须是 dde-dock_zh_CN.qm；翻译文件还需要放置
 * \~chinese 在特定的位置，此函数会按照优先级顺序在以下目录中查找翻译文件：
 * \~chinese 1. ~/.local/share/APPNAME/translations;
 * \~chinese 2. /usr/local/share/APPNAME/translations;
 * \~chinese 3. /usr/share/APPNAME/translations;
 *
 * \~chinese \param localeFallback 指定了回退的locale列表，默认只有系统locale。
 * \~chinese \return 加载成功返回 true，否则返回 false。
 */
```

### 3. 一般的格式

#### 函数

格式如下，函数简介第一句需要包含完整的主谓宾（`\brief`命令后的简介，如无特殊说明，均照此要求），如 “setSingleInstance 用于将程序设置成单实例“。

```
\brief 函数名 函数简介
函数详细作用描述
\param 参数名 参数作用介绍
\return 返回值说明
```

例如:

```c++
/**
 * \~chinese \brief DApplication::setSingleInstance setSingleInstance 用于将程序
 * \~chinese 设置成单实例。
 * \~chinese \param key 是确定程序唯一性的ID，一般使用程序的二进制名称即可。
 *
 * \~chinese \note 一般情况下单实例的实现使用 QSystemSemaphore，如果你的程序需要在沙箱
 * \~chinese 环境如 flatpak 中运行，可选的一套方案是通过 DTK_DBUS_SINGLEINSTANCE 这个
 * \~chinese 编译宏来控制单实例使用 DBus 方案。
 *
 * \~chinese \return 设置成功返回 true，否则返回 false。
 */
```

输出：

![class](img/2018/09/26/function.png)

#### 类

因为类的声明一般都在头文件中，所以在源文件中写类的文档注释时需要注意，放在相应 Private 类的函数实现之后，格式如下：

```
\class 类名
\brief 类简介
类详细作用描述
```

例如：

```c++
/*!
 * \~chinese \class DApplication
 *
 * \~chinese \brief DApplication 是 DTK 中用于替换 QCoreApplication 相关功能实现的类。
 * \~chinese 继承自 QApplication，并在此之上添加了一些特殊的设定，如：
 * \~chinese - 在 FORCE_RASTER_WIDGETS 宏生效的情况下，默认设置 Qt::AA_ForceRasterWidgets 以减少 glx 相关库的加载，减少程序启动时间；
 * \~chinese - 自动根据 applicationName 和 系统 locale 加载对应的翻译文件；
 * \~chinese - 会根据系统gsettings中 com.deepin.dde.dapplication 的 qpixmapCacheLimit 值来设置 QPixmapCache::cacheLimit；
 * \~chinese - 方便地通过 setSingleInstance 来实现程序的单实例。
 *
 * \~chinese \sa loadTranslator, setSingleInstance.
 */
```

输出：![function](img/2018/09/26/function.png)

![class](img/2018/09/26/class.png)

#### 结构体

结构体出声明关键字为 `\struct` 外，其余与类格式要求相同。

#### 枚举

枚举的声明也是一般都在头文件中，所以在源文件中写类的文档注释时需要注意，放在所属类的文档注释之后，格式如下：

```
\enum 枚举名
\brief 枚举简介
枚举主要作用详细介绍
\var 枚举名 枚举值 枚举值说明

```

需要注意枚举值使用 `\var` 命令来声明，命令之后需要先跟上枚举名，再写枚举值。例如：

```c++
/*!
 *
 * \~chinese \enum DApplication::SingleScope
 * \~chinese DApplication::SingleScope 定义了 DApplication 单实例的效应范围。
 *
 * \~chinese \var DApplication::SingleScope DApplication::UserScope
 * \~chinese 代表单实例的范围为用户范围，即同一个用户会话中不允许其他实例出现。
 *
 * \~chinese \var DApplication::SingleScope DApplication::SystemScope
 * \~chinese 代表单实例的范围为系统范围，当前系统内只允许一个程序实例运行。
 */
```

输出：

![enum](img/2018/09/26/enum.png)

#### 属性

Doxygen 可以自动识别到 Qt 中的属性，也可以使用 `\property` 命令给属性添加文档，但是属性并不会自动关联到相应的 accessor（ getter 和 setter ） 函数，这里暂不处理，属性的文档注释写到属性的 getter 函数之上，getter 和 setter 函数可以不写文档。

格式如下：

```
\property 属性名
\brief 属性名简介
```

例如：

```c++
/*!
 * \~chinese \property DApplication::visibleMenuIcon
 * \~chinese \brief visibleMenuIcon 属性代表了程序中菜单项是否显示图标。
 */
```

输出：

![property](img/2018/09/26/property.png)

#### 信号

Doxygen 可以自动识别到 Qt 的信号，我们在源文件写信号的文档时，需要使用 `\fn` 命令来给信号添加对应的文档，位置应放置在所属类的文档注释下面。格式如下：

```
\fn 信号名
\brief 信号简介
```

例如：

```c++
/*!
 * \~chinese \fn DApplication::newInstanceStarted()
 * \~chinese \brief newInstanceStarted 信号会在程序的一个新实例启动的时候被触发。
 *
 * \~chinese \fn DApplication::iconThemeChanged()
 * \~chinese \brief iconThemeChanged 信号会在系统图标主题发生改变的时候被触发。
 */
```

输出：

![signal](img/2018/09/26/signal.png)

### 4. 其他一些用法

因为写文档大家都是新手，所以其他高级用法就没有做什么特殊说明和要求了，文档中引用本类、其他类或者函数只需要写上相应的名称即可，非常简单。另外，也非常鼓励使用 `\sa` 和 `\example` 等标签来加强文档的便捷性和可读性。

> 注：引用 Qt 相关类和函数的方式跟引用本地函数没有不同，只是需要配置一下 Doxygen ，这篇文章暂时略过，会有单独的文章补充说明，写文档的时候尽情引用便是。



## 工具

推荐使用 QtCreator 进行文档编写，QtCreator 对 Doxygen 代码注释有自动补全的功能，有三种形式：

- `///`
- `//!`
- `/*!`

在函数名、类名等声明上一行输入以上几种注释格式并按下回车，QtCreator 会自动按照函数的语法信息生成 `\brief` 、`\param`  和 `\return`　等指令和响应的参数，可以说非常方便了。

不过，鉴于 deepin 内部开发主要写中文文档，手写 `\~chinese `的工作量实在太大，所以修改了一个版本的QtCreator ，作补全时会自动添加 `\~chinese` 标签。开发团队都可以在”不清真deb包“这个共享目录中找到响应的 deb 包。



## 集成

曾经有同学问 [deepin-tool-kit](https://github.com/linuxdeepin/deepin-tool-kit)  这个项目是干啥用的，这里刚好解释一下：

DTK 原来是在 deepin-tool-kit 这个项目里面的，发展壮大以后分家了：dtkcore、dtkwidget 和 dtkwm，所以现在这个项目主要是用来做文档管理用的，所有 dtk 的子项目都作为 submodule 放在这个项目内进行（文档的）版本控制，这个项目也配置了 Doxygen ，就可以在这个项目里面统一做所有 dtk 子项目的文档生成了。

所以，文档生成主要是在 deepin-tool-kit 这个项目中进行，以后持续集成之类的也会在这个项目上进行配置。



### 本地测试

如果需要在编写文档的过程中进行本地测试，如在 dtkwidget 项目中，有两种方式：

1. 通过各种手段把修改的 dtkwidget 项目放置到 deepin-tool-kit 项目中；
2. 简单地做一个 Doxygen 的临时配置。

然后使用 `doxygen` 命令生成文档进行测试即可。

第一种方式稍微麻烦点，但是很简单，就不多说了。

第二种方式也只需要简单几步：

1. 在项目源码目录运行 `doxygen -g` 生成一个 Doxyfile 配置文件；
2. 修改 Doxyfile 中 `OUTPUT_LANGUAGE=Chinese`，`INPUT=./src`，`RECURSIVE=YES`，`OUTPUT_DIR=docs`；
3. 然后运行 `doxygen`，即可在 docs/html 目录中找到生成的文档，打开 index.html 即可查看。

