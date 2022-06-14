---
title: "如何设置键盘映射"
date: 2022-06-13T13:35:40+08:00
tags: ["键盘映射", "xmodmap", "windows", "linux"]
categories: ["生产力"]
---

根据自己个人习惯，把键盘上一些键位置映射到其他地方是可以明显提高生产力的。

我的习惯是左手的大拇指按ctrl，右手大拇指按alt，然后CapsLock对我来说是常用键，所以这个键是完完全全不能换的。

所以现在在普通键盘上我的目标是把空格左边的第一个键换成ctrl，右边的第一个键换成alt，如果这些键是其他的键，再把原本的ctrl和alt键映射过来。

## Windows
微软官方的[PowerToys](https://docs.microsoft.com/en-us/windows/powertoys/) 可以设定键盘映射。没什么可说的，安装打开之后里面有个键盘管理器，里面就可以很方便地映射按键了。

微软真赛过我的亲爹娘！！！

## Mac
- [ ] 我没有mac，等什么时候公司送我一台再把这里补充上。

## Linux
重头戏来了。

linux里只要有桌面系统一般都会安装 `xmodmap`。 没有也没关系，根据关键字直接在包管理器里安装一个就可以了。

先在`$HOME`下建一个`.XmodMap`文件

```bash
touch ~/.Xmodmap
```

里面就需要写我面的个人键盘映射文件了，问题是怎么写呢？

### mod
使用`xmodmap -pm`可以查看当前的映射mod，一般长这样：

```
shift       Shift_L (0x32),  Shift_R (0x3e)
lock        Caps_Lock (0x42)
control     Control_L (0x40),  Control_R (0x69)
mod1        Alt_L (0x25),  Alt_R (0x6c),  Alt_L (0xcc),  Meta_L (0xcd)
mod2        Num_Lock (0x4d)
mod3      
mod4        Super_L (0x85),  Super_R (0x86),  Super_L (0xce),  Hyper_L (0xcf)
mod5        ISO_Level3_Shift (0x5c),  Mode_switch (0xcb)
```

前三个非常好理解，就是 shift, CapsLock 和 ctrl 键，mod1 指的是按下 alt 键的状态，  meta 键一般应额是映射在这边。 mod2 指的是数字小键盘开关 NumLock 键， mod4 对应super键，剩下两个不知道。

我面现在需要的是将键盘左边的 ctrl 和 alt 键调换，所以在设置映射之前需要把 control 和 mod1 这两个 mod 关掉，然后完成之后打开。

### 键位编号

接着我们看如何设定键位映射。使用 `xmodmap -pke` 可以看到键盘上所有映射，我面找到 Control_L 和 Alt_L 所对应的键：
```
keycode  37 = Alt_L Meta_L Alt_L Meta_L
keycode  64 = Control_L NoSymbol Control_L
```

可以看出来 37 是左 alt 键， 64 是左 ctrl 键盘。

**如果你不知道键盘对应的编号的时候，使用 `xev` 可以查看所有键的编号。**

### 文件

好了，我们根据上面的思路来补全 `.XmodMap` 文件
```
clear control
clear mod1
keycode 37 = Alt_L Meta_L
keycode 64 = Control_L
add control = Control_L Control_R
add mod1 = Alt_L Meta_L Alt_R
```

之后运行
```bash
xmodmap ~/.Xmodmap
```
就可以看到效果了。

### 能不能再简单一点啊，教练

当然可以，可以设置 `XKBOPTIONS` 的行为

把 `XKBOPTIONS="ctrl:nocaps"` 加到 `/etc/default/keyboard` 文件下就可以了，另外可以在 `/usr/share/X11/xkb/rules/base.lst` 里查看特定 option 的写法。

此外如果不想修改文件直接一句 `setxkbmap -option ctrl:swap_lalt_lctl` 也可以达到类似的效果。

### 蓝牙重连时启用

但是现在仍然有一个蛋疼的问题，我的蓝牙键盘断开再重连之后映射就消失了。这是由于xmodmap不支持对新设备的映射\[[*](https://bugs.freedesktop.org/show_bug.cgi?id=25262)\]。也就是说每次重新连接蓝牙键盘之后我需要重新映射一遍。这好吗？这不好。

所以现在想办法要对所有新设备都可以映射。

- [ ] 目前没有找到什么好方法，等什么时候搞定了再来完成这部分。我的系统比较老了，说不定新系统都有类似的功能了。