# 《绝地求生》启动黑屏闪退排查：VC++ 2022 运行库缺失与虚拟内存不足溢出深度修复

**核心排查结论：** PUBG 黑屏闪退根因是 VC++ 运行库缺失或虚拟内存不足，安装最新 VC++ 2022 并将页面文件扩容至 24GB 即可修复。

---

## 一、 底层机理排查与关键参数对比表

| 崩溃症状 | 事件查看器错误码 | 根本原因 | 修复手段 |
| --- | --- | --- | --- |
| 启动后 3 秒黑屏直接回桌面 | EventID 1000: vcruntime140.dll | VC++ 2015-2019 运行库版本过旧或损坏 | 重装官方最新 VC++ 全套 x64+x86 |
| 进度条走完黑屏后崩溃 | EventID 26: Out of Memory | 系统物理内存 + 虚拟内存总量不足 24GB | 手动扩容页面文件至 24576MB 固定大小 |
| 随机在载入地图时崩溃 | EventID 1000: TslGame.exe + 内存地址 0x0000003 | 显存流送池溢出或显卡驱动版本过旧 | 更新显卡驱动并在游戏内降低纹理质量 |
| 仅在低内存模式下崩溃 | 无事件日志，直接静默闪退 | Windows 内存压缩导致游戏内存页被回收 | 关闭内存压缩：PowerShell Disable-MMAgent |

> [!WARNING]
> **安全提示：** 修改页面文件大小前，请以管理员身份打开系统属性，确认选中的是系统盘（C 盘），并先记录当前页面文件配置（推荐截图备份）。将页面文件从「系统管理」切换为「自定义大小」后，初始大小与最大值均填写 24576（单位 MB），避免动态扩容引起磁盘碎片。

---

## 二、 核心排查与实操修复步骤

### 1. 步骤一：Windows 事件查看器精准定位崩溃原因
按 Win+R 输入 eventvwr.msc 打开事件查看器，进入【Windows 日志】->【应用程序】，筛选来源为「Application Error」的错误事件，确认崩溃 DLL 名称：

```powershell
# PowerShell 快速提取最近 5 条游戏崩溃日志
Get-EventLog -LogName Application -EntryType Error -Newest 20 | Where-Object { $_.Source -eq 'Application Error' } | Select-Object TimeGenerated, Message | Format-List
```

### 2. 步骤二：手动扩容系统虚拟内存页面文件至 24GB
按 Win+R 输入 `sysdm.cpl`，打开系统属性 -> 高级 -> 性能设置 -> 高级 -> 虚拟内存 -> 更改，按以下参数设置：

```cmd
:: 页面文件参数（在系统属性图形界面中填写）
初始大小 (MB): 24576
最大值 (MB):  24576
驱动器: C:\ (系统盘)
:: 确认后重启电脑使设置生效
```

### 3. 步骤三：重装 VC++ 运行库全套并关闭内存压缩
下载并静默安装官方最新版 VC++ Redistributable 全套，同时关闭 Windows 内存压缩功能：

```powershell
# 关闭 Windows 内存压缩（需管理员）
Disable-MMAgent -mc

# 验证关闭是否生效
Get-MMAgent
# MemoryCompression 应显示 False
```


---

## 三、 常见故障排查与深度 FAQ

#### Q: 电脑已有 32GB 内存，PUBG 还是提示虚拟内存不足？
> **A:** 32GB 物理内存充足但仍提示溢出，通常是游戏启动时瞬时内存申请超过系统限额。确保页面文件未关闭（不要设置为「无分页文件」），保持至少 8GB 固定页面文件作为应急缓冲。

#### Q: VC++ 装了 2022，游戏还是提示 vcruntime140_1.dll 缺失？
> **A:** 部分 PUBG 版本依赖 VC++ 2015（包含 vcruntime140.dll），需同时安装 2015 / 2017 / 2019 / 2022 全套 x64 与 x86 版本，缺少任意一个均会报 DLL 缺失。


---

## 四、 站内相关深度排查推荐

- [PUBG 启动报错 25 与 TslGame.exe 闪退：BattlEye 服务通信中断与运行库重装修复](https://www.221qk.com/pubg/pubg_01_error_25_crash_fix.html)
- [PUBG Nvidia 控制面板电竞级参数配置：Ultra 低延迟模式全套教程](https://www.221qk.com/pubg/pubg_01_nvidia_ultra_low_latency_settings.html)

---
*本文由 221qk.com 电竞技术研究室独家实测原创，定位于合规的电竞外设调试、掉帧排查与客户端崩溃修复。*
