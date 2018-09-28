---
title: "给 Doxygen 生成的文档中 Qt 的引用添加链接"
date: 2018-09-28T16:42:40+08:00
---

额……这个标题怎么改读着都不怎么顺口，不过不用在意这些细节，问题不大 :)



上篇文章提到了 [如何给DTK添加文档](https://hualet.org/blog/2018/09/26/%E5%A6%82%E4%BD%95%E7%BB%99-dtk-%E6%B7%BB%E5%8A%A0%E6%96%87%E6%A1%A3/) ，给同事讲的时候有同事提出来最好能自动给对 Qt 的引用加上链接，我当时拍胸脯说文档里面 Qt 的类和方法随便引用，到时候一定让生成的文档可以链接到 Qt 的文档上。其实，到啥时候我也说不准……所以，还是研究了一下，翻了半天的 Stack Overflow 和 其他一些资料才搞清楚要怎么做到，这里记录一下。



如何在文档中添加外部文档的链接呢？[官方文档](https://www.stack.nl/~dimitri/doxygen/manual/external.html) 说得其实挺简单的： `TAGFILES` ！最开始没有仔细看文档，理解歪了，以为 tags 文件是一个文档压缩包，添加到 `TAGFILES` 这项后面生成文档的时候自动下载，然后自动根据索引添加引用链接到文档中。所以，根据 [doxygen generated documentation with auto-generated links to qt project](https://stackoverflow.com/questions/22244779/doxygen-generated-documentation-with-auto-generated-links-to-qt-project) 这篇文章，Doxyfile 配置文件写了：

```
TAGFILES = qtcore.tags=http://doc.qt.io/qt-5/ \
qtgui.tags=http://doc.qt.io/qt-5/ \
qtwidgets.tags=http://doc.qt.io/qt-5/ \
qtxml.tags=http://doc.qt.io/qt-5/ \
qtnetwork.tags=http://doc.qt.io/qt-5/
```

生成文档，怎么都不生效…… WTF ?!

继续 Google ，发现一个[提交](https://github.com/pencil2d/pencil/pull/893/files)， 里面的

```
for i in core svg xmlpatterns; do
    curl -fsSLO "https://doc.qt.io/qt-5/qt$i.tags";
done;
```

引起了我的注意，恍然大悟：

**原来这个 tags 文件就是索引文件，需要预先下载下来，之所以有 = 加上一个网址，是因为 tags 中只有类名到文档名的映射，并没有规定使用某种协议以及host。**

不过 Qt 这些 tags 文件链接还真是难找，貌似没有任何正式的文档说明如何下载，真不知道这些人是怎么找到的。

把文件下载下来，放到 [Docs](https://github.com/linuxdeepin/deepin-tool-kit/tree/master/Docs/) ，然后修改 Doxyfile：

```
TAGFILES = Docs/qtcore.tags=https://doc.qt.io/qt-5/ \
Docs/qtgui.tags=https://doc.qt.io/qt-5/ \
Docs/qtwidgets.tags=https://doc.qt.io/qt-5/ \
```

重新生成文档就能在文档中看到 Qt 引用变成超链接了（[commit](https://github.com/linuxdeepin/deepin-tool-kit/commit/85e8a3f4a3ad541a62caa348c5a3b7fd01714fef)）。 



现在，除了 Qt 官网打开比较满以外就没啥问题了。



参考链接：

- [Linking to external documentation](https://www.stack.nl/~dimitri/doxygen/manual/external.html)
- [Qt Weekly #17: Linking Qt Classes in Documentation Generated with Doxygen](http://blog.qt.io/blog/2014/08/13/qt-weekly-17-linking-qt-classes-in-documentation-generated-with-doxygen/)
- [doxygen generated documentation with auto-generated links to qt project](https://stackoverflow.com/questions/22244779/doxygen-generated-documentation-with-auto-generated-links-to-qt-project)
- [Don` t see (linked) Qt classes in Doxygen](https://stackoverflow.com/questions/34209425/don-t-see-linked-qt-classes-in-doxygen) 