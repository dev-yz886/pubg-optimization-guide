# 《绝地求生》注册表级输入延迟优化：关闭 GameDVR、禁用 Nagle 算法与网络节流修复

**核心排查结论：** PUBG 输入延迟高根因是 GameDVR 后台录屏与 Nagle 算法延迟，注册表关闭 GameDVR 并写入 TCPNoDelay=1 即可优化。

---

## 一、 底层机理排查与关键参数对比表

| 调优项目 | 注册表路径 | 修改键值 | 优化收益 |
| --- | --- | --- | --- |
| 关闭 Xbox GameDVR 录屏 | HKCU\System\GameConfigStore | GameDVR_Enabled = 0 | 释放 GPU 独立编码器算力，减少帧时间波动 |
| 禁用应用捕获 | HKCU\Software\Microsoft\Windows\CurrentVersion\GameDVR | AppCaptureEnabled = 0 | 停止后台截图与录制内存占用 |
| 禁用 Nagle 算法 | HKLM\SYSTEM\...\Interface\{网卡GUID} | TCPAckFrequency=1, TcpNoDelay=1 | 消除网络包累积延迟，降低对局 Ping 波动 2~5ms |
| 关闭网络节流 | HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Multimedia\SystemProfile | NetworkThrottlingIndex = 0xFFFFFFFF | 禁止 Windows 对游戏网络数据包进行节流限速 |
| 提升游戏调度优先级 | 同上路径 Tasks\Games | Priority=6, GPU Priority=8 | 系统资源调度向游戏进程倾斜 |

> [!WARNING]
> **安全提示：** 修改注册表前请执行备份：在管理员 CMD 中运行 `reg export HKLM C:\HKLM_backup.reg` 与 `reg export HKCU C:\HKCU_backup.reg`。若修改后出现网络异常，打开注册表编辑器恢复对应键值即可。Nagle 算法禁用需要找到正确的网卡 GUID，错误 GUID 下修改无效但不会造成系统损坏。

---

## 二、 核心排查与实操修复步骤

### 1. 步骤一：注册表彻底关闭 Xbox GameDVR 后台录屏
以管理员身份打开 CMD 命令提示符，依次执行以下注册表写入命令：

```cmd
reg add "HKCU\System\GameConfigStore" /v GameDVR_Enabled /t REG_DWORD /d 0 /f
reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\GameDVR" /v AppCaptureEnabled /t REG_DWORD /d 0 /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\GameDVR" /v AllowGameDVR /t REG_DWORD /d 0 /f
reg add "HKCU\System\GameConfigStore" /v GameDVR_FSEBehavior /t REG_DWORD /d 2 /f
```

### 2. 步骤二：禁用 Nagle 算法降低网络包积累延迟
首先用 PowerShell 获取当前网卡的注册表 GUID，然后针对该网卡写入 Nagle 禁用参数：

```powershell
# 获取网卡 GUID
Get-NetAdapter | Select-Object Name, InterfaceGuid

# 假设网卡 GUID 为 {XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX}
# 在 CMD 中执行（替换 GUID）：
# reg add "HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters\Interfaces\{GUID}" /v TCPAckFrequency /t REG_DWORD /d 1 /f
# reg add "HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters\Interfaces\{GUID}" /v TcpNoDelay /t REG_DWORD /d 1 /f
# reg add "HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters" /v TCPNoDelay /t REG_DWORD /d 1 /f
```

### 3. 步骤三：提升 Windows 多媒体调度优先级
提高游戏进程网络与 CPU 调度权重，减少系统其他进程抢占资源：

```cmd
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Multimedia\SystemProfile" /v NetworkThrottlingIndex /t REG_DWORD /d 0xFFFFFFFF /f
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Multimedia\SystemProfile" /v SystemResponsiveness /t REG_DWORD /d 0 /f
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Multimedia\SystemProfile\Tasks\Games" /v Priority /t REG_DWORD /d 6 /f
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Multimedia\SystemProfile\Tasks\Games" /v "GPU Priority" /t REG_DWORD /d 8 /f
```


---

## 三、 常见故障排查与深度 FAQ

#### Q: Nagle 算法禁用后 Ping 反而升高了，是操作错误吗？
> **A:** 可能禁用了错误网卡接口的注册表项。使用 PowerShell `Get-NetAdapter` 确认正在使用的网卡 InterfaceGuid 与修改的 GUID 一致。同时确认未同时打开 VPN 等虚拟网卡接口。

#### Q: 关闭 GameDVR 后使用 Nvidia ShadowPlay 录制游戏是否受影响？
> **A:** 不受影响。GameDVR 是 Xbox 游戏栏录制功能，与 Nvidia ShadowPlay（需要 GeForce Experience）是独立的两套录制系统。关闭 GameDVR 反而会让 ShadowPlay 独占 GPU 编码器，提升录制流畅度。


---

## 四、 站内相关深度排查推荐

- [PUBG Nvidia 控制面板电竞级参数配置：Ultra 低延迟模式全套教程](https://www.303qk.com/pubg/pubg_01_nvidia_ultra_low_latency_settings.html)
- [PUBG 垂直灵敏度倍率科学换算：1.0~1.2 线性系数与跨镜头 eDPI 统一化计算](https://www.303qk.com/pubg/pubg_01_sensitivity_formula_vertical_multiplier.html)

---
*本文由 303qk.com 电竞技术研究室独家实测原创，定位于合规的电竞外设调试、掉帧排查与客户端崩溃修复。*
