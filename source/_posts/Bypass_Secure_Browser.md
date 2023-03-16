---
title: 记录绕过某考试软件的安全防护
date: 2023-3-16 13:30:29
categories: 
- Crack
tags:
- Crack
- Electron
- .Net
---
早在疫情期间就经历了好久的线上考试，最近又遇到了类似的需求，正好就写写相关的东西吧。为了防止暴露是哪几款软件，文中就不放图了，只是说说方法。

# 逆向相关

目前遇到过的主流是C#/Electron的程序，也有部分C++的程序。

C#的可以直接用DnSpy查看代码并修改

Electron的可以解包asar查看代码，修改后也可以打包替换回去

C++的一般IDA辅助分析后，可以手动跳过部分函数或判断

目前还未发现对自身文件进行校验的例子，如果后续出现，还需要hook相关函数进行绕过

注意有些修改不能无脑删除函数，因为有些会有操作记录等要注意保留

# 准备工作

想办法进入考试模式，进行初步测试，在进入前上线远控（为防止部分软件限制未知进程，可以直接注入系统进程），并记录考试过程中存在的进程，以及考试相关进程加载的dll。对这些文件进行简单的筛选即可找到涉及反作弊的文件。

以下描述注意针对考试中使用远控远程辅助答题的情况

# 防护措施及应对

## 1.截屏

### 原理

Windows下考试软件防止截屏的方法其实很简单，都是基于[SetWindowDisplayAffinity](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-setwindowdisplayaffinity)这个api

### 绕过

逆向后删除此函数调用，或者修改第二个参数内容即可。如果是Electron的浏览器，可以去ASAR解包的代码中搜索setContentProtection，如果存在，也删除其调用

## 2.模拟输入

### 原理

SetWindowsHookEx挂钩WH_KEYBOARD_LL，并判断KBDLLHOOKSTRUCT的flags，模拟键盘输入时flags为LLKHF_INJECTED

SetWindowsHookEx挂钩WH_MOUSE_LL，并判断MOUSEHOOKSTRUCT的wHitTestCode，在真实鼠标按键时为0，而在使用模拟鼠标时，为1

### 绕过

1.逆向查找挂钩的回调函数（搜索CallNextHookEx调用，一般此调用上面几行就是过滤函数），修改判断条件或直接删除相关内容即可

2.SetWindowsHookEx的回调是一个链，后添加的先处理，也就是说只有保证自己的回调在链的头部，即可自己修改flags值再CallNextHookEx继续传递

## 3.进程

将使用的远控注入进系统进程，规避可疑进程名的检测，不过大多数考试软件也没有太多的进程检测，也检测不过来

## 4.流量

可以通过代理抓包，部分考试软件回传信息没有加密，可以直接拦包之后正则匹配改包再重发

# 叠甲

本文内容仅供技术研究，请勿用于非法行为。
