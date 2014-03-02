---
layout: post
title: "Ubuntu 升级后找不到 grub， 进不了系统处理"
description: "Ubuntu, Win 双系统下，Ubuntu 升级后找不到grub 引导文件处理"
category: "Linux"
tags: [skill]

---

今天整理Evernote的时候看到的以前的升级 Ubuntu 时的一点总结。

是这样的，在刚毕业那会儿, 那时买不起 Mac, 但是出于对Unix 的热爱，不得已在我的 ThinkPad X200s 上装了个Ubuntu 12.04,
那时还不想完全放开Win, 就装了个双系统，双系统的安装方式有很多，而我也选择的常规的 [EasyBCD](http://neosmart.net/EasyBCD/)
引导安装，安装完成后我在 Ubuntu 那边又设置了 grub 2 引导整个系统，把 EasyBCD 扔到一边去了，也就是我原来的Win
也是通过Grub 2 来引导的， 就这样用了一段时间。
那段时间里基本把[apue](http://www.amazon.cn/UNIX环境高级编程-史蒂文斯/dp/B00114GRG0/ref=sr_1_1?ie=UTF8&qid=1393741396&sr=8-1&keywords=unix+环境高级编程) 里面基础的内容学完了，也写了不少那时学学得不错的小代码。

12年底的时候看到Ubuntu 12.10 界面很是炫动，想也没想就执行了:

    $ sudo apt-get install update-manager-core && do-release-upgrade

经过漫长一夜的等待，升级完成了，但是当我关机重启后就再也进入不了Ubuntu 系统了(Win好像还是能进去的，我记不清楚了) 直觉告诉我
是grub2 在重装后把我的启动参数丢了，然后就上网找到以下的一些有用技能。

1. 使用U盘启动

    1. 使用类似 [ubootin](http://unetbootin.sourceforge.net) 制件一个U盘启动盘
    2. U盘启动盘 制件完毕后，找到 U 盘下面的 *./syslinux.cfg* 文件，注释掉第一行

            #default menu.c32
    3. 重启 PC, 选择从 U 盘启动
    4. 启动后输入:

        boot# unetbootindefault

2. 启动阶段的事情完成了， 接下来是修复安装 grub2:

            $sudo mount /dev/sdXY /mnt          #X是驱动器的名，Y 为分区号， 就是你的Ubuntu 安装的硬盘分区
            $sudo mount --bind /dev /mnt/dev && sudo mount --bind /dev/pts /mnt/dev/pts && sudo mount --bind /proc /mnt/proc && sudo mount --bind /sys /mnt/sys
            $sudo chroot /mnt
            #grub-install /dev/sdX
            #grub-install --recheck /dev/sdX
            #update-grub
            #exit && sudo umount /mnt/dev && sudo umount /mnt/dev/pts && sudo umount /mnt/proc && sudo umount /mnt/sys && sudo umount /mnt
            #reboot

3. 这一系列的操作完成后，你的grub 就能完美引导 Ubuntu了

接下来把 Win 加回到 grub 的引导分支上去: 更改 */boot/grub/grub.cfg*
文件，替换成下面内容

    ### BEGIN /etc/grub.d/40_custom ###
    # This file provides an easy way to add custom menu entries. Simply type the
    # menu entries you want to add after this comment. Be careful not to change
    # the 'exec tail' line above.
        menuentry 'Win7 (Sexy mode)'{
            insmod part_msdos
            insmod ntfs
            set root='hd0,msdos1'
            chainloader +1
        }
    ### END /etc/grub.d/40_custom ###

现在你的系统应该就可以完美运行了:)

