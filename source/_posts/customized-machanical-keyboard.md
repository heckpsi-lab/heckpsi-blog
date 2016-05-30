---
title: 一款祖传的机械键盘
date: 2016-05-14 16:31:15
tags: [键盘]
---

## 引

去年的时候就打算购买一款 60% 键盘以方便日常携带。时值开源哥在搞团购 Vim 布局的机械键盘的订购，就订了一把，结果由于一些原因，这单子被取消了。于是我就购买了一把 HHKB pro 2 静电容键盘。虽然说 HHKB 的键位设置和跳线的设计携带起来非常好，但是缺乏段落感的按键还是让人没有那种敲打起来的爽快感。于是我就打算客制化一把机械键盘来满足这个要求。

<!--more-->

## 硬件

如果要说 60% 的客制化套件的话，主要还是以 GH60 以及其延伸的一系列板子为主。开源哥之前推荐我使用 NerD60 的板子，可惜这块板子在国内并不容易买到。

轴体的选择使用了相对非常激进的 Cherry 绿轴，比起青轴，同样拥有二段式设计的绿轴的压力克数更高（clerk 声也更响），敲击感非常强，反馈非常明确，当然，对于害怕吵到其他人的话，这轴显然不是一个好选择。

键帽使用了三种颜色搭配的 PBT 无刻键帽，主要是因为现成的刻字键帽来说，显然是无法印刷修改非常严重的自定义按键的，而定制刻字的 PBT 键帽成本又太高，索性也就选择了无刻。

虽然阳极氧化铝的底座轻便又质感十足，通常是首选。不过既然作为一款码农世家祖传的键盘，最终选择了相对来说观感上更有冲击力的红花梨木底座。其余配置见下表：

| 类别   | 选料               |
| ---- | ---------------- |
| 主板   | GH60 (Rev.CHN)   |
| 轴体   | Cherry 绿轴        |
| 键帽   | PBT 无刻（红色、灰色、白色） |
| LED  | 白色               |
| 定位板  | 黑色阳极氧化铝上色        |
| 底座   | 红花梨木             |
| USB  | LINDY Mini-USB   |

由于我这个烙铁苦手，就上淘宝找了一家常州的 [厂家](https://yikewaishe.taobao.com/shop/view_shop.htm?user_number_id=134583372)（并没有广告关系，也没有肮脏的交易），对方当天就组装完发货，第二天一大早竟然就送到了。于是拿到手后就进入了修改固件，修改键位的过程了。

到手的货物长这样：

![](http://cdn.heckpsi.com/keyboard-2.jpg)

![](http://cdn.heckpsi.com/keyboard-1.jpg)

## 固件

客制化键盘的一大好处就是客制化的按键配列，这一点在 60% 和 40% 的键盘上非常重要。我结合我之前 HHKB 的使用经验，我就大致设计了一个如下的配列：

![](http://cdn.heckpsi.com/keyboard_5.png)

（其中左前方的按键为 Fn 组合键，右前方的是 Fn1 组合键）

然而客制化固件的刷入还有一些问题。首先这板子是 Rev.CHN 的，一上来刷入就失败了好几次。直到看到了大鹰的 [这篇文章](https://bigeagle.me/2015/07/gh60/)（没想到大鹰去年也参了开源哥的团，后来也从同一个卖家手里组装了键盘）。

当然这之前我找到了一个简便的选择 [TKG Keypad Generator](http://www.enjoyclick.org/tkg/#help)，它能自动生成 avr 单片机的 eep 文件，理论上可以直接刷入。但是我刷入之后无法正常引导键盘，所以还是需要使用 [tmk-keyboard-custom](https://github.com/kairyu/tmk_keyboard_custom) 自己编译刷入（`git clone` 的时候切记加入 `recursive` 参数，因为核心模块以 `submodule` 的形式出现在这个 `repository` 中）。不过还是可以使用 TKG Keypad Generator 所生成的 c 语言代码，稍作修改进行编译即可。

首先把生成的文件命名成 `keypad_xxx.c` 文件，`xxx` 为你配列自己的命名。不过直接刷入似乎会遇到一些编译问题，需要在文件后加入如下代码：

```c
#ifdef KEYMAP_IN_EEPROM_ENABLE
uint16_t keys_count(void) {
    return sizeof(keymaps) / sizeof(keymaps[0]) * MATRIX_ROWS * MATRIX_COLS;
}

uint16_t fn_actions_count(void) {
    return sizeof(fn_actions) / sizeof(fn_actions[0]);
}
#endif
```

然后按下键盘背后的 dfu 按钮，使用命令行指令刷入

```bash
make KEYPAD=xxx
make KEYPAD=xxx dfu
```

刷入后拔掉 USB 线，按住 BackSpace + Space，插入线缆，3 秒后放掉按键刷新 EEPROM 后，新配列即可工作。

然后我发现我设计的三个多媒体用途的键中，只有暂停键是可用的，上一首和下一首都不能正常工作，仔细研究后才发现 Mac 机器的上一首和下一首的 KeyCode 和 PC 略有区别，修改后重新编译了固件，按键都能正常工作了。

## 后

当然这块键盘有哪些不满意的地方呢，还是有的。一个问题是右上角的大按键如果拆成两个小按键，对于 `BackSpace` 和 `\` 的设计会变得方便更多，这个实在定位板的选择上出了一些问题。

另一个就是白色的 LED 灯，哪怕是最低亮度也能轻易地透过白色的按键却无法透过别的按键，这使得这个灯的开启变成了一个比较尴尬的问题。