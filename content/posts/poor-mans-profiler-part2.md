---
title: "自制 Profiler 第二部分——调用栈回溯"
date: 2018-05-14T14:19:37+08:00
---

书接上篇，我们现在已经能在其他程序中执行我们自己的代码，并且也做到了以固定的频率去执行采样代码（我们的`printf`），但是如何采样还是一个问题，这篇文章会就这这个问题继续探讨接下去我们面临的挑战——调用栈回溯。

为什么要获取函数调用栈？一方面是因为profiler除了要分析程序存在的性能问题，即函数执行热点以外，还需要帮助我们可怜的程序员找到问题的原因，这时候能提供问题函数的堆栈信息就非常必要了；另一方面，我们上一篇文章其实说了，是为了通过堆栈信息尽量还原程序的执行过程：试想一个程序执行的过程是 main->funca->funcb->funcc，我们第一次采样 main->funca，第二次采样 main->funca->funcb->funcc，假如我们没有堆栈信息，我们只会统计一次 funca 和一次funcc，但是这并不能反应事实，相反，我们有堆栈信息的话，就会把 funca 、funcb 和 funcc 各计数一次，更能反应实际的执行过程。



## 概念

函数调用栈（Call Stack）和相应的栈帧（Stack Frame）我们其实都不陌生：在使用 gdb 调试程序的时候，bt（backtrace）命令打印出来的就是函数调用栈；而函数调用栈列表中的每一项则代表一个栈帧，我们执行 frame 命令跳转到某一个栈帧，其实就是一次回溯的过程。

想要在内存中解析出我们想要的函数调用栈，首先我们需要知道的就是一个程序的stack 段里面各个栈帧是如何布局的，要搞清楚这个，我们还需要了解一个概念叫：[调用约定](https://en.wikipedia.org/wiki/Calling_convention)（Calling Convention），调用约定主要约定了（好绕）：

- 函数的参数是如何传递的，是全都放到寄存器，还是全都放在 stack 段，还是混用两者；
- 函数的参数是按什么顺序放置到内存中的；
- 函数中的本地变量是如何分配的；
- 子函数调用是如何返回的；
- 子函数的栈帧是如何清理的；
- 等等

所以，调用约定基本上决定了函数调用中每个栈帧的产生、压栈、出栈对内存布局的影响，而这个约定是因架构和平台而异的。我们这里只关注x86 平台下的 [cdecl](https://en.wikipedia.org/wiki/X86_calling_conventions#cdecl) 约定。

在这个约定下，假如我们有一个函数 `DrawSqure` 调用了 `DrawLine` （例子来自[Wikipedia](https://en.wikipedia.org/wiki/Call_stack)），那么程序内存布局中的 stack 段就应该是类似下图所示：

![call stack](/img/2018/05/14/Call_stack_layout.svg)

每个函数调用即创建一个栈帧，每个栈帧一次压入 stack 中。

其中，`Stack Pointer`(`esp`) 永远指向栈顶， `Frame Pointer`(`ebp`) 指向当前栈帧的中一个固定的地方（基地址）；函数参数以从右往左的顺序依次压栈，然后是压入`Return Address` ，它是当前函数（或者栈帧）执行完成后，程序要继续执行的指令地址， 同时压入父函数的栈帧基地址（`Saved EBP`），它是当前函数执行完成以后，`Stack Pointer` 和 `Frame Pointer` 将会指向的地方，基于这个地址，程序指令可以方便地访问函数本地变量（`ebp`负向偏移）和函数参数（`ebp`正向偏移）。 

![stack info](/img/2018/05/14/stackIntro.png)

结合上面两张图，其实可以看出，每个栈帧其实都保存了上一个栈帧的基地址，因此所有的栈帧最终组成了一个链表，这也就是我们能拿到函数堆栈的理论基础了。

（注：上面只是粗略的讲解，参考链接 [1] 非常详细的描述了函数调用的过程中栈帧、stack 段和`esp`、`ebp`寄存器的变化，如果感兴趣，可以详细了解一下。）



## 参考方案

看完上面一大串概念以后，我们发现如果我们要按照函数约定的方式去获取函数调用堆栈，可以，但是太过蛋疼，而且不跨平台，很难受。 所以秉承不要重复造轮子的优良传统，我们发现有几个方式可以简单地获取到函数调用栈：

1. 神奇的 `__builtin_return_address` 宏；
2. `glibc` 的 `backtrace` 和 `backtrace_symbols`；
3. `libunwind`

第一种方案看起来很神奇，你在程序中可以很方便地通过

```
printf("%p", __builtin_return_address(1))
```

打印父函数的地址，将 `1` 换成 `2`就可以打印爷爷函数的地址，依次类推。但是它有两个很致命的问题，一个是这个宏的参数不能使用变量；另外一个是你无法知道调用栈啥时候到头了，就是你增加参数的值两次、三次发现都能好好工作，等到你把参数换成可能`4`的时候，你的程序“咔嚓”就崩溃了……可能是因为访问到了栈顶以上的内存地址？鬼知道……

第二种方式有一个例子：

```
#include<stdio.h>
#include<stdint.h>
#include<execinfo.h>

#define SIZE 100

void print_backtrace()
{
	int size;
	void *buffer[SIZE];

	size = backtrace(buffer, SIZE);

	char** stack = backtrace_symbols(buffer, size);
	for (int i = 0; i < size; i++) {
		printf("%s \n", stack[i]);
	}
}

int func_one(int a)
{
	print_backtrace();

	return a + 3;
}

int func_two(int a, int b)
{
	int c = func_one(a);

	return c + a;
}

int main(int argc, char** argv)
{
	func_two(1, 2);
	return 0;
}
```

需要在给  `gcc` 或者 `g++` 编译的时候加一个 `-rdynamic` 的参数[3]，输出结果：

```
./test-backtrace(print_backtrace+0x1f) [0x55d7f0ee9939] 
./test-backtrace(func_one+0x15) [0x55d7f0ee99ac] 
./test-backtrace(func_two+0x18) [0x55d7f0ee99cc] 
./test-backtrace(main+0x1e) [0x55d7f0ee99f7] 
/lib/x86_64-linux-gnu/libc.so.6(__libc_start_main+0xf1) [0x7f7c5aae22b1] 
./test-backtrace(_start+0x2a) [0x55d7f0ee983a] 
```

都依赖编译加参数了，这不是我们的风格嘛，显然不能接受。

第三种方案 `libunwind` 是目前比较流行的方案，连 `linux-perf` 都在用它，靠谱到我都想用！



## 编码实战

又到了码代码的时候了，不过因为 `libunwind` 使用起来非常简单，我们只需要一个函数即可：

```
void Backtrace()
{
    unw_cursor_t cursor;
    unw_context_t context;
    
    // context 封装了当前机器状态的参数值，例如不同寄存器的值等
    // cursor 相当于是栈帧的迭代器
    unw_getcontext(&context);
    unw_init_local(&cursor, &context);
    
    // 一帧一帧地回溯，直到没有可用父栈帧
    while (unw_step(&cursor) > 0) {
        unw_word_t offset, pc;
        // IP (instruction pointer) 就是上面个我们提到 Return address 所指向的地址
        unw_get_reg(&cursor, UNW_REG_IP, &pc); 
        if (pc == 0) {
            break;
        }
        printf("0x%lx \n", pc);
        
        // 通过函数指针，获取函数对应的名称
        char sym[256];
        if (unw_get_proc_name(&cursor, sym, sizeof(sym), &offset) == 0) {
            printf(" (%s+0x%lx)\n", sym, offset); 
        } else {
            printf(" -- error: unable to obtain symbol name for this frame\n");
        }
    }
}
```

透过一个 `unw_cursor_t` 我们每次回溯一帧，直到没有可用的父栈帧，在每一个栈帧中，获取到当前栈帧的 `Return address` 区域所指向的函数地址，这样就构成了我们最终想要的函数栈。

这个时候别忘了在我们的项目上加载 `libunwind`  这个库：

```
CONFIG += link_pkgconfig
PKGCONFIG += libunwind
```

然后，用这个函数替换我们原来的占位打印：

```
void SigProfHandler(int, siginfo_t*, void*)
{
    _instace.disableProfile();

    // printf("Doing sampling here.");
    Backtrace();

    _instace.enableProfile();
}
```

就可以打印出函数调用栈了：

```
0x7f773eb79011:  (_Z14SigProfHandleriP9siginfo_tPv+0x27)
0x7f773d66f0c0:  (__restore_rt+0x0)
0x557a9c3969e4:  (_ZN6Widget10consumeCPUEv+0x2e)
0x557a9c3970ec:  (_ZN9QtPrivate11FunctorCallINS_11IndexesListIJEEENS_4ListIJEEEvM6WidgetFvvEE4callES7_PS5_PPv+0x6b)
0x557a9c39707e:  (_ZN9QtPrivate15FunctionPointerIM6WidgetFvvEE4callINS_4ListIJEEEvEEvS3_PS1_PPv+0x42)
0x557a9c396fe8:  (_ZN9QtPrivate11QSlotObjectIM6WidgetFvvENS_4ListIJEEEvE4implEiPNS_15QSlotObjectBaseEP7QObjectPPvPb+0x75)
0x7f773dd8781c:  (_ZN11QMetaObject8activateEP7QObjectiiPPv+0x99c)
0x7f773dd93dd8:  (_ZN6QTimer10timerEventEP11QTimerEvent+0x28)
0x7f773dd8826b:  (_ZN7QObject5eventEP6QEvent+0x7b)
0x7f773e67728c:  (_ZN19QApplicationPrivate13notify_helperEP7QObjectP6QEvent+0x9c)
0x7f773e67c37f:  (_ZN12QApplication6notifyEP7QObjectP6QEvent+0x23f)
0x7f773dd5c268:  (_ZN16QCoreApplication15notifyInternal2EP7QObjectP6QEvent+0x108)
0x7f773ddadbde:  (_ZN14QTimerInfoList14activateTimersEv+0x4de)
0x7f773ddae101:  (_ZN14QTimerInfoList14activateTimersEv+0xa01)
0x7f773c2ef887:  (g_main_context_dispatch+0x287)
0x7f773c2efab8:  (g_main_context_dispatch+0x4b8)
0x7f773c2efb5c:  (g_main_context_iteration+0x2c)
0x7f773ddaec3f:  (_ZN20QEventDispatcherGlib13processEventsE6QFlagsIN10QEventLoop17ProcessEventsFlagEE+0x5f)
0x7f773dd5a53a:  (_ZN10QEventLoop4execE6QFlagsINS_17ProcessEventsFlagEE+0xea)
0x7f773dd625ed:  (_ZN16QCoreApplication4execEv+0x8d)
0x557a9c3967a5:  (main+0x4b)
0x7f773ca452b1:  (__libc_start_main+0xf1)
0x557a9c39667a:  (_start+0x2a)
```

不过那些函数名称是怎么搞得？当然是著名的 [C++ mangling](https://en.wikipedia.org/wiki/Name_mangling) 了，使用命令行工具 `c++filt` 可以方便地进行“解密”工作：

```
$> c++filt _ZN10QEventLoop4execE6QFlagsINS_17ProcessEventsFlagEE
$> QEventLoop::exec(QFlags<QEventLoop::ProcessEventsFlag>)
```

在代码里面我们可以借助于  `libstdc++` 提供的 `cxxabi.h` 的 `API`  来实现 demangling：

```
char sym[256];
if (unw_get_proc_name(&cursor, sym, sizeof(sym), &offset) == 0) {
    char* nameptr = sym;
    int status;
    char* demangled = abi::__cxa_demangle(sym, nullptr, nullptr, &status);
    if (status == 0) {
        nameptr = demangled;
    }
    printf(" (%s+0x%lx)\n", nameptr, offset);
    free(demangled);
} else {
    printf(" -- error: unable to obtain symbol name for this frame\n");
}
```

这时候再看函数调用栈打印就比较完美了。

```
0x7fc6ef4350fe:  (SigProfHandler(int, siginfo_t*, void*)+0x27)
0x7fc6edf2b0c0:  (__restore_rt+0x0)
0x5581be96575b:  (Widget::consumeCPU()+0x25)
0x5581be965b16:  (QtPrivate::FunctorCall<QtPrivate::IndexesList<>, QtPrivate::List<>, void, void (Widget::*)()>::call(void (Widget::*)(), Widget*, void**)+0x6b)
0x5581be965aa8:  (void QtPrivate::FunctionPointer<void (Widget::*)()>::call<QtPrivate::List<>, void>(void (Widget::*)(), Widget*, void**)+0x42)
0x5581be965a12:  (QtPrivate::QSlotObject<void (Widget::*)(), QtPrivate::List<>, void>::impl(int, QtPrivate::QSlotObjectBase*, QObject*, void**, bool*)+0x75)
0x7fc6ee64381c:  (QMetaObject::activate(QObject*, int, int, void**)+0x99c)
0x7fc6ee64fdd8:  (QTimer::timerEvent(QTimerEvent*)+0x28)
0x7fc6ee64426b:  (QObject::event(QEvent*)+0x7b)
0x7fc6eef3328c:  (QApplicationPrivate::notify_helper(QObject*, QEvent*)+0x9c)
0x7fc6eef3837f:  (QApplication::notify(QObject*, QEvent*)+0x23f)
0x7fc6ee618268:  (QCoreApplication::notifyInternal2(QObject*, QEvent*)+0x108)
0x7fc6ee669bde:  (QTimerInfoList::activateTimers()+0x4de)
0x7fc6ee66a139:  (QTimerInfoList::activateTimers()+0xa39)
0x7fc6ecbab887:  (g_main_context_dispatch+0x287)
0x7fc6ecbabab8:  (g_main_context_dispatch+0x4b8)
0x7fc6ecbabb5c:  (g_main_context_iteration+0x2c)
0x7fc6ee66ac3f:  (QEventDispatcherGlib::processEvents(QFlags<QEventLoop::ProcessEventsFlag>)+0x5f)
0x7fc6ee61653a:  (QEventLoop::exec(QFlags<QEventLoop::ProcessEventsFlag>)+0xea)
0x7fc6ee61e5ed:  (QCoreApplication::exec()+0x8d)
0x5581be965525:  (main+0x4b)
0x7fc6ed3012b1:  (__libc_start_main+0xf1)
0x5581be9653fa:  (_start+0x2a)
```

再仔细看一眼，输出的一个函数是我们的信号处理函数，这个不奇怪，但是第二个 `__restore_rt`  是什么鬼？

另外，其实上面代码中通过函数地址获取函数名称的地方是比较耗时的，所以如果我们每次采样都做这个操作是比较坑的，会严重影响 tracee 程序的执行效率，我们只能在采样的时候先记录这些函数地址，事后进行统计操作的时候再解析到函数的名称。为什么通过函数地址获取函数名称这么耗时呢？

连同上一个问题，我们会在后续的两篇中给出答案。



本篇代码地址： https://github.com/hualet/simple-profiler/tree/part2



## 参考链接

\[1\] [Journey to the stack, Part I](https://manybutfinite.com/post/journey-to-the-stack/)

\[2\] [Programmatic access to the call stack in C++](https://eli.thegreenplace.net/2015/programmatic-access-to-the-call-stack-in-c/)

\[3\] [How to make backtrace()/backtrace_symbols() print the function names?](https://stackoverflow.com/questions/6934659/how-to-make-backtrace-backtrace-symbols-print-the-function-names)
