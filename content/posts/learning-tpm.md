---
title: "Learning TPM"
date: 2026-09-05T19:55:00+08:00
draft: false
---

今天下午摸鱼，跟 Kimi 学了一下午 TPM。它设计的路子挺对我胃口：先简单讲讲概念，然后马上让我在虚拟机里动手——亲手 extend 一下 PCR、把秘密封进 TPM 再故意"搞破坏"看着它解不开，比啃十遍规范都管用。最后还真把 LUKS 全盘加密绑到 TPM 上了，开机不用输密码，舒服。

怕过阵子就忘干净，顺手让它把这一下午的过程整理成了一本小书：[《TPM 入门实战》](https://hualet.org/tpm_study/)（源码在 [GitHub](https://github.com/hualet/tpm_study)）。主要是给以后的自己翻的，有兴趣的朋友也欢迎看看、挑挑错。顶部菜单加了入口。
