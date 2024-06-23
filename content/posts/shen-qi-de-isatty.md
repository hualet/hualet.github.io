---
title: "神奇的isatty"
date: 2017-08-29T21:26:48+08:00
---

前些日子才从apt-get命令系列换成更为时尚的apt系列，作为一个debian系发行版——deepin的开发者，我表示很汗颜……新的apt命令除了在功能上将apt-get、apt-cache等几个命令统一到了一个命令上外，更是有了不错的TUI，如文档所说：

    The `apt` command is meant to be pleasant for end users and does not need to be backward compatible like apt-get(8).

毕竟，还多了进度条呢…… 😂

不过这不是今天要说的重点，换到apt以后，把apt操作的一些结果跟管道结合一起用，经常会收到警告，例如：

    (ssh) hualet@hualet-PC : ~
    [0] % apt search deepin | grep -i superstar
    
    WARNING: apt does not have a stable CLI interface. Use with caution in scripts.
    

CLI的输出也算是API要保持stable么？汗颜again……dtk作为一个系统级的开发库都还没有到stable的状态、某in公司的CTO写得命令行工具，第二天接口就全变了😂……敬畏之余，困惑我的倒不是这个警告的内容，毕竟新的东西都不保证稳定么，但是apt是怎么知道它的输出被连到管道了呢？

一直没空处理，直到昨天，又一次遇到了，遂记下，晚上思来想去没什么想法，遂请教前文提到的不靠谱CTO，丫直接甩过来一句：

“istty检测output啊，很多命令行程序都会根据这个做不同的反应”

我：“man istty没有结果啊“

他：”你man page没装全吧？“

我默默敲下sudo apt install manpages-dev，显示已经安装了……

我：“装全了啊，我Google一下吧”

他：“肯定没装全，应该是在libbsd或者termios里的吧……“ 信誓旦旦。

我默默Google了一圈，发现那个函数其实叫isatty，在unistd.h里面定义

我：“你是不是记错了，有isatty，unistd.h里面的……”

他：“恩额”

就是这么不靠谱一人……

知道了isatty，写了一个程序测试一下：

    #include<unistd.h>
    #include<stdio.h>
    
    int main(int argc, char** argv) {
      printf("%d", isatty(1)); // 1 是特殊的fd，代表标准输出
    }

没挂管道的话输出是1，挂了管道输出是0。就是这么神奇，只怪自己对POSIX的标准函数都这么生疏，再次汗颜……是时候翻翻TUPE了。

有兴趣看isatty是什么原理的，可以看看apple的[实现](https://opensource.apple.com/source/Libc/Libc-167/gen.subproj/isatty.c)。



参考链接：

[1], [How does isatty() get the information from the terminal?](https://superuser.com/a/932170)
