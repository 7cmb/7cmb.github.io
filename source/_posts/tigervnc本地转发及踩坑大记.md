---
title: TigerVNC本地及踩坑大记
date: 2024-01-06 2:35:14
tags:
 - linux
 - de
 - vnc
 - 折腾
categories:
 - [ linux, vnc]
 - [ linux, de]
 - [ 折腾, linux]
---

# 概述

多电脑协同工作总是困难的，但是做远程却很简单。vnc另开一个桌面相对来说是是性价比很好的方案。但是笔者使用的wm即使在本地不同tty开多个x会话都无法正常工作，抱着试试的心态部署了vnc,果然无法正常使用。遂在archwiki上发现tigervnc的`x0vncserver`[本地转发功能](https://wiki.archlinuxcn.org/wiki/TigerVNC#%E8%BF%90%E8%A1%8C_x0vncserver_%E6%9D%A5%E7%9B%B4%E6%8E%A5%E6%8E%A7%E5%88%B6%E6%9C%AC%E5%9C%B0%E6%98%BE%E7%A4%BA%E5%86%85%E5%AE%B9)，看到这功能我啪一下就跳起来了，这不正合我意吗，四舍五入等于开箱即用。没想到这是踩坑的开始.....

<br>

装好tigervnc，配置vnc好密码，直接启动`x0vncserver`，直接就能在vnc客户端连接上，一切都很顺利。然而快速写完脚本添加desktop entry扔到rofi里后台运行，却发现事情并没有这么简单。有几点问题是需要解决的:

- 首先是屏幕问题，oled远程期间屏幕长期开着并不好
- 其次是flameshot的dbus问题
- 最后的问题和第二个问题是关联的，就是某些软件无法捕获到x窗口的变化

最后一个问题具体表现为x的vnc转发、flameshot截图、还有rofi的透明效果背景在输入法或者某些窗口触发时，**只要碰到屏幕边缘就会让以上三个软件画面定格在某个画面**，只要lightdm锁屏后或者suspend后再回到桌面就又恢复正常状态，直到再次触发

前两个问题其实已经很久了，但是在本地却影响并不大，遇到要使用flameshot的情况锁屏解锁就好。但是对于x0vncserver，这会伴随着持续性屏幕闪烁、定格的情况，这意味着一旦问题复现，机器不能在物理上操作或ssh将无法解决问题

<br>

<br>

-----

<br>

就此打住，先谈一下配置过程中的小插曲，首先是屏幕。在远程的时候我希望屏幕时保持关闭，或者说尽管无法做到把电断掉，也要保持最低亮度甚至黑屏
在阅读相关文档之后，发现相关组件可以控制屏幕电源管理的行为，首先是安装`acpi`后自带的`dpms`工具，还有就是可以控制dpms行为的xorg的工具`xset`。了解了相关信息之后发现想法是可行的，但是在深入了解后，发现显示屏又有三种节能状态(也就是关闭的状态)，按节能程度依次到高来排序分别是"standby"、"suspend"和"off"

> 一开始还以为是关于系统睡眠/休眠的描述
> 
> 直到找到了这篇文章:
> 
> [Battery Powered Linux Mini-HOWTO DPMS](https://tldp.org/HOWTO/Battery-Powered/displaytypes.html)

而经过实践，除非关闭dpms，否则无论dpms将屏幕设置为上述三种节能状态的哪种，将触发lightdm的锁定。但是关闭dpms之后再利用dpms关闭屏幕，dpms又会再次开启。dpms自带定时将屏幕设置为节能状态(类似windows的自动熄屏)的功能，如果此时节能状态打开了，lightdm锁定将再次触发

当x0vncserver正在运行时，触发lightdm的锁定，vnc客户端将变成黑屏。如果操作物理机激活屏幕试图解锁，将会发现物理机屏幕会黑屏并且左上角有一条下划线快速闪烁而无论任何操作都无法进入登录界面(包括切换tty)

再者考虑到笔者使用的是设备是二合一设备，设置屏幕上述三种节能状态并不是一个保险的选项。即使dpms将屏幕关闭后再关闭dpms，也有例外情况----误触屏幕导致屏幕被点亮。关闭屏幕的手段是dpms，dpms将再次开启，开启了dpms将定时触发节能状态。这时就又可能会导致上面的状态

对此，笔者的解决方法是**放弃**dpms关闭屏幕；相对的是把屏幕亮度调节为最低(效果和关闭是一样的)，再用xset工具控制dpms设置屏幕为永不超时，并且用xet设置lightdm永不自动锁定。这时dpms虽然是开启的，却永远不会自动触发节能状态

综上所述，做这一切就是为了把屏幕关闭并且不触发lightdm的锁屏

# 配置x的vnc转发并解决屏幕问题

为了解决这个问题，写了两个脚本，并且把他们加到desktop entries以实现rofi里能快速启动脚本并后台运行
运行x的vnc转发`vnc.sh`:

```bash
#!/usr/bin/bash

# 将屏幕亮度调到最低
light -U 100

# 关闭自动锁屏
xset s 0 0

# 将节能状态的触发时间改为永不
xset dpms 0 0 0

# 关闭dpms功能，防止一次"主动"触发节能模式
xset -dpms

# 开启主程序
x0vncserver -rfbauth ~/.vnc/passwd -rfbport 5901 -Geometry 3840x2160
```

关闭转发`shutX0vncserver.sh`:

```bash
 #!/usr/bin/bash

# 将亮度调到人能看到的状态
light -A 10

# 设置下次10分钟不操作后锁屏，并且每次10分钟不操作后锁屏
xset s 600 600

# 设置节能状态超时时间
xset dpms 0 0 600

# 给x0vncserver及相关进程组发送正常关闭信号
kill -SIGTERM "-$( ps jx | sed -n '/\/home\/baka\/commands\/vnc.sh/p' | awk '{print $3}'| head -n 1)"
```

# 解决flameshot的dbus问题

之后x0vncserver转发中出现了导致无法使用的情况。因为状况相似，且几乎同时发生，我第一个反应就是flameshot在某种情况下无法捕获到x窗口的变动直到下次重新登录到wm。即使启动wm中用的是`dbus-launch [wm-session]`，问题也经常复现。具体表现为截图开启后捕获的不是实时画面，而是该bug出现时的那个瞬间，伴随还有rofi本该透明的地方也会定格在那个画面

根据flameshot的Troubleshooting，这是dbus没有正确配置导致的。之后顺着文档顺便配置好了dbus上的桌面通知服务器`dunst`

> 参考:
> 
> [Flameshot-Troubleshooting](https://flameshot.org/docs/guide/troubleshooting/#in-tiling-window-managers-e-g-i3wm-dwm-bspwm-flameshot-does-not-pin-the-screenshot)
> 
> [ArchWiki-Dunst](https://www.bing.com/search?q=dunst+archiwiki&qs=n&form=QBRE&sp=-1&ghc=1&lq=0&pq=dunst+archiwiki&sc=2-15&sk=&cvid=B7285795BA7F43B789B0BA33EA7FEA17&ghsh=0&ghacc=0&ghpl=)
> 
> [Dunst-Documentation](https://dunst-project.org/documentation/)

此时兴奋测试后又是失望，问题依旧

# 排查问题

几番搜索后，无果。遂开始控制变量法。因为最开始是由输入法快速输入后稳定触发此问题的，遂从fcitx5换到了ibus <del>ibus好丑</del> ，后失望而归。继续搜索查看文档的时候，在机缘巧合中发现只要是浮动的窗口只要碰到屏幕的边界都会触发这个问题
机缘巧合beLike:
<img title="" src="https://telegraph.7cmb.com/file/e1d88ff2e5793e82a2fa4.png" alt="beLike">
这时候想起来x11环境下跑的`picom`是别人fork后打包的aur，遂立刻在配置文件中关闭它的自启动并测试。果然，就是它的问题

# 修改picom

之后把picom的后端换成了xrender，虽然没问题但是显卡无法硬解。卸载后在aur里找到了日期新鲜的版本，安装后把xrender改回glx一切问题都迎刃而解

# 结语

没想到预计二十分钟的能解决的问题搞了这么久，太难受了，在此记录
