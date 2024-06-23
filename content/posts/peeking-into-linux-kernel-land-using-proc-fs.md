---
title: "初探Linux内核态——通过proc文件系统作快速问题定位"
date: 2018-01-23T19:59:00+08:00
---

文章翻译自 [Peeking into Linux kernel-land using /proc filesystem for quick’n’dirty troubleshooting](https://blog.tanelpoder.com/2013/02/21/peeking-into-linux-kernel-land-using-proc-filesystem-for-quickndirty-troubleshooting/)



这篇博客的内容完全是关于现代Linux内核的。换句话说，指的是与RHEL6一样使用的2.6.3x系列内核，而不是古老的RHEL5所使用的2.6.18内核（都什么鬼了？！），虽然大部分企业都还在使用RHEL5。另外，这篇文章也不会涉及内核调试器或者SystemTap脚本之类的东西，完全是最最简单地在有用的proc文件系统节点上执行“cat /proc/PID/xyz”这样的命令。



# 定位一个程序“运行缓慢”的问题 

下面要举的这个例子是这样的：一个DBA反映说他们的find命令一直运行缓慢，半天都没有什么输出，他们想知道这是为什么。听到这个问题的时候我就大概有直觉造成这个问题的原因，但是他们还是想知道怎么系统地追踪这类问题，并找到解决方案。刚好出问题的现场还在……



还好，系统是运行在OEL6上的，内核比较新，确切地说是2.6.39 UEK2。



首先，让我们看看find进程是否还在：

```bash
[root@oel6 ~]# ps -ef | grep find
root     27288 27245  4 11:57 pts/0    00:00:01 find . -type f
root     27334 27315  0 11:57 pts/1    00:00:00 grep find
```

跑的好好的，PID是27288（请记好这个将会伴随整篇博客的数字）。



那么，我们就从最基础的开始分析它的瓶颈：如果它不是被什么操作卡住了（例如从cache中加载它所需要的内容），它应该是100%的CPU占用率；如果它的瓶颈在IO或者资源竞争，那么它应该是很低的CPU占用率，或者是%0。



我们先看下top：

```bash
[root@oel6 ~]# top -cbp 27288
top - 11:58:15 up 7 days,  3:38,  2 users,  load average: 1.21, 0.65, 0.47
Tasks:   1 total,   0 running,   1 sleeping,   0 stopped,   0 zombie
Cpu(s):  0.1%us,  0.1%sy,  0.0%ni, 99.8%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Mem:   2026460k total,  1935780k used,    90680k free,    64416k buffers
Swap:  4128764k total,   251004k used,  3877760k free,   662280k cached

  PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
27288 root      20   0  109m 1160  844 D  0.0  0.1   0:01.11 find . -type f
```

结果很清楚：这个进程的CPU占用率很低，几乎为零。但是CPU占用低也分情况：一种是进程完全卡住了，根本没有机会获得时间片；另一种是进程在不停进入等待的状态（例如poll动作就是时不时超时后，进程进入休眠状态）。虽然这剩下的细节top还不足以完全给我们展示，但是至少我们知道了这个进程没有在烧CPU时间。



通常情况下，如果进程处于这种状态（%0的CPU占用一般说明进程是卡在了某个系统调用，因为这个系统调用阻塞了，内核需要把进程放到休眠状态），我都会用strace跟踪一下这个进程具体卡在了哪个系统调用。而且，如果进程不是完全卡住了，那进程中的系统调用情况也会在strace的输出中有所展示（因为一般阻塞的系统调用会在超时返回后，过一段时间再进入阻塞等待的状态）。



让我们试试strace：

```bash
[root@oel6 ~]# strace -cp 27288
Process 27288 attached - interrupt to quit

^C
^Z
[1]+  Stopped                 strace -cp 27288

[root@oel6 ~]# kill -9 %%
[1]+  Stopped                 strace -cp 27288
[root@oel6 ~]# 
[1]+  Killed                  strace -cp 27288

```

尴尬，strace自己也卡住了！半天没有输出，也不响应Ctrl-C，我不得不通过Ctrl-Z把它扔到后台再杀掉它。想简单处理还真是不容易啊。



那只好再试试pstack了（Linux上的pstack只是用shell脚本包了一下GDB）。尽管pstack看不到内核态的内容，但是至少它能告诉我们是哪个系统调用最后执行的（通常pstack输出的用户态调用栈最顶部是一个libc库的系统调用）：

```bash
[root@oel6 ~]# pstack 27288

^C
^Z
[1]+  Stopped                 pstack 27288

[root@oel6 ~]# kill %%
[1]+  Stopped                 pstack 27288
[root@oel6 ~]# 
[1]+  Terminated              pstack 27288
```

呵呵，pstack也卡住了，什么输出都没有！



至此，我们还是不知道我们的程序是怎么卡住了，卡在哪里了。



好吧，还怎么进行下去呢？还有一些常用的信息可以搜集——进程的status字段和WCHAN字段，这些使用古老的ps就能查看（或许最开始就应该用ps看看这个进程是不是已经成僵尸进程了）：

```bash
[root@oel6 ~]# ps -flp 27288
F S UID        PID  PPID  C PRI  NI ADDR SZ WCHAN  STIME TTY          TIME CMD
0 D root     27288 27245  0  80   0 - 28070 rpc_wa 11:57 pts/0    00:00:01 find . -type f
```

需要注意的是，你要多运行几次ps以确保进程还在同一个状态（不然在不凑巧的时候获取了一个错误的状态就麻烦了），我这里为了简短就只贴一次输出了。



进程此时正处于D状态，按照man手册的说法，这通常是因为磁盘IO导致的。而WCHAN字段（表示导致程序处于休眠/等待状态的函数调用）则有点儿被切掉了。显然我可以翻一下ps的man手册，看看怎么把这个字段调宽一点好完整打印出来，不过既然我都知道了这个信息来自于proc文件系统，就没这个必要了。

直接从它的源头读一下看看（再次说明一下，多试几次看看，毕竟我们还不知道这个进程到底是不是完全卡死了呢）：

```bash
[root@oel6 ~]# cat /proc/27288/wchan
rpc_wait_bit_killable
```



额……这个进程居然在等待某些RPC调用。RPC调用通常意味着这个程序在跟其他进程通信（不管是本地还是远程）。总归，我们还是不知道为什么会卡住。



# 到底是不是完全卡住了？

在我们揭开这篇文章最后的谜底之前，我们还是先搞清楚这个进程到底是不是完全卡住了。



其实，在新一点的Linux内核中，/proc/PID/status 这个文件可以告诉我们这点：

```bash
[root@oel6 ~]# cat /proc/27288/status 
Name:	find
State:	D (disk sleep)
Tgid:	27288
Pid:	27288
PPid:	27245
TracerPid:	0
Uid:	0	0	0	0
Gid:	0	0	0	0
FDSize:	256
Groups:	0 1 2 3 4 6 10 
VmPeak:	  112628 kB
VmSize:	  112280 kB
VmLck:	       0 kB
VmHWM:	    1508 kB
VmRSS:	    1160 kB
VmData:	     260 kB
VmStk:	     136 kB
VmExe:	     224 kB
VmLib:	    2468 kB
VmPTE:	      88 kB
VmSwap:	       0 kB
Threads:	1
SigQ:	4/15831
SigPnd:	0000000000040000
ShdPnd:	0000000000000000
SigBlk:	0000000000000000
SigIgn:	0000000000000000
SigCgt:	0000000180000000
CapInh:	0000000000000000
CapPrm:	ffffffffffffffff
CapEff:	ffffffffffffffff
CapBnd:	ffffffffffffffff
Cpus_allowed:	ffffffff,ffffffff
Cpus_allowed_list:	0-63
Mems_allowed:	00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000001
Mems_allowed_list:	0
voluntary_ctxt_switches:	9950
nonvoluntary_ctxt_switches:	17104
```

(主要看State字段和最后两行：voluntary_ctxt_switches和  nonvoluntary_ctxt_switches)



进程处于D——Disk Sleep（不可打断休眠状态），voluntary_ctxt_switches 和 nonvoluntary_ctxt_switches的数量则能告诉我们这个进程被分给时间片的次数。过几秒再看看这些数字有没有增加，如果没有增加，则说明这个进程是完全卡死的，目前我在追踪的例子就是这个情况，反之，则说明进程不是完全卡住的状态。



顺便提一下，还有两种办法也可以获取进程的上下文切换数量（第二种在老旧的内核上也能工作）：

```bash
[root@oel6 ~]# cat /proc/27288/sched
find (27288, #threads: 1)
---------------------------------------------------------
se.exec_start                      :     617547410.689282
se.vruntime                        :       2471987.542895
se.sum_exec_runtime                :          1119.480311
se.statistics.wait_start           :             0.000000
se.statistics.sleep_start          :             0.000000
se.statistics.block_start          :     617547410.689282
se.statistics.sleep_max            :             0.089192
se.statistics.block_max            :         60082.951331
se.statistics.exec_max             :             1.110465
se.statistics.slice_max            :             0.334211
se.statistics.wait_max             :             0.812834
se.statistics.wait_sum             :           724.745506
se.statistics.wait_count           :                27211
se.statistics.iowait_sum           :             0.000000
se.statistics.iowait_count         :                    0
se.nr_migrations                   :                  312
se.statistics.nr_migrations_cold   :                    0
se.statistics.nr_failed_migrations_affine:                    0
se.statistics.nr_failed_migrations_running:                   96
se.statistics.nr_failed_migrations_hot:                 1794
se.statistics.nr_forced_migrations :                  150
se.statistics.nr_wakeups           :                18507
se.statistics.nr_wakeups_sync      :                    1
se.statistics.nr_wakeups_migrate   :                  155
se.statistics.nr_wakeups_local     :                18504
se.statistics.nr_wakeups_remote    :                    3
se.statistics.nr_wakeups_affine    :                  155
se.statistics.nr_wakeups_affine_attempts:                  158
se.statistics.nr_wakeups_passive   :                    0
se.statistics.nr_wakeups_idle      :                    0
avg_atom                           :             0.041379
avg_per_cpu                        :             3.588077
nr_switches                        :                27054
nr_voluntary_switches              :                 9950
nr_involuntary_switches            :                17104
se.load.weight                     :                 1024
policy                             :                    0
prio                               :                  120
clock-delta                        :                   72
```



我们要的就是输出中的nr_switches字段（等于 nr_voluntary_switches + nr_involuntary_switches），值是27054，跟/proc/PID/schedstat 中的第三个字段是一致的：

```bash
[root@oel6 ~]# cat /proc/27288/schedstat 
1119480311 724745506 27054
```



等一段时间，也是一样的结果，数量没有增长……



# 通过/proc文件系统初探Linux内核态世界

看情况我们的程序是卡死无疑了，strace和pstack这些使用ptrace系统调用来attach到进程上来进行跟踪的调试器也没啥用，因为进程已经完全卡住了，那么ptrace这种系统调用估计也会把自己卡住。（顺便说一下，我曾经用strace来跟踪attach到其他进程上的strace，结果strace把那个进程搞挂了。别说我没警告过你）！



没了strace和pstack，我们还能怎么查程序卡在了哪个系统调用呢？当然是 /proc/PID/syscall 了！

```bash
[root@oel6 ~]# cat /proc/27288/syscall 
262 0xffffffffffffff9c 0x20cf6c8 0x7fff97c52710 0x100 0x100 0x676e776f645f616d 0x7fff97c52658 0x390e2da8ea
```



这鬼输出怎么看呢？很简单，0x加很大的数字一般就是内存地址（使用pmap类似的工具可以看具体他们指向了什么内容），如果是很小的数字的话，很可能就是一些索引号（例如打开的文件描述符，就像/proc/PID/fd的内容一样）。在这个例子中，因为是syscall文件，你可以”合理猜测“这是一个系统调用号：进程卡在了262号系统调用！



需要注意的是系统调用号在不同的系统上可能是不一致的，所以你必须从一个合适的.h文件中去查看它具体指向了哪个调用。通常情况下，在/usr/include中搜索 syscall* 是个很好的切入点。在我的系统上，这个系统调用是在 /usr/include/asm/unistd_64.h中定义的：

```bash
[root@oel6 ~]# grep 262 /usr/include/asm/unistd_64.h 
#define __NR_newfstatat				262
```



根据上面显示的结果可以看到这个系统调用叫 newfstatat，我们只需要man newfstatat就可以知道这是干啥的了。针对系统调用有一个实用小技巧分享：如果man手册中没有发现一个系统调用，那么请尝试删除一些特定的前缀或者后缀（例如man pread64不行就尝试man pread）。还或者，干脆google之……



总之，这个叫“new-fstat-at”的系统调用的作用就是让你可以像 stat 系统调用一样读取文件的属性，我们的程序就是卡在这个操作上，终于在调试这个程序的道路上迈出了一大步，不容易！但是为啥会卡在这个调用呢？



好吧，终于要亮真本事了。隆重介绍：/proc/PID/stack，能让你看到一个进程内核态的调用栈信息的神器，而且只是通过cat一个proc文件！！！

```bash
[root@oel6 ~]# cat /proc/27288/stack
[] rpc_wait_bit_killable+0x24/0x40 [sunrpc]
[] __rpc_execute+0xf5/0x1d0 [sunrpc]
[] rpc_execute+0x43/0x50 [sunrpc]
[] rpc_run_task+0x75/0x90 [sunrpc]
[] rpc_call_sync+0x42/0x70 [sunrpc]
[] nfs3_rpc_wrapper.clone.0+0x35/0x80 [nfs]
[] nfs3_proc_getattr+0x47/0x90 [nfs]
[] __nfs_revalidate_inode+0xcc/0x1f0 [nfs]
[] nfs_revalidate_inode+0x36/0x60 [nfs]
[] nfs_getattr+0x5f/0x110 [nfs]
[] vfs_getattr+0x4e/0x80
[] vfs_fstatat+0x70/0x90
[] sys_newfstatat+0x24/0x50
[] system_call_fastpath+0x16/0x1b
[] 0xffffffffffffffff
```



可以看到最顶部的函数就是我们现在卡在的地方，正是上面WCHAN字段显示的内容：rpc_wait_bit_killable。（注意：实际上不一定最顶部的函数就是我们想要的，因为内核可能也执行了 schedule 之类的函数来让程序进入休眠或者运行。这里没有显示可能是因为卡主是因为其他调用卡主了才进入睡眠状态，而不是相反的逻辑）。



多亏了这个神器，我们现在可以从头到尾推导出程序卡主的整个过程和造成最终 rpc_wait_bit_killable 函数卡主的原因了：

​	最底部的  system_call_fastpath 是一个非常常见的系统调用处理函数，正是它调用了我们刚才一直有疑问的 newfstatat 系统调用。然后，再往上可以看到一堆nfs相关的子函数调用，这样我们基本可以断定正nfs相关的下层代码导致了程序卡住。我没有推断说问题是出在nfs的代码里是因为继续往上可以看到rpc相关的函数，可以推断是nfs为了跟其他进程进行通信又调用了rpc相关的函数——在这个例子中，可能是`[kworker/N:N]`, `[nfsiod]`, `[lockd]` or `[rpciod]`等内核的IO线程，但是这个进程因为某些原因（猜测是网络链接断开之类的问题）再也没有收到响应的回复。



同样的，我们可以利用以上的方法来查看哪些辅助IO的内核线程为什么会卡在网络相关的操作上，尽管kworkers就不简简单单是NFS的RPC通信了。在另外一次问题重现的尝试（通过NFS拷贝大文件）中，我刚好捕捉到一次kwrokers等待网络的情况：

```bash
[root@oel6 proc]# for i in `pgrep worker` ; do ps -fp $i ; cat /proc/$i/stack ; done
UID        PID  PPID  C STIME TTY          TIME CMD
root        53     2  0 Feb14 ?        00:04:34 [kworker/1:1]

[] __cond_resched+0x2a/0x40
[] lock_sock_nested+0x35/0x70
[] tcp_sendmsg+0x29/0xbe0
[] inet_sendmsg+0x48/0xb0
[] sock_sendmsg+0xef/0x120
[] kernel_sendmsg+0x41/0x60
[] xs_send_kvec+0x8e/0xa0 [sunrpc]
[] xs_sendpages+0x173/0x220 [sunrpc]
[] xs_tcp_send_request+0x5d/0x160 [sunrpc]
[] xprt_transmit+0x83/0x2e0 [sunrpc]
[] call_transmit+0xa8/0x130 [sunrpc]
[] __rpc_execute+0x66/0x1d0 [sunrpc]
[] rpc_async_schedule+0x15/0x20 [sunrpc]
[] process_one_work+0x13e/0x460
[] worker_thread+0x17c/0x3b0
[] kthread+0x96/0xa0
[] kernel_thread_helper+0x4/0x10
```

通过开启内核的tracing肯定可以确切找到是内核的哪两个线程之间再通信，不过限于文章篇幅，这里就不展开了，毕竟这只是一个实用（且简单）的问题追踪定位练习。



# 诊断和“修复”

多亏了现代内核提供的栈信息存样，我们得以系统地追踪到我们的find命令卡在了哪里——内核中的NFS代码。而且一般情况下，NFS相关卡死，最需要怀疑的就是网络问题。你猜我是怎么重现上面的这个问题的？其实就是在功过NFS挂在虚拟中的一块磁盘，执行find命令，然后让虚拟机休眠……这样就可以重现类似网络（配置或者防火墙）导致的链接默默断掉但是并没有通知TCP另一端的进程的情况。



因为 rpc_wait_bit_killable 是可以直接被安全干掉的函数，这里我们选择通过 kill -9直接干掉它：

```bash
[root@oel6 ~]# ps -fp 27288
UID        PID  PPID  C STIME TTY          TIME CMD
root     27288 27245  0 11:57 pts/0    00:00:01 find . -type f
[root@oel6 ~]# kill -9 27288

[root@oel6 ~]# ls -l /proc/27288/stack
ls: cannot access /proc/27288/stack: No such file or directory

[root@oel6 ~]# ps -fp 27288
UID        PID  PPID  C STIME TTY          TIME CMD
[root@oel6 ~]#
```

进程被杀掉了，好了，问题解决 :)



注：文章没有翻译完，下面还有一段不是那么有意思的小工具和广告，没动力翻下去了 :P
