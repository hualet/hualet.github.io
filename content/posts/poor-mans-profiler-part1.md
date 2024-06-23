---
title: "自制 Profiler 第一部分"
date: 2018-05-13T14:42:41+08:00
---

​	最近对性能剖析的技术颇感兴趣。好不容易来了三分钟热度，自然不能浪费，因此在余热消失之前研究并实践了其中一部分细节，对于其中一些知识点，个人感觉对于自身编程能力提升还是比较有益的，因此在这里写出来，抛砖引玉。



## Linux profiler简介

​	性能剖析的工具，其实在Linux平台还是挺多的，比如小巧实用的strace、ltrace、latrace，大名鼎鼎的google-perftools、gprof、valgrind，以及瑞士军刀型的linux-perf等等，它们主要分为三个阵营，一个是针对程序执行的性能进行剖析，对程序执行的热点进行分析，如*trace、gprof这些工具；另一个是针对程序运行过程中内存的使用进行剖析，方便针对性地做内存优化，如valgrind；最后一部分就是“脚踏两条船”的，两个我都做，比如google-perftools、linux-perf这些，我们这个系列主要集中在“针对程序执行的性能进行剖析”方面。



## Profiler的本质

​	要自制Profiler，首先要知道Profiler的本质是什么。简单来说，Profiler的本质其实就是在程序执行的过程中对程序正在“做什么”进行搜集和统计，如何搜集呢？无非两种：

1. Instrumentaion - 程序主动Profiler告诉它在做什么；
2. Sampling - Profiler自己间断性地去看程序在做什么；

前者的实现主要依靠程序运行时提供的某些钩子机制、代码插桩等方式，例如gprof主要是依靠 gcc 在编译程序的时候”夹带私货“来达到程序运行的时候主动提供给gprof采样样本，来达到事后分析的目的[1]。这种方式相对来说虽然比较可靠、准确，但是对于无法控制编译条件的程序就比较无可奈何了。

后者的实现则主要是定期对程序当前执行的指令和对程序执行的函数堆栈进行回溯（unwind）来尽量还原程序执行的过程，例如google-perftools就是采用这种方式，这种情况下，采样的周期就显得尤为重要。

​	对比以上两种方式，我们果断采用第二种。

​	PS: 这里需要说明一下的是，利用ptrace系统调用完成工作的strace和ltrace，虽然不依赖编译器夹带私货，但是相当于依赖了内核的“钩子”，不属于sampling的范围；同样，latrace则是依赖了ld的LD_AUDIT“钩子”，也不属于sampling的范围。



## Profiler启动

​	那么问题又来了，ptrace没得用，我们怎么去获取被剖析程序的执行状态呢？总不至于profiler要搞成root权限的吧？答案是：不，内核管天管地，总管不到程序自己偷看自己的数据吧？我们想办法把我们的代码塞到被剖析程序中去就可以了！

​	刚才提到了ld的LD_AUDIT，这次就轮到它的兄弟——大名鼎鼎的LD_PRELOAD——登场了。想法是这样的，我们的profiler其实只需要在程序开始的时候执行一个定时器，以后每次定时器执行的时候去抓取我们的样本就OK了，所以我们完全可以把自己伪装成一个人畜无害的动态库，等到别个程序有意无意加载到我们，哼哼……事实上，很多profiler都是采取类似的策略，比如google-perftools，再比如heaptrack等等。



## 编码实战

​	废话少说，放码过来！	

​	我们在QtCreator中创建一个动态库项目 simple-profiler，主类 SimpleProfiler。首先，我们需要设置好我们的定时器：

``` c++
void SimpleProfiler::enableProfile()
{
    int ret = setitimer(ITIMER_PROF, &m_tick, nullptr);
    if ( ret != 0) {
        fprintf(stderr, "failed to enable profiler: %s", strerror(errno));
    }
}
```

根据 `setitimer` 的man文档，计时器主要有三种类型：

- `ITIMER_REAL` 这个计时器是根据墙上时间进行倒计时，最终触发 `SIGALRM` 信号；
- `ITIMER_VIRTUAL` 这个计时器是根据进程花费在用户空间的 CPU 时间进行倒计时的，最终触发 `SIGVTALRM` 信号；
- `ITIMER_PROF` 这个计时器是根据进程话费在用户空间和内核空间的 CPU 时间进行倒计时的，最终触发  `SIGPROF` 信号；

个人认为，假如都是用在 profiler 上的话，第一种计时器用来查找程序执行慢（某个函数有IO等情况）的瓶颈比较有效果；后两种计时器用来查找程序 CPU 占用瓶颈比较有效果。不过使用第一种计时器的话，对被追踪程序的执行影响相较于后两者就比较大了，所以这里采用第三种计时器。

​	`setitimer` 的第二个参数 `m_tick` 是 `SimpleProfiler` 类的一个成员，类型是： `struct itimerval`，在 `SimpleProfiler::setupProfileTimer` 中初始化：

 ``` c++
void SimpleProfiler::setupProfileTimer()
{
    m_tick.it_value.tv_sec = 0;
    m_tick.it_value.tv_usec = m_sampleInterval;
    m_tick.it_interval.tv_sec  = 0;
    m_tick.it_interval.tv_usec = m_sampleInterval;
}
 ```

​	其中，`m_sampleInterval` 固定在 10000，因此上面计时器的配置相当于是 10ms 后开启计时器，每 10ms 触发一次 SIGPROF 信号。

​	再看我们的信号处理：

```c++
void SigProfHandler(int, siginfo_t*, void*)
{
    _instace.disableProfile();

    printf("Doing sampling here. \n");

    _instace.enableProfile();
}

void SimpleProfiler::setupSignal()
{
    struct sigaction action;
    action.sa_sigaction = reinterpret_cast<void (*)(int, siginfo_t *, void *)>(SigProfHandler);
    sigfillset(&action.sa_mask);
    action.sa_flags = SA_SIGINFO;
    sigaction(SIGPROF, &action, 0);
}
```

​	这里之所以不使用 `signal` 函数是因为如果程序有多个线程，某一个线程触发 signal handler 的时候，其他线程会不会被暂停执行是没有统一定义的，所以不同的系统的反应可能不一致，你必须对 signal handler 做特殊处理，比如加锁等，才能保证这个函数的正确性；但是 `sigaction` 函数却能保证一致性。而我们的程序，按照我们的伟大理想，是要对不同线程进行区分剖析的，所以这里使用 `sigaction` ，一劳永逸。

​	同时，我们需要注意在我们的 `SigProfHandler` 中先把我们的 profiler 禁用掉（`_instance` 是 `SimpleProfiler`的一个实例），执行完我们的采样以后（现在是 `printf` 站位），再打开。否则，我们的 profiler 很可能会把自己的很多动作也算到采样中，增加太多的误差。

​	现在，我们只需要一步就能完成我们 profiler 的第一个版本了：在我们主类的构造函数中开始profile，并且初始化一个 `SimpleProfiler` 类的全局对象，全局对象会在程序加载的过程中被初始化，这样只要被剖析的程序一加载我们的动态库，我们的 profiler 就开始跑了，美滋滋。

```C++
SimpleProfiler _instace;

SimpleProfiler::SimpleProfiler()
{
    setupSignal();
    setupProfileTimer();
    enableProfile();
}
```



## tracee 程序

​	为了方便测试，我们自制了一个被剖析程序，其实就是使用 QtCreator 创建一个 `Qt Widgets Application` 项目，然后设定一个计时器，执行一点耗费 CPU 的函数，假如我们最终版的 profiler 可以找到这个函数，就相当于我们达成了自制 profiler 的小目标：

```c++
Widget::Widget(QWidget *parent)
    : QWidget(parent)
{
    QTimer *timer = new QTimer(this);
    timer->setSingleShot(false);

    connect(timer, &QTimer::timeout,
            this, &Widget::consumeCPU);

    timer->start(100);
}

void Widget::consumeCPU()
{
    int j = 0;
    for (int i = 0; i < 1000000; i++) {
        j += i;
    }
}


int main(int argc, char *argv[])
{
    QApplication a(argc, argv);

    Widget w;
    w.show();

    return a.exec();
}
```



## 连接和测试

​	我们只需要简单地在 tracee 的项目文件中加入以下行，就可以让tracee程序加载我们的 profiler 了：

```
QMAKE_RPATHDIR += $$OUT_PWD/../src
LIBS += -L$$OUT_PWD/../src/ -lsimple-profiler
```

其中的`src`目录即我们动态库项目的地址。

​	按说这时候执行 tracee 程序，我们就能看到一大堆 `Doing sampling here.`的输出了，但是这里有两个比较坑的地方，很可能导致你的程序在执行的时候并没有任何输出：

- 首先是 QtCreator 比较坑爹，假如你用 `qDebug` 输出的话，会很及时的输出到 `Application Output` 面板，但是用 `printf` 就是不行。我猜测可能是因为 QtCreator 使用`QProcess` 处理子程序输出的时候有问题之类的；
- 另外一个是跟我们之前使用 `ITIMER_PROF`有关，因为我最开始的时候 tracee 程序并没有 `consumeCPU` 这个函数，导致显示出窗口以后进程主要的状态就是等待 IO （X11 事件），怎么也触发不到这个计时器，当时被搞得好沮丧。

好了，这里只需要到程序编译目录，手动执行 ./tracee 就可以看到一大堆 `Doing sampling here.` 的输出了，不过现在我们的 tracee 程序是修改了pro文件，主动加载了我们的动态库，怎么做到不修改源程序就达到目的呢？很简单，刚才说了使用 `LD_PRELOAD` 嘛:

```
LD_PRELOAD=../src/libsimple-profiler.so ./tracee
```

测试效果灰常好。 喜滋滋！



代码地址： https://github.com/hualet/simple-profiler/tree/part1



其余的下一篇继续 :P



## 参考链接

\[1\] [GCC Instrumentation Options](https://gcc.gnu.org/onlinedocs/gcc/Instrumentation-Options.html)

\[2\] [C++全局对象的初始化和析构](https://blog.csdn.net/challeng_everything/article/details/45460091)

