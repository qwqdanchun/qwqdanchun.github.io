---
title: Code Page注入方法的武器化
date: 2022-11-25 21:03:31
categories: 
- 注入
tags:
- 注入
- CodePage
- NLS
- 武器化
---
# 起源

一年多以前，[Jonas L](https://twitter.com/jonasLyk)在[推特](https://twitter.com/jonasLyk/status/1352729173631135751)首次提出了这个注入方法，并在评论区提出了一些可能的利用方法。半年前，有人在GitHub发布了一份[Poc](https://github.com/NtQuerySystemInformation/NlsCodeInjectionThroughRegistry)，某种程度上进行了对注入方案的验证。

# 原理

控制台程序，会有一个对应的Code Page，也就是代码页，这个东西是字符代码的一个映射，每个控制台对应两个代码页，一个输入一个输出。

可以在`HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Nls\CodePage`看到系统中存在的代码页，也可以在这里添加我们自己的代码页。

大部分Code Page都是nls文件，但是可以看到有一部分是dll文件，确切地说是5开头的五位数对应的部分（此处未经逆向，仅通过测试进行判断，如有错误，还请见谅）。通过修改或添加一个符合条件的代码页，就可以使使用这个代码页的进程加载这个dll。

这个dll需要有一个名称为NlsDllCodePageTranslation的导出函数，来保证其正常加载。（其实没有也可以，根据需求自行判断吧）

在Jonas L的推特帖子中，使用控制台作为示例，直接通过chcp命令进行了演示，这也是最简单的一个实现方法。而后在评论区提到了SetThreadLocale、WM_INPUTLANGCHANGE等方法，后面出现的poc就使用了SetThreadLocale的思路，但是由于api的限制，这个poc只适合作为一个演示使用，并不适合在实战环境利用。

# 武器化

经过对SetThreadLocale的分析了解，可以得到结论，这种方法并不适合武器化，而WM_INPUTLANGCHANGE理论上可行，实现起来略微复杂，此处也暂且忽略。

至此，针对已有进程进行注入的路子都不考虑，剩下就是对新进程的注入，也就是类似于常用的spawn。

由于这个方法不可避免的dll文件落地，所以先考虑dll的问题。每次单独生产dll是不现实的，未经测试的陌生文件直接落地太不尊重目标机器的防护软件了，剩下要考虑的就是使用一个通用的dll去加载模块（dll或shellcode），所以可以制作一个从注册表、文件、环境变量等地方读取payload再加载的dll作为落地的文件。如果需要导出NlsDllCodePageTranslation函数，可以直接转发，示例代码：

```
//根据位数自己选择
#pragma comment(linker, "/export:NlsDllCodePageTranslation=\"C:\\Windows\\System32\\C_gsm7.NlsDllCodePageTranslation\"")
#pragma comment(linker, "/export:NlsDllCodePageTranslation=\"C:\\Windows\\Syswow64\\C_gsm7.NlsDllCodePageTranslation\"")
```

确定了文件的方案之后，就是修改代码页的问题

首先是添加带有自己dll的代码页，示例代码：`reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Nls\CodePage" /v 55555 /d "..\Temp\qwqdanchun.dll" /f `

这样只需要将dll放置在C:\Windows\Temp\qwqdanchun.dll处即可。

而后考虑修改程序对应的代码页，这里有两种方法：

* 1.如果直接选用cmd、powershell或类似的控制台程序作为spawn对象，可以在创建进程时指定将输入输出重定向，重定向输入chcp指令触发加载。
* 2.修改HKEY_CURRENT_USER\Console里的注册表项，添加控制台程序路径，并指定其使用的Code Page。示例代码：`reg add "HKEY_CURRENT_USER\Console\%SystemRoot%_system32_cmd.exe" /f & reg add "HKEY_CURRENT_USER\Console\%SystemRoot%_system32_cmd.exe" /v CodePage /t REG_DWORD /d 55555 /f` （此处以cmd为例，如果有其他需求可自行修改注册表项的名称）

至此，整套方案就已经可以投入使用了，但是存在一些缺点，使用时要注意

1. 目标进程必须是控制台进程
2. 添加代码页注册表键需要管理员权限（添加一次即可，后续不再需要管理员权限）
3. dll会落地，要考虑免杀
4. 不能用作inject，只能用于spawn
