[English](README.md) · **简体中文**

# OptiScaler DLSSNR-PreSR + XeMFG（社区整合版）

> **English TL;DR** — A community merge of three **GPL-3.0** projects: upstream
> [OptiScaler](https://github.com/optiscaler/OptiScaler) (base),
> [wilsjo2's DLSSNR-PreSR-Multipass fork](https://github.com/wilsjo2/OptiScaler-DLSSNR-PreSR-Multipass)
> (DLSS Neural Rendering, NR-before-SR, NVFP4 hybrid kernels) and
> [Coldwood1026's Dp4aUnlock](https://github.com/Coldwood1026/OptiScalerDp4aUnlock)
> (unlocks Intel XeFG/XeLL multi-frame generation, including on non-Intel adapters).
> Not affiliated with, endorsed by or authorised by Intel, NVIDIA, AMD or CD Projekt Red.
> You must supply `nvngx_dlssnr.dll` yourself — no NVIDIA runtime is redistributed.
> MFG is currently capped at 6X; see the [roadmap](#1-后续更新目标) and
> [known limitations](#7-实测数据与已知限制).
> Source is GPL-3.0; build with **VS 2022 + MSBuild** (this is *not* a CMake project).

---

## 0. 这是什么，和上游什么关系

这是一个**整合版**，不是从零写的分支。三份代码的关系：

| 组件 | 来源 | 提供的部分 |
|---|---|---|
| 底座 | [optiscaler/OptiScaler](https://github.com/optiscaler/OptiScaler) | 超分/插帧互换、spoofing、D3D12/Vulkan hook、叠加层与 ini |
| NR / PreSR | [wilsjo2/OptiScaler-DLSSNR-PreSR-Multipass](https://github.com/wilsjo2/OptiScaler-DLSSNR-PreSR-Multipass) | DLSS 神经网络渲染（NR）、NR 在 SR 之前、多 pass、NVFP4 hybrid 内核 |
| XeMFG 解锁 | [Coldwood1026/OptiScalerDp4aUnlock](https://github.com/Coldwood1026/OptiScalerDp4aUnlock) | XeFG/XeLL 多帧生成解锁（让非 Intel 显卡也出现倍率选项）、XeFG pacing |
| 本仓库 | [OhanhanO/OptiScaler-DLSSNR-PreSR-Multipass-With-XeMFG](https://github.com/OhanhanO/OptiScaler-DLSSNR-PreSR-Multipass-With-XeMFG) | 把上面三者合成一份可编译、可运行的树 |

> **如果你只需要其中一个特性，请直接用对应的上游仓库，不要用本整合版。**
> 只要 NR / PreSR → 用 wilsjo2 的；只要 XeMFG 解锁 → 用 Coldwood1026 的。
> 本整合版存在的理由只有一个：**同时**要 NR 和 XeMFG。

关于仓库名里的 "Dp4a"：XeSS 在没有 XMX 单元的显卡上走 DP4A 路径，而这套解锁针对的正是那条路径上的门控——Intel 用"我是不是那个 `igxess_fg.dll` 构建"来判断是否开放 MFG，而不是用硬件能力，所以在非 Intel 显卡上也能出现倍率选项。

---

## 1. 后续更新目标

### 1.1 把倍率下发移出 Present 路径（已知设计缺陷，**尚未实际触发**）

**现状**：游戏运行中修改 MFG 倍率时，新倍率是在 Present 调用内部、帧生成仍处于**启用**状态时同步下发到 Intel provider 的。

**问题**：这是一个调用时机上的设计缺陷。理论上它可能在 Present 路径持有的锁与 provider 内部状态之间形成相互等待，表现为"改完倍率后画面卡死，日志最后一行停在 `Interpolation count changed X -> Y`"。

**计划**：把下发移到"先禁用 → 等待静止 → 修改 → 暂停若干帧 → 再启用"的安全窗口，即在 evaluate 阶段完成，而不是在 Present 阶段。

**状态说明**：本项目在大量实机使用中——包括在游戏内反复来回切换 2X / 3X / 4X——**没有实际遇到过该问题**；日志显示这个调用每次都在微秒级返回。因此它被列为**待修的设计问题**，而不是已确认的故障。

### 1.2 恢复更高的倍率上限

当前上限被压在 6X（原因见 [7.1](#71-mfg-上限当前为-6x)）。要恢复到 8X，需要先把菜单的倍率下拉框改成动态生成标签，再放宽上限——顺序不能反，否则会引入另一个问题：打开下拉框即越界。

---

## 2. 版本记录

| 版本 | 变更 |
|---|---|
| v1.1 | MFG 上限默认降到 6X，修掉"打开 MFG 下拉框越界卡死" |
| v1.0 | 首次整合：wilsjo2 NR/PreSR + Coldwood XeMFG |

---

## 3. 功能对比

| 功能 | 上游 OptiScaler | wilsjo2 分支 | Coldwood 分支 | **本整合版** |
|---|:---:|:---:|:---:|:---:|
| 超分/插帧互换底座（DLSS/FSR/XeSS、spoofing、菜单、ini） | ✓ | 继承 | 继承 | ✓ |
| DLSS 神经网络渲染（NR，需 `nvngx_dlssnr.dll`） | ✗ | ✓ | ✗ | **✓** |
| **NR 在 SR 之前**（PreSR multipass，`RunBeforeSR`） | ✗ | ✓ | ✗ | **✓** |
| NR 多 pass、逐 pass 控制、皮肤遮罩 | ✗ | ✓ | ✗ | ✓ |
| NR 应用到成品画面（`FinishedPicture`、HDR 传输） | ✗ | ✓ | ✗ | ✓ |
| padded pre-SR（原尺寸零偏移修正） | ✗ | ✓ | ✗ | ✓ |
| XeFG（XeSS 帧生成输出） | ✓（倍率受 provider 上限与适配器门控） | 继承 | 继承 | ✓ |
| **XeFG MFG 解锁**（非 Intel 适配器出现倍率选项） | ✗ | ✗ | ✓ | **✓** |
| XeLL 生成帧数上限解锁 | ✗ | ✗ | ✓ | ✓ |
| XeFG pacing（>2X 帧节奏修正，默认开） | ✗ | ✗ | ✓ | ✓ |
| DLSS MFG 解锁（Ada / RTX 40） | ✗ | ✓ | ✗ | ✓ |
| DLSS MFG（Ampere SM75/SM86，经 `dlssg_for_sm86`） | ✗ | ✓ | ✗ | ✓（需构建时勾选） |
| NVFP4 hybrid NR 内核（RTX 50 专属） | ✗ | ✓ | ✗ | ✓ |
| MFG 倍率上限 | provider 决定（出厂 3 = 4X） | 同左 | 可提到 7（8X） | **默认 5（6X）** |

图例：✓ = 有；✗ = 没有；继承 = 该分支从更上游带下来的功能，并非该分支作者原创。

---

## 4. 系统需求

- **Windows 10/11 x64**，游戏走 OptiScaler 的 D3D12 路径（原生 D3D12 最佳；D3D11/Vulkan 游戏可用其桥接）。
- **NR（神经网络渲染）**：NVIDIA RTX 20/30/40/50 + 驱动 **≥ 616.56**。
  - 必须自备 `nvngx_dlssnr.dll`（约 158 MB，NVIDIA 衍生，**不可再分发**）：
    - RTX 50 → NVIDIA 原版 310.8，SHA-256 `e16bcf15e16e13f527491cdf7845b2fe6521a738d8f7c9c721866a8496e1fc8e`
    - RTX 20/30/40 → ShortFuse 跨代版 310.8，SHA-256 `e67dee209320cdafe0e93e45675d7aa34323a53acc57a72b2e40a181581c989a`
  - 获取方式与游戏专属步骤见仓库自带 [`INSTALL-DLSSNR.md`](INSTALL-DLSSNR.md)。
- **XeMFG**：依赖 Intel 的 `libxess_fg.dll` / `libxell.dll`，**发布包内已带**匹配版本（`libxess_fg.dll` 1.3.1.78，TimeDateStamp `0x69cb0f4d`；`libxell.dll` 1.3.2.10，TimeDateStamp `0x6a561284`），这两个 build 正是解锁补丁所针对的，**无需自备**。
- ⚠️ **不要在带反作弊的多人游戏里使用。**

---

## 5. 安装

1. **关掉游戏和启动器**，备份已有的代理 DLL 与 `OptiScaler.ini`。
2. 把 release zip **全部**解压到**游戏 exe 所在目录**（不是 `OptiScaler\` 子目录，也不是游戏根目录；虚幻引擎游戏通常是 `...\Binaries\Win64`）。
3. 把你自备的 `nvngx_dlssnr.dll` 放进同一目录（它和包里的 `nvngx.dll_dlssnr.dll` 是**两个不同文件，都要在**）。
4. 运行 `setup_windows.bat`，选择代理文件名。一般先用 `dxgi.dll`；若游戏里已被其他加载器占用，可换 `dbghelp.dll`。
5. 进游戏，按 `Insert` 开叠加层，按下面的推荐设置调。
6. 便携用法：保持 `[ProcessFilter] TargetProcessName=auto`，**不要**把别的游戏的 exe 名复制进来——不匹配会让 OptiScaler 进入直通模式（没有菜单、没有 NR）。

---

## 6. 推荐设置（在叠加层里调）

以下都在游戏内 `Insert` 叠加层里完成；改动会自动写回 `OptiScaler.ini`，所以下面的 ini 键只是**对照参考**，不需要手工编辑。

### A. 画质优先

| 位置 | 项目 | 设为 |
|---|---|---|
| Frame Generation | `Active` | 勾上 |
| Frame Generation | FG 输出 / Output | **XeFG** |
| Frame Generation | `MFG` 下拉 | 3X 或 4X |
| DLSS Neural Rendering | `Enable` | 勾上 |
| DLSS Neural Rendering | **Apply before Super Resolution** | **勾上**（这一项就是"NR 在 SR 之前"） |
| DLSS Neural Rendering | `Model passes` | 1（先一遍；多一遍多一份开销） |
| DLSS Neural Rendering | 模型分辨率 | 1.0（帧数不够再往下降） |

对照 ini：

```ini
[FrameGen]
Enabled=true
FGOutput=XeFG
[DlssNr]
Enabled=true
RunBeforeSR=true
Passes=1
WorkingScale=1.0
[XeFG]
InterpolationCount=3    ; 3 = 4X；若要 3X 请改成 2
```

### B. 低延迟 / 省性能

同上，但把**模型分辨率降到 0.5**（只降模型的工作分辨率，游戏分辨率不动，开销大致随比例平方下降），倍率用 **2X 或 3X**。

对照 ini：

```ini
[DlssNr]
Enabled=true
RunBeforeSR=true
Passes=1
WorkingScale=0.5
[XeFG]
InterpolationCount=2    ; 2 = 3X；若要 2X 请改成 1
```

**为什么不建议 4X 以上**：倍率只增加显示帧数，**不改善输入延迟**——延迟由基础帧决定。基础帧 20 fps 时输出 80，手感仍是 20 fps 的响应，而 pacing 的目标间隔会被压到 12 ms 以下，实测根本交不出货（见[第 7 节](#7-实测数据与已知限制)）。若目标是 60 fps，3X 通常已经够；把余量花在模型分辨率上收益更大。

> `InterpolationCount` 的值是"**每基础帧额外生成的帧数**"：`1` 表示基础帧 + 1 张生成帧 = 2X，`2` = 3X，`3` = 4X，以此类推。所以想要 2X 应写 1，不是 2。ini 里写 1–3；更高的倍率在叠加层里选。

### 分步启用（建议顺序）

出问题时能立刻定位是哪一侧，所以建议分两步：

1. 先只开帧生成：`FG Output = XeFG`、`MFG` 选 2X，**先不开 NR**。确认解锁生效——看日志有没有 `XeFG unlock: recognised provider build 0x69cb0f4d`，以及叠加层里倍率下拉是否可用、画面是否正常。
2. 再开 NR：`DLSS Neural Rendering → Enable` + `Apply before Super Resolution`，`Model passes = 1`。确认日志出现 `DLSS-NR running before SR: ...`。

---

## 7. 实测数据与已知限制

数据取自 **《巫师 3》（DX12）+ 驱动 616.92**，为发布时的实测样本。

### 7.1 MFG 上限当前为 6X

`XeFGMaxInterpolations = 5`。原因：菜单的倍率下拉框用的是一个 **5 项固定标签数组**，而循环上界取的是 provider 上报的上限；上限一旦超过 5，打开下拉框就会越界读到栈上的垃圾指针 → **卡死或崩溃**。

→ **不要在未同时修菜单的情况下把它改回 7。** 想恢复 8X，见 [1.2](#12-恢复更高的倍率上限)。

### 7.2 NR 的开销（同一场景内随模型分辨率变化）

| 一帧内 NR 耗时 | 模型 / 引导缓冲分辨率 |
|---|---|
| 29.3 – 30.7 ms | 1280×720 |
| 21.8 – 23.4 ms | 1129×635 |

NR 在整帧里占的比重很大，所以**降模型分辨率是提升基础帧最有效的杠杆**。日志里也能看到 NR 确实在 SR 之前生效：

```
DLSS-NR running before SR: target 1280x720, model 1280x720, guides 1280x720 ...
DLSS-NR before SR: the game's DLSS colour space is linear HDR so the colour transform is on
```

### 7.3 XeFG pacing 只在 >2X 参与，且交不出目标间隔

它的调度块触发条件是**生成帧数 > 2，也就是 4X 及以上**（3X 时不参与）。4X 实测：

| 目标间隔 | 实测平均 gap | 实测最大 gap |
|---:|---:|---:|
| 12.8 – 16.9 ms | 30 – 57 ms | 最高 1192 ms |

> 目标间隔 = 当前基础帧间隔 ÷ 倍率，由 pacing 按实测帧时间动态计算；所以基础帧越慢、倍率越高，这个目标就越紧。同一时间窗内 `scheduler ... 22 ms avg inside`（约 1.5 万次 deadline 被 rebase、7 千余次被 clamp）。

所以**降到 3X** 不只是少一档倍率，还会绕开这一整类抖动。

### 7.4 运行中切换倍率是可行的

巫师 3 这一场里，倍率在 **2X / 3X / 4X 之间来回切换多次**，每次都正常生效，没有卡死、没有 `SetNumInterpolatedFrames error`：

```
XeFG_Dx12::Dispatch Interpolation count changed 3 -> 2     ; 4X 降到 3X
XeFG_Dx12::Dispatch Interpolation count changed 2 -> 1     ; 3X 降到 2X
XeFG_Dx12::Dispatch Interpolation count changed 1 -> 3     ; 2X 升到 4X
```

（这与 [1.1](#11-把倍率下发移出-present-路径已知设计缺陷尚未实际触发) 的状态说明一致：那条是设计层面的隐患，不是已确认的故障。）

### 7.5 其他

- `OptiInput::LogInputHealthSnapshotLocked ... no window/queue/raw input` 会在部分游戏持续刷屏，但菜单实际可用（日志里能看到菜单可见性切换与设置保存）。属已知噪声。

---

## 8. 故障排查

| 现象 | 可能原因 | 处理 |
|---|---|---|
| 打开 MFG 下拉框就卡死/崩溃 | provider 上报的上限 > 菜单标签数组长度（5） | 把 `XeFGMaxInterpolations` 降回 5，或先把菜单改成动态标签 |
| 改倍率后立刻卡死，日志最后一行是 `Interpolation count changed X -> Y` | 见 [1.1](#11-把倍率下发移出-present-路径已知设计缺陷尚未实际触发) | 临时可用 ini 预设倍率规避；长期按 1.1 修 |
| NR 不初始化（`the model would not initialise`） | `nvngx_dlssnr.dll` 版本不对 | RTX 20/30/40 要用跨代版 `e67dee20…`，不是原版 `e16bcf15…`；重新核对 SHA-256 |
| 完全没有菜单 | 代理没被加载 / 进程过滤不匹配 | 检查安装目录与代理名、杀软隔离；`[ProcessFilter] TargetProcessName=auto` |
| XeFG 没有倍率选项 | provider 未命中解锁目标 | 看日志有没有 `recognised provider build 0x69cb0f4d`；不是这个 build 就会走兜底、不生效 |
| 某游戏里 RR / 帧生成选项变灰 | `d3d12.dll` 代理与 Streamline 冲突 | 换 `dxgi.dll`（也有报告是反向的，两个都试） |
| 崩溃，日志尾出现 `Failed to present Interpolated frame, HRESULT: 0x887a0005`（即 `DXGI_ERROR_DEVICE_REMOVED`），紧接着 `Device removed reason: DXGI_ERROR_INVALID_CALL`（后者是设备移除的具体原因，对应 `0x887a0001`） | 设备级故障：驱动重置，或交换链重建期间的失效调用 | 先降倍率、再关 NR 做对比；退出游戏或切换显示模式前后最容易出现；必要时更新驱动 |

---

## 9. 卸载与回滚

- **用自带卸载器**：运行 `setup_windows.bat` 时它会同时在游戏目录生成 `Remove_OptiScaler.bat`。它会删除代理 DLL（即那些"原始文件名是 `OptiScaler.dll`"的 `dxgi.dll`/`winmm.dll`/… ）、`OptiScaler.ini`、`OptiScaler.log`、`OptiScaler.asi`、`Licenses\`、`OptiScaler\`（含 `D3D12_Optiscaler\`、`Streamline\`、`plugins\`）以及自带的 `OptiScaler\*`。
- **它不会删**：你自备的 `nvngx_dlssnr.dll`、转发器 `nvngx.dll_dlssnr.dll`，以及 `docs\`、`redist\`、`setup_windows.bat`、`get_streamline.ps1` 等文件。要彻底清干净就手动删掉它们。
- **手动回滚**（更省事）：把代理 DLL 改回原名或直接删除即可；想完全恢复就把 `OptiScaler.ini`、`OptiScaler.log`、`OptiScaler\` 目录一起删掉。
- 用 Steam 等平台的话，删完做一次文件校验即可。

---

## 10. 日志与验证

release 包里 `LogToFile=true`、`LogLevel=2` 已默认打开，日志在游戏 exe 旁 `OptiScaler.log`。确认三项生效：

```
XeFG unlock: recognised provider build 0x69cb0f4d
XeLL unlock: recognised provider build 0x6a561284, gate at 0xd1c9, 3 -> N     ← N 即实际生效的上限（当前 5）
DlssNr_Dx12::Dispatch DLSS-NR running before SR: target ... model ... guides ...
```

> `gate at 0xd1c9` 里的地址随 `libxell.dll` 的 build 变化，只用来确认命中的是同一个构建，不是固定值。

---

## 11. 构建

本项目是**源码仓库**，主构建是 **MSBuild + `OptiScaler.sln`**（**不是 CMake**；仓库里的 `CMakeLists.txt` 只属于 `dlssnr/forwarder`，它同时也由解决方案的 `dlssnr_forwarder.vcxproj` 构建）。工具链：Visual Studio 2022（平台工具集 `v143`、`/std:c++latest`）+ Windows SDK 10.0。

- **推荐**：直接跑仓库自带的 GitHub Actions（`.github/workflows/package_release.yml`，手动触发），无需本机工具链。
- **本机**：装了 VS 2022 后，在仓库根目录执行 `build-release.bat -Version v1.1`（会依次拉子模块、校验 hybrid kernels、编译打包）。也可手工分步：`git submodule update --init --recursive` → `get_hybrid_assets.ps1` → `package_release.ps1`。

---

## 12. 许可证与合规

### 12.1 本项目

本仓库（含 `OptiScaler/` 全部源码）继承上游，采用 **GNU GPL-3.0**，全文见根目录 [`LICENSE`](LICENSE)。wilsjo2 与 Coldwood 分支的 `LICENSE` 与本项目为同一份 GPLv3 文本，因此**互相整合不存在授权冲突**。

GPL-3.0 要求分发二进制时提供对应源码：本仓库公开源码，release 页也附带源码包，满足该要求。

### 12.2 第三方组件与再分发（请自行确认）

发布包内**包含**以下第三方二进制，它们**不是** GPL，各自受自己的许可证约束（包内 `Licenses\` 已附对应文本）：

| 组件 | 文件 | 许可证文本 |
|---|---|---|
| Intel XeSS 运行库 | `OptiScaler\libxess.dll`、`libxess_dx11.dll`、`libxess_fg.dll`、`libxell.dll` | `Licenses\XeSS_LICENSE.txt` |
| AMD FidelityFX | `OptiScaler\amd_fidelityfx_*.dll` | `Licenses\FidelityFX_v2_LICENSE.md` |
| Microsoft DirectX / Agility SDK | `OptiScaler\D3D12_OptiScaler\D3D12Core.dll` | `Licenses\DirectX_LICENSE.txt` |
| RenoDX（NR 颜色合成着色器的来源，MIT） | 源码：`OptiScaler/shaders/dlssnr/precompile/dlssnr.hlsl` | `Licenses\RenoDX_ATTRIBUTION.txt` |

> ⚠️ 注意区分两件事：**解锁代码本身**不打包、不修改任何 Intel 文件（只在内存里打补丁，磁盘上的 DLL 一个字节都不动）；但**发布包**里确实带了上面这些运行库——它们来自构建产物。若你不想承担这部分再分发责任，可以在打包时排除它们，让用户自备。

**不包含**：`nvngx_dlssnr.dll` 与 DLSS FG 运行库（NVIDIA 的，打包脚本会拒绝收录），用户自备。

### 12.3 免责声明

本项目**仅供学习与研究**。它不隶属于、未获 Intel、NVIDIA、AMD 或 CD Projekt Red 的认可或授权；相关商标仅用于指明兼容对象。

XeMFG 解锁针对的是 Intel 运行库中的硬件门控逻辑，**可能与你所在地区的法律或相关软件许可条款冲突**；不同游戏、不同发布渠道对此的态度也不同。使用者需自行承担全部风险，本项目不提供任何担保。

请勿在带反作弊保护的多人游戏中使用。作者不对任何封号、数据损坏或硬件问题负责。

---

## 13. 致谢

- [optiscaler/OptiScaler](https://github.com/optiscaler/OptiScaler) — 底座
- [wilsjo2/OptiScaler-DLSSNR-PreSR-Multipass](https://github.com/wilsjo2/OptiScaler-DLSSNR-PreSR-Multipass) — NR / PreSR / hybrid
- [Coldwood1026/OptiScalerDp4aUnlock](https://github.com/Coldwood1026/OptiScalerDp4aUnlock) — XeMFG 解锁 / pacing
- [clshortfuse/renodx](https://github.com/clshortfuse/renodx) — NR 颜色合成着色器来源
- [Dagherbou/OptiScaler_DLSSNR](https://github.com/Dagherbou/OptiScaler_DLSSNR) — Neural Rendering 分支来源
- [intel/xess](https://github.com/intel/xess) — XeSS SDK
- [optiscaler/OptiPatcher](https://github.com/optiscaler/OptiPatcher) — 可选插件，用于在不 spoofing 的前提下暴露 DLSS/DLSSG 输入

完整第三方清单见 `docs/CREDITS.md`。

---

## 14. 问题反馈

反馈时请附上：`OptiScaler.log`（默认已开到 Info）、显卡与驱动版本、游戏名、以及你在叠加层里用的设置。release 包内 `SHA256SUMS.txt` 可用于校验文件完整性。
