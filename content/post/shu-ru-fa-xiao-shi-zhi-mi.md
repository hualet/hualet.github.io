---
title: "输入法消失之谜"
date: 2017-08-05T21:26:48+08:00
---

最近不少用户在deepin论坛上报告说搜狗输入法的图标不见了，收到反馈我就心想坏了，我的输入法图标很早前就消失不见了，之前发现这个问题但是没有去跟踪是因为没有看到其他同事出现类似情况，我的电脑平常为了调试用户反馈的bug又经常XJB装软件，觉得是个例。现在收到多人反馈，大概又是什么“更新事故”？

带着沉重的心情，首先要确定的是这个问题影响的范围：

- 15.4.1的ISO是不存在这个问题的；
- 不用输入法的老外是没有受到影响的；
- 只有少量论坛用户报告了这个问题，大部分人表示更新到最新也没有这个问题；

还好影响范围还比较小，那么下一步就是找解决办法了：

- 想办法绕过？对于这个问题来说好像不太好使；
- debug找到问题的根本原因；

（注：其实用户反馈是集中在这两周的时间甚至更短，如果是其他人的话可能会去排查最近更新了什么包，但是我刚好很早以前遇到了这个问题，受其干扰，所以走了一条冤枉路……）

下面进入本文主题，怎么去debug？以前压根儿没有调过输入法的问题啊，遂去请教公司大神，大神表示：”我也不知道啊 （无奈脸 ”～ 那么我们只能瞎猜了：

第一步：卸载掉搜狗，排除是否是搜狗输入法（问题多多，值得怀疑）导致的。

结果显示卸载掉搜狗输入法以后，小企鹅的图标可以正常显示，看来果然是搜狗输入法导致的？

第二步：下载官方搜狗搜狗输入法，重新安装，排除搜狗输入法版本的问题。

重新安装后没卵用。但是突然想起来搜狗在家目录下的.config/sogou-qimpanel/skin 里面有皮肤缓存，据Google说是皮肤有可能导致输入法图标不见的问题……我们在做2D极速版适配的时候，玩了点黑魔法，替换了一些资源文件，可能是改坏了什么东西？

第三步：删除皮肤缓存，排除修改皮肤的问题。

没卵用…… 陷入僵局，但是发现如果将sogou-qimpanel这个进程杀掉，输入法的图标就会显示出来了……那么估计是这个进程负责读取sogou的皮肤文件然后更改输入法图标，中间可能出现了什么岔子？

第四步：使用strace跟踪sogou-qimpanel，感觉胜利在望。

    $ killall sogou-qimpanel && strace -e open sogou-qimpanel 

打印着打印着，吭哧，sogou-qimpanel尼玛居然退出了……什么鬼，再次尝试，还是退出了。看来是daemonize了？那么strace attach到子进程呢？貌似关键的信息已经丢了……陷入僵局……

如何在系统级别追踪呢？？？

第五步： 看来要祭出大杀器systemtap了，随手写（抄）了一个stp：

    probe syscall.open.return {
      printf("%s %s", filename, execname())
    }

运行失败……

    WARNING: never-assigned local variable 'filename' (similar: name, __nr, retstr): identifier 'filename' at sogou_qimpanel_open.stp:2:20
     source:   printf("%s, %s", filename, execname())

请教公司另外一大神，大神表示“你装一下kernel的debug包就行了……” 结果，还是不行……坑……

这时候考虑sysdig（翻了好长时间bearychat的聊天记录无果，又去请教公司另另外一个大神才找到这个工具），本来准备晚上再跟踪一下，这时候中间又闹了一个 xscreensaver 在deepin上使用无法解锁的问题，引出使用capabilities替换setuid设定的问题，进而引出如果检测二进制需要的capabilities的方法（跳得有点快，略过细节），进而发现bcc里面的opensnoop工具，而且这个工具集在deepin仓库里面有打包：bpfcc-tools。

这不就是我一直要找的系统级别文件打开追踪工具么！！！至此，从问题暴露到现在已经过去两天时间……

第六步：使用opensnoop-bpfcc工具来跟踪sogou-qimpanel

    $ sudo opensnoop-bpfcc -n sogou-qimpanel -T 

在另外一个终端里面启动sogou-qimpanel

    $ killall sogou-qimpanel && sogou-qimpanel

分别在正常的机器（输入法图标可见）上和在不正常的机器（输入法图标不可见）上查看opensnoop-bpfcc的输出，经过复杂的对比可发现在正常的机器上多了一行输出：

    4.169245000   5282   sogou-qimpanel     30   0 /usr/share/icons/hicolor/16x16/status/fcitx-kbd.png

但是不正常的机器上并没有这一条，而这个图标确实就是应该显示在托盘里面的输入法图标。这下可以肯定是有问题的机器上因为某种原因，sogou-qimpanel查找hicolor主题里面的fcitx-kbd这个图标失败了；

第七步：测试qt4程序访问hicolor主题里面的fcitx-kbd图标，缩小范围

写了一个简单的qt4测试程序，输出 QIcon::hasThemeIcon("fcitx-kbd")的结果。

果然，在输入法图标可见的机器上，输出为true；在输入法图标不可见的机器上，输出为false；这样就可以断定是某个原因导致qt4在有问题的机器上查找图标失败了。

继续使用strace跟踪这个测试程序在两种机器上的open情况，发现在正常的机器上，测试程序open了/usr/share/icons/hicolor/index.theme；而在不正常的机器上，测试程序open了 ~/.local/share/flatpak/exports/share/icons/hicolor/index.theme，这就很明显了，是因为机器上有flatpak的影响。我之所以很早就出现了这个问题，是因为我很早以前就装了flatpak进行测试，而最近论坛有人反馈这个问题是因为我们最近在搞flatpak应用内测。多么简单的道理……

继续追踪不难从XDG_DATA_DIRS有多余的flatpak的路径，进而发现是flatpak丢在 /etc/X11/Xsession.d/中的20flatpak文件（以及/etc/profile.d/中的flatpak.sh）为整个user session导入了上面的环境变量；那么按照图标主题的覆盖规则，就算是有更高优先级路径里面存在hicolor主题，并且里面不存在我们要查找的图标，Qt也应该能找到系统hicolor主题里面的图标呀？为啥失败了呢？ 这个我并不吃惊，因为Qt对于Linux下面的图标主题查找很渣啊～ 

读qt4的源码，发现其逻辑是对于一个主题，在XDG_DATA_DIRS里面找到一个有效的主题目录（目录名一致 && 包含index.theme文件），后面的就不再继续处理了 …… 遂修改之，提交patch，谜团终于解开了。

又可以愉快地写我的go代码了 😄
