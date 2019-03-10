---
title: "Flirting With Fuse2"
date: 2019-03-10T23:12:14+08:00
draft: true
---

## libfuse

​	上面简单介绍了FUSE，但是跟开发者打交道的主要是FUSE框架中的libfuse，如果说FUSE让开发文件系统的门槛大大降低的话，libfuse则是让开发文件系统的工作量大大降低。因为libfuse将大部分跟具体业务不相关的事情都做了（如多线程处理等），开发者只需要根据自己具体的业务逻辑需求将所有需要实现的接口函数都实现了即可。

​	libfuse的API分为high level和low level两种类型，在high level的接口中，开发者需要实现的函数的第一个参数一般是文件的路径；而在low level的接口中，开发者需要实现的函数的第一个参数则一般是类似内核中`inode`的一个结构体。我的感觉是high level的实现更快，low level的性能更好，又是一个效率和性能的取舍，对于初学者当然是使用high level了。

​	