---
title: "Things about kernel updates on deepin"
date: 2016-10-30T21:26:48+08:00
---

There were discussions about kernel updates on both deepin forum and deepin telegram group. Users are curious about why security updates are always being lag on deepin, and why there's no newer version kernel for so long. I explained it in telegram group, here I'll do it again in case someone's interested:

As we all know, deepin's maintaining kernel by ourselves now, based on version 4.4 LTS, our kernel gets patches by our kernel team, from Debian and also from Ubuntu (Yes, we do!). Basically it guarantees the kernel can work well on most deepin users' machines. After going through all these basics here're my explanations about why there're no newer versions:

* Newer versions from Debian are provided, (If you are really itching for a newer version) skilled users can install them without any trouble.
* We want our kernel team to focus on just one version, make it stable and well maintained.
* Newer kernels are not as stable as they seem to be, we can't take risks to ruin our users' machines by simply follow the upstream.
* The next LTS version is the best version to be the base of our next officially supported kernel.
* There'll be a 4.8 based official kernel version out soon.

Then why security updates are always being lag ? Well, the truth is I don't want them to be lag either, but deepin has a overwhelming system to guarantee updates are well tested, there's no single person can let the kernel updates go into our repository without carefully testings, so lags are made here. The system should be a good thing to protect our users, but it apparently doesn't work well with security updates. So improvements should be made to make security updates get out faster:
* More kernel test cases will be done by robots.
* Security updates will get higher priorities than normal updates.
* More improvements under discussion...

That's all, and feel free to contact me if you're having trouble with any of these things.
