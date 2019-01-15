---
title: "WPS中的字体名"
date: 2019-01-14T18:31:08+08:00
---

今天又被  `fontconfig` 坑到了……仔细想想，半年到一年前我还对 `fontconfig`  狗屁不通呢，而现在我已经被 `fontconfit` 坑了有几次了，这印证了一个真理：“只要你足够迟钝，世界就是美好的，一旦你有了某种能力，麻烦就会找到你”——这只是个玩笑，事实上不管你有没有能力，现实都不会让你好过 😈

在这些坑里面，有两个是跟今天主题相关的，也就是关于WPS中字体名称的两个问题：

- WPS 字体列表中中文字体在中文环境下不显示中文名（比如系统里面有宋体，列表里面显示 `SimSun`）；
- WPS 设置段落字体（中文字体）后，工具栏显示的字体为英文名（比如你设置一段文字是宋体，但是标题栏显示的仍然是`SimSun`）

这两个问题有点像，但是却不是同一个问题（也不只是WPS有问题，而是WPS对字体的需求比较多）：

第一个问题原先是 [论坛用户报告](https://bbs.deepin.org/forum.php?mod=viewthread&tid=166999) 的（实际上刘老大也报过，不过被自动忽略了😂），论坛用户又报过来以后，大家看用户的报告太详细了，感动得一塌糊涂，顿时解决问题的动力都有了。我也折腾了大半天，不过脑袋里面一直有一个同事的声音：那是字体有问题！我也挺赞同的：盗版的字体，不靠谱很正常……所以，我就考虑能不能给字体做一些别名啥的，比如 `SimSun` 就叫 `宋体`……当然了最后就是玩了一下 `fontconfig` 的配置以后，默默就放弃了。

最后经过 [@felixonmars](https://github.com/felixonmars) 同学的排查，发现是上游的一个 [bug](https://bugs.freedesktop.org/show_bug.cgi?id=105756) ，而且已经修复，所以赶紧更了一波，解决了这个问题。

本以为皆大欢喜了，今天又有客户报过来 bug，说 Windows 上写好的文档，调整好的字体，跑到deepin下字体就不对了，废了老大的劲儿才又调好，其中就提到了选择方正字库里面的字体，显示的不是中文字体名称的问题。

怎么说呢，幸亏我脑子不好，不记得之前已经解决过中文字体名显示为英文的问题，要不然得跟客户和老板掰扯一会儿，我只记得字体的中英文名字好像是有点问题，所以默默赶紧去看了一下，还真真的有问题。

因为 DDE 并没有设置过多的 `fontconfig` 配置，所以我相信这不是 DDE 的问题，但是又觉得对于一个文字处理软件来说，如果这么重要的功能都有 BUG，我真不信 WPS 的人还能坐得住，所以就想试一下在 Ubuntu 下 WPS 的这个行为是否正确。刚好年前 ElementryOS 发新版的时候尝鲜装了一个在测试机器上，所以很快切进去下了一个 WPS 安装上，从一个正版的网站上下载了一个盗版的宋体，试了一下，果然踏马的是好的……

理性的我用事实证明了感性的我是错误的，气氛一度很尴尬。

更让人尴尬的是，我也不知道这是怎么回事。但是，无巧不成书，就在这种尴尬氛围的笼罩下，我鬼使神差地在 ElementryOS （太长了，下面简称 EOS ）上运行了 `fc-match 宋体` 这个命令，而且很神奇的发现输出的 `simsun.ttf: "宋体" "Regular"` 跟 deepin 下同样命令输出的 `simsun.ttf: "SimSun" "Regular"` 不太一样，这可把我乐坏了——要知道一个稍微复杂点的图形软件，打开 `fontconfig` 的调试以后，输出就是很可怕的，更别说 WPS！别问我是怎么知道的，回忆起来都是泪。但是现在使用 `fc-match` 就能重现的问题（我打心底认为这俩是同一个问题），调试起来就比较简单了。

`fontconfig` 本身为了方便程序调试，内建了一些调试项，可以通过环境变量 `FC_DEBUG` 控制：

```
Name         Value    Meaning
  ---------------------------------------------------------
  MATCH            1    Brief information about font matching
  MATCHV           2    Extensive font matching information
  EDIT             4    Monitor match/test/edit execution
  FONTSET          8    Track loading of font information at startup
  CACHE           16    Watch cache files being written
  CACHEV          32    Extensive cache file writing information
  PARSE           64    (no longer in use)
  SCAN           128    Watch font files being scanned to build caches
  SCANV          256    Verbose font file scanning information
  MEMORY         512    Monitor fontconfig memory usage
  CONFIG        1024    Monitor which config files are loaded
  LANGSET       2048    Dump char sets used to construct lang values
  MATCH2        4096    Display font-matching transformation in patterns
```

我们只需要最简略的匹配信息就可以了，所以：

```bash
$ FC_DEBUG=1 fc-match 宋体
FC_DEBUG=1
Match Pattern has 25 elts (size 32)
	family: "宋体"(s) "Noto Sans CJK SC"(s) "Noto Sans"(s)
	familylang: "en"(s) "en-us"(w)
	stylelang: "en"(s) "en-us"(w)
	fullnamelang: "en"(s) "en-us"(w)
	slant: 0(i)(s)
	weight: 80(i)(s)
	width: 100(i)(s)
	size: 12(f)(s)
	pixelsize: 12.5(f)(s)
	hintstyle: 1(i)(w)
	hinting: True(s)
	verticallayout: False(s)
	autohint: False(s)
	globaladvance: True(s)
	dpi: 75(f)(s)
	scale: 1(f)(s)
	lang: "zh-CN"(w)
	fontversion: 2147483647(i)(s)
	embeddedbitmap: True(s)
	decorative: False(s)
	lcdfilter: 1(i)(w)
	namelang: "en"(w)
	prgname: "fc-match"(s)
	symbol: False(s)
	variable: False(s)

Best score 0 0 0 0 0 0 0 0 0 0 1e+99 0 0 0 0 0 0 0 0 0 0 0 0 0 2.14735e+12
Pattern has 25 elts (size 25)
	family: "SimSun"(w) "宋体"(w)
	familylang: "en"(w) "zh-cn"(w)
	style: "Regular"(w)
	stylelang: "en"(w)
	fullname: "SimSun"(w) "宋体"(w)
	fullnamelang: "en"(w) "zh-cn"(w)
	slant: 0(i)(w)
	weight: 80(f)(w)
	width: 100(f)(w)
	spacing: 90(i)(w)
	foundry: "ZYEC"(w)
	file: "/home/hualet/.fonts/simsun.ttf"(w)
	index: 0(i)(w)
	outline: True(w)
	scalable: True(w)
	charset: 
	0000: 00000000 ffffffff ffffffff ffffffff 00000000 ffffffff ffffffff ffffffff
	0001: 08080002 00000800 000c2110 01000803 00040000 00000000 15554000 00000000
	0002: 00000000 00000000 00020000 00000002 00000000 00000000 12000ec0 00000000
	0003: 00000000 00000000 00000000 00000000 fffe0000 fffe03fb 000003fb 00000000
	0004: ffff0002 ffffffff 0002ffff 00000000 00000000 00000000 00000000 00000000
	0020: 77790000 0e2d0067 00000000 00000000 00000000 00001000 00000000 00000000
	0021: 00400228 00000006 00000000 03ff0fff 03cf0000 00000000 00000000 00000000
	0022: e4228100 20f04fa9 00041100 0000c0f3 02200000 80000020 00000000 00000000
	0023: 00040000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
	0024: 00000000 00000000 00000000 fff003ff 0fffffff 00000000 00000000 00000000
	0025: ffffffff ffffffff ffff0fff 000fffff 0038fffe 300c0003 0000c8c0 0000003c
	0026: 00000260 00000000 00000005 00000000 00000000 00000000 00000000 00000000
	0030: 60ffffef 000003fe fffffffe ffffffff 780fffff fffffffe ffffffff 707fffff
	0031: ffffffe0 000003ff 00000000 00000000 00000000 00000000 00000000 00000000
	0032: 00000000 000203ff 00000000 00000000 00000000 00000008 00000000 00000000
	0033: 00000000 00000000 00000000 00000000 7000c000 00000002 00264010 00000000
	004e: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	004f: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0050: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0051: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0052: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0053: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0054: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0055: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0056: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0057: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0058: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0059: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	005a: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	005b: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	005c: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	005d: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	005e: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	005f: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0060: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0061: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0062: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0063: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0064: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0065: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0066: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0067: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0068: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0069: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	006a: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	006b: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	006c: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	006d: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	006e: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	006f: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0070: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0071: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0072: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0073: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0074: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0075: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0076: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0077: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0078: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0079: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	007a: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	007b: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	007c: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	007d: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	007e: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	007f: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0080: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0081: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0082: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0083: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0084: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0085: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0086: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0087: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0088: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0089: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	008a: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	008b: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	008c: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	008d: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	008e: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	008f: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0090: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0091: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0092: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0093: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0094: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0095: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0096: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0097: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0098: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	0099: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	009a: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	009b: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	009c: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	009d: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	009e: ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff ffffffff
	009f: ffffffff ffffffff ffffffff ffffffff ffffffff 0000003f 00000000 00000000
	00e7: 00000000 00000000 00000000 00000000 00000000 00000000 00000180 000fff80
	00e8: ffe00000 ffffffff ffffffff 0000001f 00000000 00000000 00000000 00000000
	00f9: 00000000 00001000 00000000 02000000 00200000 00000000 00000000 00020080
	00fa: 811af000 0000039b 00000000 00000000 00000000 00000000 00000000 00000000
	00fe: 00000000 fffb0000 fef7fe1f 00000f7f 00000000 00000000 00000000 00000000
	00ff: fffffffe ffffffff 7fffffff 00000000 00000000 00000000 00000000 0000003f
(w)
	lang: aa|ay|bg|bi|br|ch|co|da|de|en|es|eu|fj|fo|fr|fur|fy|gd|gl|gv|ho|ia|id|ie|io|is|it|kum|lb|mg|nb|nds|nl|nn|no|nr|nso|oc|om|os|pt|rm|ru|sel|sma|smj|so|sq|ss|st|sv|sw|tl|tn|ts|uz|vo|wa|xh|yap|zh-cn|zh-sg|zu|an|fil|ht|jv|kj|kwm|li|ms|ng|pap-an|pap-aw|rn|rw|sc|sg|sn|su|za(w)
	fontversion: 131072(i)(w)
	capability: "otlayout:hani"(w)
	fontformat: "TrueType"(w)
	decorative: False(w)
	postscriptname: "SimSun"(w)
	color: False(w)
	symbol: False(w)
	variable: False(w)

simsun.ttf: "SimSun" "Regular"

```

因为以前调 `fontconfig` 的经验，我模模糊糊知道那个 `familylang` 是控制 `family` 也就是字体名显示的语言的，所以看到 

```bash
familylang: "en"(s) "en-us"(w)
```

这行以后，大致能确定应该是系统某个配置出了问题让 `fontconfig` 误以为语言环境是英文。看了一下环境变量，没发现哪个是设置英文的，所以只能去看代码了。

下了 `fontconfig` 的源码，搜了一下关键字 `familylang` ，猜测这个属性应该是通过操纵 `FC_FAMILYLANG_OBJECT` ，再次搜索：

```
 $ rg -i FC_FAMILYLANG_OBJECT
src/fcmatch.c
380:	case FC_FAMILYLANG_OBJECT:
551:	if (fe->object == FC_FAMILYLANG_OBJECT ||
565:	    FC_ASSERT_STATIC ((FC_FAMILY_OBJECT + 1) == FC_FAMILYLANG_OBJECT);
695:	    pe->object != FC_FAMILYLANG_OBJECT &&

src/fclist.c
430:		familyidx = FcGetDefaultObjectLangIndex (font, FC_FAMILYLANG_OBJECT, lang);

src/fcfreetype.c
1671:	    while (FcPatternObjectGetString (pat, FC_FAMILYLANG_OBJECT, n, &familylang) == FcResultMatch)

src/fcdefault.c
316:    if (!FcPatternFindObjectIter (pattern, &iter, FC_FAMILYLANG_OBJECT))
318:	FcPatternObjectAdd (pattern, FC_FAMILYLANG_OBJECT, namelang, FcTrue);
319:	FcPatternObjectAddWithBinding (pattern, FC_FAMILYLANG_OBJECT, v2, FcValueBindingWeak, FcTrue);

```

发现其中只有 `fcdefault.c` 里面有执行 `FcPatternObjectAdd` 这个操作像是在修改这个属性，其他地方明显都只是读取，刚好 `fc-match/fc-match.c` 里面也有调用这个函数，所以仔细看了一下这个函数，从逻辑上看意思是 `familylang` 如果没有设置，就会从 `namelang` 属性继承，然后我忽然发现 `fc-match` 的帮助里面写得位置参数概念上叫 `pattern` ，难道就对应 `FcPattern`，那它应该也可以指定 `familylang` 咯？网上搜寻了一番，发现可以 `fc-match 宋体:familylang=zh-cn`，果然输出正常的中文名。这算是一个小插曲吧。

接着 `namelang` 属性说，这个属性是从一个叫 `FcGetDefaultLang` 的函数处获取，我心想总算是要到头了，然而发现从函数的实现来看，这个函数会从 `FC_LANG`、`LC_ALL`、`LC_CTYPE`和`LANG`这些环境变量里面挨个读取默认的语言（后来发现直接 `man FcGetDefaultlangs` 就可以看到），看起来没毛病呀。

写了个测试程序：

```c
#include <stdlib.h>
#include <stdio.h>

#include <fontconfig/fontconfig.h>

int main(int argc, char const *argv[])
{
    FcStrSet *langs = NULL;

    langs = FcGetDefaultLangs();
    if (langs)
    {
        FcChar8 *lang = NULL;
        FcStrList *lang_list = NULL;

        lang_list = FcStrListCreate(langs);
        FcStrListFirst(lang_list);

        while(lang = FcStrListNext(lang_list)) {
            printf("%s", lang);
        }
    }

    return 0;
}
```

编译执行，输出 `zh-CN` ，666

中间尝试了使用 `gdb` 在 `fc-match.c`中调用 `FcDefaultSubstitute` 的部分加断点，但是死活无法跳入函数体里面，直接在函数里面响应的行数处断点，直接无效……撞鬼了

没办法了，只能改代码了，找了找代码里面的 log 模块叫 `fcdbg.c` ，我本来还说从 `FcPattern`  里面读取属性有点麻烦，又不熟悉代码，但是意外地找到一个 `FcPatternPrint` ，这下连继续研究下去的动力都没有了，赶紧加代码打印，分别在 `FcConfigSubstitute` 和 `FcDefaultSubstitute` 前后加了打印看对 `pat` 的代码：

```
	FcPatternPrint(pat);
    FcConfigSubstitute (0, pat, FcMatchPattern);
	FcPatternPrint(pat);
    FcDefaultSubstitute (pat);
	FcPatternPrint(pat);
```

输出：

```bash
$ ./fc-match 宋体
Pattern has 1 elts (size 16)
	family: "宋体"(s)

Pattern has 6 elts (size 16)
	family: "宋体"(s) "Noto Sans CJK SC"(s) "Noto Sans"(s)
	hintstyle: 1(i)(w)
	lang: "zh-CN"(w)
	lcdfilter: 1(i)(w)
	namelang: "en"(w)
	prgname: "fc-match"(s)

Pattern has 25 elts (size 32)
	family: "宋体"(s) "Noto Sans CJK SC"(s) "Noto Sans"(s)
	familylang: "en"(s) "en-us"(w)
	stylelang: "en"(s) "en-us"(w)
	fullnamelang: "en"(s) "en-us"(w)
	slant: 0(i)(s)
	weight: 80(i)(s)
	width: 100(i)(s)
	size: 12(f)(s)
	pixelsize: 12.5(f)(s)
	hintstyle: 1(i)(w)
	hinting: True(s)
	verticallayout: False(s)
	autohint: False(s)
	globaladvance: True(s)
	dpi: 75(f)(s)
	scale: 1(f)(s)
	lang: "zh-CN"(w)
	fontversion: 2147483647(i)(s)
	embeddedbitmap: True(s)
	decorative: False(s)
	lcdfilter: 1(i)(w)
	namelang: "en"(w)
	prgname: "fc-match"(s)
	symbol: False(s)
	variable: False(s)

simsun.ttf: "SimSun" "Regular"
```

可以确认是 `FcConfigSubstitute` 函数里面给 `pat` 设置了 `en` 的 `namelang`，那之后在 ``FcDefaultSubstitute`` 中 `familylang`  就从 `namelang` 继承了 `en` ，又加上了 `en-us` ，变成了我们看到的 `"en"，"en-us"` 。看函数名， ``FcConfigSubstitute`` 应该就是读取了配置文件，然后做了什么修改，至于怎么改的我也不知道。

这时候我偷懒症犯了，没有去看代码了（看着略复杂），我发现新版 `fontconfig` 多了一个工具叫 `fc-conflist` 能列出来 `fontconfig` 加载的所有配置文件列表，通过这个命令我发现系统里面的配置文件主要放在 `/etc/fonts` 和 `/usr/share/fontconfig` ，然后就用 `rg` 去里面搜关键字 `lang` 了，排除其中 `<test>` 的行，还真发现一个叫 `/usr/share/conf.avail/15-assign-lang-en.conf` 的文件里面有 `edit` `namelang` 这个属性，在此之前，我都不知道还有 `edit` 这种操作……我也更不信这个问题会出在 `fontconfig` 的配置上。

```bash
$ dpkg -S conf.avail/15-assign-lang-en.conf
deepin-default-settings: /usr/share/fontconfig/conf.avail/15-assign-lang-en.conf
```

事实证明，这个配置文件还真是 deepin 提供的……

本来想搞清楚之前为什么要做这个的，但是提交信息太简单了，没有提供相关线索，猜测应该是以前有些字体名称会乱码——字体名称需要特殊字体的支持，然而系统（终端）的字体不支持这种字体的字体名，为了解决乱码，默认使用英文输出字体名称 😅 

不过有了 Noto 这种问题应该就很少了，所以狠心删掉了这个配置：[提交](https://github.com/linuxdeepin/default-settings/commit/34ce7928e70f082b02a7da7e60a9ddaaf7a036f4)。



结合这两个问题，猜测 WPS 字体设置分两个部分，一部分是从系统获取字体列表，这个部分是从 Qt 拿的，用的 Qt 提供的字体名（中文），第二部分是从这个字体名拿到系统中匹配度最高的字体，应该是通过 `libfontconfig` 直接做得（deepin下英文），然后再把这个字体设置给实际的文档，工具栏中显示的也就是这个名字。总之，人家又不开源，瞎猜一下图个开心咯。