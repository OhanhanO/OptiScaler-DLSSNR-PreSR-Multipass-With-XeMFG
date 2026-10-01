**English** · [简体中文](README.zh-CN.md)

# OptiScaler DLSSNR-PreSR + XeMFG (community merge)

A community merge of three **GPL-3.0** projects into one buildable, runnable tree:

| Component | Source | What it contributes |
|---|---|---|
| Base | [optiscaler/OptiScaler](https://github.com/optiscaler/OptiScaler) | Upscaler/frame-generation interposer, spoofing, D3D12/Vulkan hooks, overlay and ini |
| NR / PreSR | [wilsjo2/OptiScaler-DLSSNR-PreSR-Multipass](https://github.com/wilsjo2/OptiScaler-DLSSNR-PreSR-Multipass) | DLSS Neural Rendering, NR-before-SR, multi-pass, NVFP4 hybrid kernels |
| XeMFG unlock | [Coldwood1026/OptiScalerDp4aUnlock](https://github.com/Coldwood1026/OptiScalerDp4aUnlock) | XeFG/XeLL multi-frame generation unlock (multiplier options on non-Intel adapters), XeFG pacing |
| This repo | [OhanhanO/OptiScaler-DLSSNR-PreSR-Multipass-With-XeMFG](https://github.com/OhanhanO/OptiScaler-DLSSNR-PreSR-Multipass-With-XeMFG) | Combines the three above into one tree |

**Not affiliated with, endorsed by or authorised by Intel, NVIDIA, AMD or CD Projekt Red.** Trademarks are used only to identify what this is compatible with.

- You must supply `nvngx_dlssnr.dll` yourself — no NVIDIA runtime is redistributed.
- MFG is currently capped at 6X; see the [roadmap](#1-roadmap) and [known limitations](#7-measured-data-and-known-limitations).
- The source is GPL-3.0. The build is **VS 2022 + MSBuild** — this is *not* a CMake project.

> **If you only need one of the two features, use the corresponding upstream fork instead of this merge.**
> NR / PreSR only → wilsjo2's fork. XeMFG unlock only → Coldwood1026's fork.
> The only reason this merge exists is to have **both at once**.

About the "Dp4a" in the repo name: XeSS uses a DP4A path on GPUs without XMX units, and the unlock targets the gate on that path. Intel decides whether to expose MFG by asking "am I the `igxess_fg.dll` build?", not by querying hardware capability — which is why the multiplier options can appear on non-Intel adapters.

---

## 1. Roadmap

### 1.1 The multiplier hand-off happens inside Present (known design flaw)

**Current behaviour**: when the MFG multiplier is changed while the game is running, the new value is handed to the Intel provider synchronously from inside the Present call, **while frame generation is still enabled**.

**The problem**: this is a design flaw in the call timing. In theory it can deadlock between a lock held by the Present path and the provider's internal state, showing up as a freeze right after changing the multiplier, with the log stopping at `Interpolation count changed X -> Y`.

**Plan**: move the hand-off into the safe window — disable, wait until quiescent, apply the new value, pause for a few frames, re-enable — i.e. do it during the evaluate stage instead of during Present.

**Status**: across extensive real use — including switching 2X / 3X / 4X back and forth in-game — **this has never actually been reproduced**. The log shows the call returning in microseconds every time. It is therefore tracked as a **known design flaw**, not a confirmed failure.

### 1.2 Restoring the higher multiplier ceiling

The ceiling is currently held at 6X (see [7.1](#71-the-mfg-ceiling-is-currently-6x)). Raising it back to 8X requires making the multiplier combo generate its labels dynamically **first** — the order cannot be reversed, otherwise opening the dropdown goes out of bounds, which is a separate problem from 1.1.

---

## 2. Version history

| Version | Changes |
|---|---|
| v1.1 | MFG ceiling lowered to 6X by default; fixes the "opening the MFG dropdown goes out of bounds and freezes" crash |
| v1.0 | First merge: wilsjo2 NR/PreSR + Coldwood XeMFG |

---

## 3. Feature comparison

| Feature | Upstream OptiScaler | wilsjo2 fork | Coldwood fork | **This merge** |
|---|:---:|:---:|:---:|:---:|
| Upscaler/FG interposer base (DLSS/FSR/XeSS, spoofing, menu, ini) | ✓ | inherited | inherited | ✓ |
| DLSS Neural Rendering (NR, needs `nvngx_dlssnr.dll`) | ✗ | ✓ | ✗ | **✓** |
| **NR before SR** (PreSR multipass, `RunBeforeSR`) | ✗ | ✓ | ✗ | **✓** |
| NR multi-pass, per-pass controls, skin mask | ✗ | ✓ | ✗ | ✓ |
| Apply NR to the finished picture (`FinishedPicture`, HDR transfer) | ✗ | ✓ | ✗ | ✓ |
| Padded pre-SR (origin-zero offset fix) | ✗ | ✓ | ✗ | ✓ |
| XeFG (XeSS frame generation output) | ✓ (ceiling and adapter gating decided by the provider) | inherited | inherited | ✓ |
| **XeFG MFG unlock** (multiplier options on non-Intel adapters) | ✗ | ✗ | ✓ | **✓** |
| XeLL generated-frame-count unlock | ✗ | ✗ | ✓ | ✓ |
| XeFG pacing (>2X frame pacing fix, on by default) | ✗ | ✗ | ✓ | ✓ |
| DLSS MFG unlock (Ada / RTX 40) | ✗ | ✓ | ✗ | ✓ |
| DLSS MFG (Ampere SM75/SM86, via `dlssg_for_sm86`) | ✗ | ✓ | ✗ | ✓ (opt-in at build time) |
| NVFP4 hybrid NR kernels (RTX 50 only) | ✗ | ✓ | ✗ | ✓ |
| MFG multiplier ceiling | decided by the provider (3 = 4X out of the box) | same | can be raised to 7 (8X) | **5 (6X) by default** |

Legend: ✓ = present; ✗ = absent; *inherited* = the fork carries this down from further upstream; it is not that fork author's original work.

---

## 4. Requirements

- **Windows 10/11 x64**, with the game reaching one of OptiScaler's D3D12 paths (native D3D12 is best; D3D11 and Vulkan games can use its bridges).
- **NR (Neural Rendering)**: NVIDIA RTX 20/30/40/50 and driver **≥ 616.56**.
  - `nvngx_dlssnr.dll` must be supplied by you (~158 MB, NVIDIA-derived, **not redistributable**):
    - RTX 50 → original NVIDIA 310.8, SHA-256 `e16bcf15e16e13f527491cdf7845b2fe6521a738d8f7c9c721866a8496e1fc8e`
    - RTX 20/30/40 → ShortFuse cross-generation 310.8, SHA-256 `e67dee209320cdafe0e93e45675d7aa34323a53acc57a72b2e40a181581c989a`
  - Where to obtain it, plus game-specific steps, are in [`INSTALL-DLSSNR.md`](INSTALL-DLSSNR.md).
- **XeMFG**: depends on Intel's `libxess_fg.dll` / `libxell.dll`. **The release package already includes** matching builds (`libxess_fg.dll` 1.3.1.78, TimeDateStamp `0x69cb0f4d`; `libxell.dll` 1.3.2.10, TimeDateStamp `0x6a561284`) — these are exactly the builds the unlock patches target, so nothing needs to be supplied.
- ⚠️ **Do not use this in anti-cheat-protected multiplayer games.**

---

## 5. Installation

1. **Close the game and its launcher**, and back up any existing proxy DLL and `OptiScaler.ini`.
2. Extract the **entire** release zip into the directory containing the **game executable** (not the `OptiScaler\` subfolder, and not the game root; for Unreal Engine games this is usually `...\Binaries\Win64`).
3. Put your own `nvngx_dlssnr.dll` in the same directory. It and the bundled `nvngx.dll_dlssnr.dll` are **two different files, and both are required**.
4. Run `setup_windows.bat` and pick a proxy filename. Start with `dxgi.dll`; if another loader already occupies it in that game, try `dbghelp.dll`.
5. Launch the game, open the overlay with `Insert`, and apply the recommended settings below.
6. Portable use: keep `[ProcessFilter] TargetProcessName=auto`. **Do not** paste another game's executable name in — a mismatch puts OptiScaler into pass-through mode (no menu, no NR).

---

## 6. Recommended settings (adjusted in the overlay)

Everything below is done in the in-game `Insert` overlay. Changes are written back to `OptiScaler.ini` automatically, so the ini keys below are a **reference only** — no manual editing required.

### A. Image quality first

| Location | Item | Set to |
|---|---|---|
| Frame Generation | `Active` | on |
| Frame Generation | FG output | **XeFG** |
| Frame Generation | `MFG` dropdown | 3X or 4X |
| DLSS Neural Rendering | `Enable` | on |
| DLSS Neural Rendering | **Apply before Super Resolution** | **on** (this is what puts NR before SR) |
| DLSS Neural Rendering | `Model passes` | 1 (start with one pass; each extra pass costs more) |
| DLSS Neural Rendering | model resolution | 1.0 (lower it only if frame rate is short) |

Equivalent ini:

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
InterpolationCount=3    ; 3 = 4X; use 2 for 3X
```

### B. Low latency / performance first

Same as above, but set **model resolution to 0.5** (this lowers only the model's working resolution, not the game's; cost falls roughly with the square of the scale) and use a **2X or 3X** multiplier.

Equivalent ini:

```ini
[DlssNr]
Enabled=true
RunBeforeSR=true
Passes=1
WorkingScale=0.5
[XeFG]
InterpolationCount=2    ; 2 = 3X; use 1 for 2X
```

**Why 4X and above is not recommended**: the multiplier only raises the *displayed* frame count; it does **not** improve input latency, which is set by the base frame rate. With a 20 fps base, an 80 fps output still feels like 20 fps of responsiveness, and the pacing target interval drops below 12 ms — a target that measurements show it cannot meet (see [section 7](#7-measured-data-and-known-limitations)). If 60 fps is your goal, 3X is usually enough; spending the headroom on model resolution pays off more.

> `InterpolationCount` is the **number of extra frames generated per base frame**: `1` means base frame + 1 generated frame = 2X, `2` = 3X, `3` = 4X, and so on. So 2X is written as 1, not 2. The ini accepts 1–3; higher multipliers are selected in the overlay.

### Staged bring-up (recommended order)

Enabling one side at a time makes it obvious which side a problem belongs to:

1. Frame generation only: `FG Output = XeFG`, `MFG` = 2X, **NR still off**. Confirm the unlock took effect — look for `XeFG unlock: recognised provider build 0x69cb0f4d` in the log, and check that the multiplier dropdown is usable and the picture looks right.
2. Then NR: `DLSS Neural Rendering → Enable` plus `Apply before Super Resolution`, with `Model passes = 1`. Confirm `DLSS-NR running before SR: ...` appears in the log.

---

## 7. Measured data and known limitations

The data below was measured on **The Witcher 3 (DX12) with driver 616.92** and represents the sample taken at release time.

### 7.1 The MFG ceiling is currently 6X

`XeFGMaxInterpolations = 5`. The reason: the multiplier combo in the menu indexes a **fixed five-entry label array**, while its loop bound comes from the ceiling the provider reports. Above 5 the loop reads past the end of that array and hands ImGui a stack value as a label → **a freeze or a crash**.

→ **Do not set it back to 7 without fixing the menu first.** To restore 8X, see [1.2](#12-restoring-the-higher-multiplier-ceiling).

### 7.2 NR cost (varies with model resolution within the same scene)

| NR time per frame | Model / guide resolution |
|---|---|
| 29.3 – 30.7 ms | 1280×720 |
| 21.8 – 23.4 ms | 1129×635 |

NR takes a large share of the frame, so **lowering the model resolution is the most effective lever for raising the base frame rate**. The log also confirms NR really is running before SR:

```
DLSS-NR running before SR: target 1280x720, model 1280x720, guides 1280x720 ...
DLSS-NR before SR: the game's DLSS colour space is linear HDR so the colour transform is on
```

### 7.3 XeFG pacing only engages above 2X, and misses its target

Its scheduling block triggers when the **generated frame count is > 2, i.e. 4X and above** (it does not engage at 3X). Measured at 4X:

| Target interval | Measured average gap | Measured maximum gap |
|---:|---:|---:|
| 12.8 – 16.9 ms | 30 – 57 ms | up to 1192 ms |

> The target interval is the current base frame interval divided by the multiplier, computed dynamically by pacing from measured frame times. So the slower the base frame and the higher the multiplier, the tighter that target becomes. In the same window, `scheduler ... 22 ms avg inside` (roughly 15,000 deadlines rebased and 7,000+ clamped).

Dropping to **3X** therefore does more than remove one multiplier step — it sidesteps this entire class of stutter.

### 7.4 Changing the multiplier at runtime works

In this The Witcher 3 session the multiplier was switched back and forth between **2X / 3X / 4X** many times, and every change took effect normally — no freeze, no `SetNumInterpolatedFrames error`:

```
XeFG_Dx12::Dispatch Interpolation count changed 3 -> 2     ; 4X to 3X
XeFG_Dx12::Dispatch Interpolation count changed 2 -> 1     ; 3X to 2X
XeFG_Dx12::Dispatch Interpolation count changed 1 -> 3     ; 2X to 4X
```

(This is consistent with the status note in [1.1](#11-the-multiplier-hand-off-happens-inside-present-known-design-flaw): that one is a design-level hazard, not a confirmed failure.)

### 7.5 Other

- `OptiInput::LogInputHealthSnapshotLocked ... no window/queue/raw input` can spam the log in some games, while the menu still works (log entries show menu visibility changes and settings being saved). This is known noise.

---

## 8. Troubleshooting

| Symptom | Likely cause | What to do |
|---|---|---|
| Freeze or crash the moment the MFG dropdown is opened | The provider-reported ceiling exceeds the menu's label array length (5) | Lower `XeFGMaxInterpolations` back to 5, or make the menu generate labels dynamically |
| Freeze immediately after changing the multiplier, with `Interpolation count changed X -> Y` as the last log line | See [1.1](#11-the-multiplier-hand-off-happens-inside-present-known-design-flaw) | As a workaround, preset the multiplier in the ini; fix it properly as described in 1.1 |
| NR will not initialise (`the model would not initialise`) | Wrong `nvngx_dlssnr.dll` build | On RTX 20/30/40 use the cross-generation build `e67dee20…`, not the original `e16bcf15…`; re-check the SHA-256 |
| No menu at all | Proxy not loaded / process filter mismatch | Check the install directory, the proxy filename and antivirus quarantine; keep `[ProcessFilter] TargetProcessName=auto` |
| No multiplier options for XeFG | The provider did not match the unlock target | Check the log for `recognised provider build 0x69cb0f4d`; anything else falls back and does nothing |
| Ray Reconstruction / frame generation greyed out in some game | The `d3d12.dll` proxy conflicts with Streamline | Switch to `dxgi.dll` (there are also reports of the reverse; try both) |
| Crash with `Failed to present Interpolated frame, HRESULT: 0x887a0005` (that is `DXGI_ERROR_DEVICE_REMOVED`) near the end of the log, immediately followed by `Device removed reason: DXGI_ERROR_INVALID_CALL` (the specific removal reason, `0x887a0001`) | Device-level failure: a driver reset, or an invalid call during swapchain recreation | Lower the multiplier first, then disable NR for comparison; it is most likely around exiting the game or switching display modes; update the driver if it persists |

---

## 9. Uninstalling and rolling back

- **Use the bundled uninstaller**: running `setup_windows.bat` also generates `Remove_OptiScaler.bat` in the game directory. It deletes the proxy DLLs (those whose original filename is `OptiScaler.dll` — `dxgi.dll`, `winmm.dll`, …), `OptiScaler.ini`, `OptiScaler.log`, `OptiScaler.asi`, `Licenses\`, `OptiScaler\` (including `D3D12_Optiscaler\`, `Streamline\`, `plugins\`) and the bundled `OptiScaler\*` files.
- **It does not delete** your own `nvngx_dlssnr.dll`, the forwarder `nvngx.dll_dlssnr.dll`, or `docs\`, `redist\`, `setup_windows.bat`, `get_streamline.ps1` and similar files. Remove those by hand for a completely clean directory.
- **Manual rollback** (simpler): rename or delete the proxy DLL. To remove everything, also delete `OptiScaler.ini`, `OptiScaler.log` and the `OptiScaler\` directory.
- On a platform like Steam, verify the game files afterwards.

---

## 10. Log and verification

Release packages ship with `LogToFile=true` and `LogLevel=2` already set; the log is `OptiScaler.log` next to the game executable. Three lines confirm everything is in place:

```
XeFG unlock: recognised provider build 0x69cb0f4d
XeLL unlock: recognised provider build 0x6a561284, gate at 0xd1c9, 3 -> N     <- N is the effective ceiling (currently 5)
DlssNr_Dx12::Dispatch DLSS-NR running before SR: target ... model ... guides ...
```

> The address in `gate at 0xd1c9` changes with the `libxell.dll` build. It only serves to confirm that the same build was matched; it is not a fixed value.

---

## 11. Building

This is a **source repository**. The main build is **MSBuild + `OptiScaler.sln`** — **not CMake**. The `CMakeLists.txt` in the tree belongs only to `dlssnr/forwarder`, which is also built as `dlssnr_forwarder.vcxproj` from the solution. Toolchain: Visual Studio 2022 (platform toolset `v143`, `/std:c++latest`) plus Windows SDK 10.0.

- **Recommended**: run the bundled GitHub Actions workflow (`.github/workflows/package_release.yml`, dispatched manually). No local toolchain required.
- **Local**: with VS 2022 installed, run `build-release.bat -Version v1.1` from the repository root (it checks out submodules, verifies the hybrid kernels, then compiles and packages). The steps can also be run by hand: `git submodule update --init --recursive` → `get_hybrid_assets.ps1` → `package_release.ps1`.

---

## 12. Licence and compliance

### 12.1 This project

This repository — including all of `OptiScaler/` — inherits its upstream licence and is released under the **GNU GPL-3.0**; the full text is in [`LICENSE`](LICENSE). The wilsjo2 and Coldwood forks carry the identical GPLv3 text, so **merging them creates no licensing conflict**.

GPL-3.0 requires that distributing a binary be accompanied by the corresponding source: this repository publishes the source, and the release page also carries a source archive, which satisfies that requirement.

### 12.2 Third-party components and redistribution (verify this yourself)

The release package **includes** the following third-party binaries. They are **not** GPL and are each governed by their own licence (the corresponding texts are bundled under `Licenses\`):

| Component | Files | Licence text |
|---|---|---|
| Intel XeSS runtime | `OptiScaler\libxess.dll`, `libxess_dx11.dll`, `libxess_fg.dll`, `libxell.dll` | `Licenses\XeSS_LICENSE.txt` |
| AMD FidelityFX | `OptiScaler\amd_fidelityfx_*.dll` | `Licenses\FidelityFX_v2_LICENSE.md` |
| Microsoft DirectX / Agility SDK | `OptiScaler\D3D12_OptiScaler\D3D12Core.dll` | `Licenses\DirectX_LICENSE.txt` |
| RenoDX (MIT; source of the NR colour composition shader) | source: `OptiScaler/shaders/dlssnr/precompile/dlssnr.hlsl` | `Licenses\RenoDX_ATTRIBUTION.txt` |

> ⚠️ Keep two things apart: the **unlock code itself** bundles and modifies no Intel file — it patches in memory only, and not one byte of the on-disk DLL changes. But the **release package** does carry the runtimes listed above, because they come out of the build output. If you would rather not take on that redistribution responsibility, exclude them at packaging time and let users supply their own.

**Not included**: `nvngx_dlssnr.dll` and the DLSS FG runtime (NVIDIA's; the packaging script refuses to include them). Users supply those themselves.

### 12.3 Disclaimer

This project is **for study and research only**. It is not affiliated with, endorsed by or authorised by Intel, NVIDIA, AMD or CD Projekt Red; trademarks are used only to identify what it is compatible with.

The XeMFG unlock targets hardware gating logic inside Intel's runtime and **may conflict with the laws of your jurisdiction or with the relevant software licence terms**. Games and distribution platforms differ in how they treat this. You assume all risk; this project comes with no warranty of any kind.

Do not use it in anti-cheat-protected multiplayer games. The authors are not responsible for bans, data corruption or hardware problems.

---

## 13. Credits

- [optiscaler/OptiScaler](https://github.com/optiscaler/OptiScaler) — the base
- [wilsjo2/OptiScaler-DLSSNR-PreSR-Multipass](https://github.com/wilsjo2/OptiScaler-DLSSNR-PreSR-Multipass) — NR / PreSR / hybrid
- [Coldwood1026/OptiScalerDp4aUnlock](https://github.com/Coldwood1026/OptiScalerDp4aUnlock) — XeMFG unlock / pacing
- [clshortfuse/renodx](https://github.com/clshortfuse/renodx) — source of the NR colour composition shader
- [Dagherbou/OptiScaler_DLSSNR](https://github.com/Dagherbou/OptiScaler_DLSSNR) — origin of the Neural Rendering branch
- [intel/xess](https://github.com/intel/xess) — XeSS SDK
- [optiscaler/OptiPatcher](https://github.com/optiscaler/OptiPatcher) — optional plugin that exposes DLSS/DLSSG inputs without spoofing

The complete third-party list is in `docs/CREDITS.md`.

---

## 14. Reporting issues

Please include: `OptiScaler.log` (already set to Info by default), your GPU and driver version, the game, and the settings you used in the overlay. `SHA256SUMS.txt` inside the release package can be used to verify file integrity.
