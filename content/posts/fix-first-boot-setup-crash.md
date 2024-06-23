---
title: "Fix First Boot Setup Crash"
date: 2019-05-31T15:41:49+08:00
draft: true
---





第一步安装好后 保存环境开始安装



> hostname: Name or service not known
> xauth: (stdin):1:  bad display name "localhost.localdomain:0" in "add" command
>
> X.Org X Server 1.19.6
> Release Date: 2017-12-20
> X Protocol Version 11, Revision 0
> Build Operating System: Linux 4.15.0-42deepin-generic x86_64 Debian
> Current Operating System: Linux localhost.localdomain 4.15.0-30deepin-generic #31 SMP Fri Nov 30 04:29:02 UTC 2018 x86_64
> Kernel command line: BOOT_IMAGE=/boot/vmlinuz-4.15.0-30deepin-generic root=UUID=53d6851c-77a8-47c8-9947-72df4f62f4ea ro splash quiet DEEPIN_GFXMODE=1,1152x864,1600x1200,1280x1024,1024x768
> Build Date: 12 May 2019  04:37:50AM
> xorg-server 2:1.19.6-2deepin (https://www.debian.org/support) 
> Current version of pixman: 0.34.0
> 	Before reporting problems, check http://wiki.x.org
> 	to make sure that you have the latest version.
> Markers: (--) probed, (**) from config file, (==) default setting,
> 	(++) from command line, (!!) notice, (II) informational,
> 	(WW) warning, (EE) error, (NI) not implemented, (??) unknown.
> (==) Log file: "/var/log/Xorg.0.log", Time: Fri May 31 07:49:33 2019
> (==) Using config directory: "/etc/X11/xorg.conf.d"
> (==) Using system config directory "/usr/share/X11/xorg.conf.d"
> VMware: No 3D enabled (0, Success).
> xinit: Unable to run program "/usr/bin/xterm": No such file or directory
> Specify a program on the command line or make sure that /usr/bin
> is in your path.
>
> xinit: connection to X server lost
>
> waiting for X server to shut down (II) Server terminated successfully (0). Closing log file.
>
> xauth: (argv):1:  bad display name "localhost.localdomain:0" in "remove" command

xeyes ->  No protocol specified



os/auth.c  CheckAuthorization



看看启动失败的时候 /var/run/lightdm/run 文件创建了没有。



env > /tmp/env_list

xauth list > /tmp/auth_list



libxcb 加打印



xtrace 



startx -> enable_xauth=0  可以成功



生成 cookie 创建 Xauthority  跟 lightdm 是等价的  

startx xinit lightdm 三者的关系



在 pkexec 中添加 xhost > /tmp/host_list

打印为空



display-setup-script -> xhost + &> /tmp/xhost.log



```
No protocol specified
xhost: unable to open display ":0"
```



仿佛被Xorg嘲讽：我都没同意你进来呢，你就敢把其他人放进来？



对比前后的日志，看到 xauth list 内容不一样

good -> localhost.localdomain  bad -> localhost



看看 lightdm 是怎么写这个 authority 文件的

根据 lightdm.log 中 Writing X server authority to  -> x-server-local.c write_authority_file

-> xserver_get_authority -> xserver_set_authority -> create_x_server -> 

x_authority_new_local_cookie  -> gethostname

- man Xorg
- man Xserver
- man Xsecutiry
- man xhost
- man xauth



localhost.localdomain vs localhost





