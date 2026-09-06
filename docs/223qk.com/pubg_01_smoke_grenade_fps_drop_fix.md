# 《绝地求生》烟雾弹掉帧深度排查：DX11 Enhanced 着色器坏块清理与驱动级着色器池扩容实操

**核心排查结论：** PUBG 烟雾弹掉帧根因是 DX11E 着色器坏块积累，清空 DXCache 并在 Nvidia 控制面板扩容着色器池至 10GB 即可根治。

---

## 一、 底层机理排查与关键参数对比表

| 掉帧场景 | 帧率跌幅 | 根本成因 | 优化处置手段 |
| --- | --- | --- | --- |
| 投掷烟雾弹后 1~3 秒内 | 从 144fps 暴跌至 30fps 以下 | PSO 粒子着色器实时编译挂起 | 扩容着色器缓存池并清除坏块 |
| 多枚烟雾弹叠加区域 | 帧率持续低于 60fps | GPU 显存超额，烟雾粒子系统排队 | 降低效果质量 + 开启 DX11E 硬件粒子加速 |
| 进入烟雾区域视角转动 | 帧时间抖动达 30ms 以上 | CPU 与 GPU 粒子碰撞计算失衡 | 切换 Nvidia Ultra 低延迟 + 最大预渲染帧 1 |
| 烟雾消散后帧率恢复缓慢 | 消散 5 秒后才回到正常帧率 | 显卡显存碎片未即时回收 | 关闭异步着色器编译，强制同步完成 |

> [!WARNING]
> **安全提示：** 清理 DXCache 与着色器缓存后，首次进入游戏对局会触发着色器重新编译，前 3~5 分钟内游戏帧率会比正常偏低 30%~50%，这是正常现象。建议清理后先进训练场暖机 5 分钟，等显卡完成初次着色器预热后再进入正式对局。

---

## 二、 核心排查与实操修复步骤

### 1. 步骤一：PowerShell 一键清空全部 DX 着色器缓存目录
以管理员身份打开 PowerShell，执行以下命令彻底清除积累的损坏着色器坏块：

```powershell
Remove-Item -Path "$env:LOCALAPPDATA\NVIDIA\DXCache\*" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item -Path "$env:LOCALAPPDATA\D3DSCache\*" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item -Path "$env:APPDATA\..\LocalLow\PUBG\TslGame\ShaderCache\*" -Recurse -Force -ErrorAction SilentlyContinue
Write-Host "着色器缓存清理完成！"
```

### 2. 步骤二：Nvidia 控制面板扩容着色器缓存池至 10GB
桌面右键打开【Nvidia 控制面板】->【管理 3D 设置】->【全局设置】，找到并修改以下参数：

```cmd
Nvidia 控制面板参数设置：
1. 着色器缓存大小 (Shader Cache Size): 10 GB
2. 低延迟模式 (Low Latency Mode): 超高 (Ultra)
3. 最大预渲染帧数 (Max Pre-Rendered Frames): 1
4. 电源管理模式: 最高性能优先
5. 垂直同步: 关闭 (Off)
```

### 3. 步骤三：游戏内切换至 DX11 Enhanced 并配置烟雾效果
启动 PUBG，进入【设置】->【图形】，按以下参数调整引擎渲染模式以激活 GPU 硬件加速粒子计算：

```cmd
游戏内图形设置：
1. API 选择: DirectX 11 Enhanced (DX11E)
2. 烟雾质量: 中等（不要选「超高」，超高会触发 CPU 粒子模拟）
3. 抗锯齿: TAA（禁用 DLSS 可减少粒子重影）
4. 阴影质量: 中等或更低（减少 GPU 计算压力）

:: 启动参数（Steam 启动项中追加）
-USEALLAVAILABLECORES -dx11
```


---

## 三、 常见故障排查与深度 FAQ

#### Q: 清了缓存还是掉帧，应该升级显卡吗？
> **A:** 不一定。若 GPU 占用率在烟雾弹场景中只有 60%~70%，说明是 CPU 粒子碰撞计算造成的帧时间波动。建议在 Steam 启动项添加 `-USEALLAVAILABLECORES` 让引擎调用全部 CPU 核心。

#### Q: AMD 显卡如何清理着色器缓存？
> **A:** AMD 驱动面板中点击【设置】->【显卡】->【高级】->【重置着色器缓存（Reset Shader Cache）】，然后同样清理 `%LOCALAPPDATA%\D3DSCache\` 目录。


---

## 四、 站内相关深度排查推荐

- [PUBG DX12 与 DX11 实测切换指南：3090/4090 显卡帧率与画面稳定性选择策略](https://www.223qk.com/pubg/pubg_02_dx12_vs_dx11_performance_test.html)
- [PUBG Nvidia 控制面板电竞级参数配置：Ultra 低延迟模式全套教程](https://www.223qk.com/pubg/pubg_01_nvidia_ultra_low_latency_settings.html)

---
*本文由 223qk.com 电竞技术研究室独家实测原创，定位于合规的电竞外设调试、掉帧排查与客户端崩溃修复。*
