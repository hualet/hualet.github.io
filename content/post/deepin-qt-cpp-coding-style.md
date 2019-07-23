---
title: "deepin Qt/C++ 代码风格指南"
date: 2019-07-23T19:17:06+08:00
draft: true
---

本代码风格为深度科技Qt/C++代码风格规范，主要是在[Qt Coding Style](https://wiki.qt.io/Qt_Coding_Style)上进行删减和修改。

## 代码缩进

- 使用4个空格进行缩进；
- 禁止使用Tab进行缩进。

## 声明变量

- 不同的变量声明不要放在同一行；
- 变量尽量起有意义的变量名；
- 单字符变量尽量避免，且只能在局部变量和临时变量处使用；
- 需要变量时再去定义变量（对比C语言在头部声明所有变量）；

```
 // 错误写法
 int a, b;
 char *c, *d;

 // 正确写法
 int height;
 int width;
 char *nameOfThis;
 char *nameOfThat;
```

- 变量名和函数名使用小写开头，命名规范遵循驼峰式命名规范；

```
 // 错误写法
 short Cntr;
 char ITEM_DELIM = ' ';

 // 正确写法
 short counter;
 char itemDelimiter = ' ';
```

- 类名以大写开头，命名规范遵循驼峰式命名规范。

## 空格

- 在代码段落之间使用空行隔开；
- 任何地方的空行不要超过两行；
- 关键词后和大括号前均需空格:

```
 // 错误写法
 if(foo){
 }

 // 正确写法
 if (foo) {
 }
```

- 对于指针和引用，*和&需要和变量名紧挨:

```
 char *x;
 const QString &myString;
 const char * const y = "hello";
```

- 二元操作符两侧需有空格；
- 尽量避免C风格的类型转换：

```
 // Wrong
 char* blockOfMemory = (char* ) malloc(data.size());

 // Correct
 char *blockOfMemory = reinterpret_cast<char *>(malloc(data.size()));
```

- 不要硬把多行代码放在同一行；
- 逻辑控制操作的执行体需换行:

```
 // 错误写法
 if (foo) bar();

 // 正确写法
 if (foo)
     bar();
```

## 大括号

- 大括号跟逻辑控制关键字（如if、switch等）放在同一行：

```
 // 错误写法
 if (codec)
 {
 }
 else
 {
 }

 // 正确写法
 if (codec) {
 } else {
 }
```

- 特殊情况：函数实现（非lambda函数）和类声明时大括号都开新行写：

```
 static void foo(int g)
 {
     qDebug("foo: %i", g);
 }

 class Moo
 {
 };
```

## switch语句

- case和switch关键字对齐（包括default）；
- 每个case处理体都需要显式使用break或者使用 Q_FALLTHROUGH()以表示刻意不进行break，除非两个case可以直接合并在一起进行处理：.

```
 switch (myEnum) {
 case Value1:
   doSomething();
   break;
 case Value2:
 case Value3:
   doSomethingElse();
   Q_FALLTHROUGH();
 default:
   defaultHandling();
   break;
 }
```

## 换行和折行

- 代码一行的内容不要太长，保证在100个字符以内，否则进行折行处理；
- 注释同上；.
- 逗号写在折行末尾，操作符写在折行开始：

```
 // Wrong
 if (longExpression +
     otherLongExpression +
     otherOtherLongExpression) {
 }

 // Correct
 if (longExpression
     + otherLongExpression
     + otherOtherLongExpression) {
 }
```

## 配置插件

1. 在QtCreator中启用Beautifier插件：

![Enable Beautifier](//img/2019/07/enable-beautifier.png)

2. 在QtCreator设置中配置Beautifier插件风格规则：

![Configure Beautifier](//img/2019/07/enable-beautifier-configuration.png)

```
# .astylerc 文件内容
--style=kr 
--indent=spaces=4 
--align-pointer=name 
--align-reference=name 
--convert-tabs 
--attach-namespaces
--max-code-length=100 
--max-instatement-indent=120 
--pad-header
--pad-oper
```
3. 设置Beautifier保存文件时自动风格化：

![Format on save](//img/2019/07/enable-format-on-save.png)