---
title: "Got a PC"
date: 2019-06-18T20:00:08+08:00
---

从 15 年初购入人生第一台 mbp 之后，再也没有使用过 PC 了。最近趁着 618 组了一台电脑，平时的主力机器是 mbp，然后家里放了台 mac mini，但是最近为了在家里跑一些深度学习的模型，而 mac 的显卡实在是让人一言难尽。另外，由于苹果最近和英伟达开始了”战争”，而英伟达又是搞深度学习绕不过去的，万般无奈，只好自行装了台机器。

可以说，我是为了张显卡装了台机器。。。开始的预算是 8000 块，最后大概超出预算 30 块样子吧。下面简单记录一下装机组件列表与一些注意点。

## 组件

### CPU+主板

cpu 和主板一定要配套买，最好不要分开买。另外 cpu 和主板要先选好，要注意看主板的接口。我买的是 i7 8700 + Z370 套装。其中 i7 8700 cpu 是 6 核，3.2G 主频，12 线程。3370 主板支持 4 个 DDR4 插槽，3 个显卡槽。

注意，i7 8700 可以使用自带的 cpu 风扇，如果买其他型号的 cpu 可能需要升级一下 cpu 风扇。

### 机箱

机箱不能太小，第一是装机的时候方便，第二是考虑以后的可扩展性。除此之外，机箱是否静音也比较重要。我买的是先马黑洞，中塔机箱，内置 3 个风扇。

### 电源

一定要 600W 以上的。电源大小要看好，确保能放入机箱内，如果机箱比较大的话，一般都没什么问题。我买的是 silverstone 的 ET650-G,额定功率 650W。

### 显卡

毕竟是为了显卡才装的机器，显卡是关键。参考 Tim Dettmers 的[这篇文章](https://timdettmers.com/2020/09/07/which-gpu-for-deep-learning/)，如果没有时间的话可以看他最后写的 TL,DR。如果还没有时间并且有 3200 块预算的话，先买个 RTX2070 吧。我买的是索泰 RTX2070，8G 显存，2304 个 CUDA 核心。京东价格在 3200 到 3400 间吧。

### SSD 和 内存

SSD 1T，闭着眼睛随便买。我买的是海康威视的 1T SSD，不到 800 块。内存 16G 单条，闭着眼睛随便买（事实上还是要看下接口协议啥的）。我买的是金士顿 DDR4 3200。

### 其他

键盘鼠标我通常都用罗技的蓝牙键盘，但是家里最好要有有线键盘和鼠标（bios 设置的时候要用）。显示器的话随便买吧，平时我都是从 mac ssh 上去，基本用不太到，偶尔 mac 没带回家的时候，才会接上显示器。

另外，装机前要先买好一套螺丝刀什么的。

## 系统

我安装的是 ubuntu1904，装完就有点后悔，当前 1904 还不是很稳定，应该保守一点先装 18 系的，不过不得不说，1904 确实蛮好看的。装完后需要安装显卡驱动，更换各种源等等。

## 别的解决方案

如果你或者你的朋友也出于和我一样的目的想要装一台机器，欢迎直接使用我的 list。而除了这种方案外，还有一种方案是购置一款显卡坞（一般在 2000 到 3000 之间可以搞定），然后再买块显卡，就搞定啦。但是需要注意，如果你正在使用 macos，高版本的 macos 是装不了 N 卡驱动的，你需要维持低版本的 macos 系统，或者选择 amd 的显卡，或者离开苹果阵营。由于这三点我都做不到，所以我只好自己装一台了。

## benchmark

完成装机后，我简单 benckmark 了一下。使用相同版本的 tensorflow 和相同参数的 lenet-5 模型，处理相同的 Mnist 识别问题，这台新机器大概是我的 mbp-15 的 9 倍。