# Mesa 3D 架构分析

> 基于本地源码 `E:\mesa\mesa-main`（版本 **27.0.0-devel**，主线开发版）
> 所有路径均相对 `src/`。

---

## 1. Mesa 是什么

Mesa 是开源的 OpenGL / Vulkan / OpenCL 图形驱动实现框架，包含：

- **OpenGL / OpenGL ES 实现**（classic Mesa + Gallium 两套路径）
- **多个厂商的 Vulkan ICD 驱动**（RADV、ANV、NVK、Turnip 等）
- **GLSL / SPIR-V → NIR 编译器基础设施**
- 一批辅助前端（视频加速 VA-API、计算 OpenCL/Rusticl、张量 Teflon/Torx 等）

---

## 2. 总体分层

```
 应用程序
    │
    ├──────────────────────┬───────────────────────────────┐
    │ OpenGL / GLES        │ Vulkan                        │
    ▼                      ▼                               │
 libGL / libEGL        Vulkan Loader (系统提供)            │
 (src/glx, src/egl)        │ 按=json/ICD manifest 枚举       │
    │                      ▼                               │
 glapi 调度层           各厂商 ICD 驱动                     │
 (src/mesa/glapi)      (src/*/vulkan)                      │
    │                      │                               │
    ▼                      ▼                               │
 classic Mesa 状态机    vk_common 共享运行时                │
 或 st/mesa 状态跟踪器   (src/vulkan/runtime + wsi)         │
    │                      │                               │
    ▼                      ▼                               │
 Gallium pipe 接口      NIR → 各硬件后端编译器              │
 (src/gallium)         (ACO / brw / NAK / ...)             │
    │                      │                               │
    ▼                      ▼                               │
 硬件驱动 + winsys ──→ DRM 内核驱动 (Linux) / D3D12 / Metal │
```

核心思想：**API 前端**把 OpenGL/Vulkan 语义翻译成中间表示（Gallium pipe 指令 / NIR），**驱动后端**只面对稳定的中间层，不用关心 API 细节。

---

## 3. OpenGL 链路

### 3.1 调用流向

```
 glDrawArrays()
   │
   ▼
 libGL (GLX, src/glx) 或 libEGL (src/egl)     ← 窗口系统入口
   │
   ▼
 glapi/mapi 调度 (src/mesa/glapi)              ← 按上下文分发到具体驱动
   │
   ▼
 src/mesa/  classic Mesa 核心                   ← OpenGL 状态机
   ├── main/      核心 GL 状态（bufferobj/texture/blend/draw...数百个文件）
   ├── vbo/       顶点缓冲与顶点获取
   ├── program/   固定功能顶点/片段程序的模拟
   ├── math/      数学库
   └── glapi/     GL 入口点表
   │
   ▼
 src/mesa/state_tracker/  (st/mesa)            ← 把 GL 状态翻译成 Gallium 管线状态
   │
   ▼
 Gallium pipe 接口 (src/gallium)               ← 与硬件无关的驱动抽象层
   pipe_screen / pipe_context / pipe_resource
   │
   ▼
 具体驱动 (src/gallium/drivers/*)
   │
   ▼
 winsys (src/gallium/winsys/*)                 ← 对接内核 DRM/驱动接口
```

两条特殊路径：
- **纯软件渲染**：classic 路径直接走 `src/mesa/main` + swrast；Gallium 路径走 `llvmpipe`（LLVM JIT，常用）或 `softpipe`（纯 C 参考实现）。
- **Zink**：OpenGL 实现"跑在 Vulkan 上"（`drivers/zink`），GL 调用最终翻译成 Vulkan 命令，是 GL/Vulkan 两条链路的交汇点。

### 3.2 窗口系统入口

| 目录 | 职责 |
|---|---|
| `src/glx/` | Linux/X11 的 GLX 接口（libGL），含 DRI2/DRI3、swrast 后端 |
| `src/egl/` | EGL 实现（libEGL），`main/` 主逻辑 + `drivers/`（dri2、haiku 等）+ Wayland 支持；桌面 Linux 经 glvnd 分发 |
| `src/gbm/` | 通用缓冲管理（无窗口系统的 allocate/scanout） |
| `src/gallium/frontends/wgl`、`dri`、`hgl` | Windows WGL、DRI（X11/Wayland 通用）、Haiku 前端 |

### 3.3 Gallium 结构（src/gallium）

| 子目录 | 内容 |
|---|---|
| `frontends/` | **API 前端**：`dri`（GL 主力前端）、`lavapipe`（软渲染 Vulkan）、`rusticl`（Rust 写的 OpenCL）、`teflon`（TensorFlow Lite NPU 前端）、`torx`（ExecuTorch/EXIR 前端）、`va`（VA-API 视频解码）、`wgl`、`mediafoundation`、`hgl` |
| `auxiliary/` | **公共设施**：`util`、`nir`（NIR 通用 pass）、`tgsi`、`gallivm`（LLVM 代码生成，供 llvmpipe）、`draw`（软件顶点处理/软光栅）、`cso_cache`（状态对象缓存）、`pipebuffer`、`tessellator`、`vl`（视频）、`driver_trace/noop/ddebug`（调试包装驱动） |
| `drivers/` | 所有 Gallium 驱动（见下表） |
| `winsys/` | 每个驱动的内核接口薄层 |

### 3.4 OpenGL 驱动清单（src/gallium/drivers + classic）

**硬件驱动（Gallium）**

| 驱动 | 硬件 | 说明 |
|---|---|---|
| `iris` | Intel Gen8+ / Xe | 当前 Intel GL 主力 |
| `crocus` | Intel Gen4–Gen11 | 老平台维护 |
| `i915` | Intel Gen2–Gen6（经典 i915/i965 时代） | 很老的硬件 |
| `radeonsi` | AMD GCN→RDNA（SI+） | AMD GL 主力，用 ACO/LLVM 编译 |
| `r600` | AMD R600–Northern Islands | 老硬件 |
| `r300` | ATI R300–R500 | 老硬件 |
| `nouveau` | NVIDIA NV04–Maxwell(GM20x) | 老卡 GL；新卡走 NVK(Vulkan)+Zink |
| `freedreno` | Qualcomm Adreno（a2xx–a6xx） | 与 Turnip（Vulkan）同源 |
| `panfrost` | ARM Mali Midgard/Bifrost/Valhall | |
| `lima` | ARM Mali 400/450 (Utgard) | |
| `v3d` | Raspberry Pi VideoCore VI/VII | |
| `vc4` | Raspberry Pi VideoCore IV | |
| `etnaviv` | Vivante GC 系列 | |
| `asahi` | Apple M 系列 GPU（AGX） | Linux 逆向驱动，GL 侧 |
| `svga` | VMware 虚拟 GPU | |
| `virgl` | virtio-gpu 虚拟化（主机转发） | |
| `d3d12` | Windows D3D12（WDDM vGPU/WSL） | GL 翻译到 D3D12 |
| `tegra` | NVIDIA Tegra（配合 nouveau） | 辅助 |
| `zink` | 任意 Vulkan 驱动之上 | **GL on Vulkan** |

**软件驱动**

| 驱动 | 说明 |
|---|---|
| `llvmpipe` | LLVM JIT 软光栅化（现代默认软渲染，也支撑 lavapipe） |
| `softpipe` | 纯 C 参考实现，慢但可调试 |
| swrast | classic 路径软件渲染器（随 `src/mesa`） |

**NPU/张量驱动（非图形）**

| 驱动 | 说明 |
|---|---|
| `ethosu` | Arm Ethos-U NPU（配合 teflon 前端） |
| `rocket` | Google 的 ML 加速器驱动（rkt_ml，2026 新增） |

---

## 4. Vulkan 链路

### 4.1 调用流向

Vulkan 没有类似 Gallium 的强制抽象，而是"**共享运行时 + 各自独立 ICD**"模式：

```
 vkCmdDraw()
   │
   ▼
 Vulkan Loader（系统组件，按 ICD json 枚举驱动）
   │
   ▼
 厂商 ICD（如 radv/anv/nvk...，各自实现完整 vk* 入口）
   │  ├─ 继承 vk_common 基类（src/vulkan/runtime）
   │  │    vk_device/vk_instance/vk_command_buffer/vk_buffer...
   │  │    动态渲染、descriptor、同步、acceleration structure(RT)、RMV 显存分析
   │  └─ WSI（src/vulkan/wsi）: X11 / Wayland / Win32 / Android / display / headless / Metal
   │
   ▼
 Shader: SPIR-V → NIR (src/compiler/spirv + nir)
   │
   ▼
 各硬件后端编译器（ACO / Intel IR / NAK / ...）
   │
   ▼
 命令提交：内核 DRM（amdgpu/i915/nouveau/...）或 D3D12 / Metal
```

### 4.2 共享层（src/vulkan）

| 目录 | 职责 |
|---|---|
| `runtime/` | **vk_common 层**：所有驱动继承的基类与公共实现（设备、命令缓冲、descriptor、buffer/image、同步、动态渲染、光线追踪 BVV/BVH、内存分析 rmv 等），约占每个驱动 50%+ 的通用逻辑 |
| `wsi/` | 窗口系统集成：`wsi_common_x11/wayland/win32/display/drm/headless/metal` |
| `util/` | vk_util 基础工具宏 |
| `overlay-layer/` | FPS/帧时间覆盖层 |
| `device-select-layer/` | 多 GPU 设备选择层 |
| `screenshot-layer/`、`vram-report-limit-layer/` | 截图与显存限制层 |
| `registry/` | Vulkan 注册数据 |

### 4.3 Vulkan 驱动（ICD）清单

| 驱动 | 目录 | 硬件/平台 | 说明 |
|---|---|---|---|
| **RADV** | `src/amd/vulkan` | AMD GCN→RDNA4 | 社区主力，ACO 编译器，光线追踪、ray querying |
| **ANV** | `src/intel/vulkan` | Intel Gen9+ / Xe | Intel 官方+社区维护 |
| **hasvk** | `src/intel/vulkan_hasvk` | Intel Gen7–Gen8.5 | 从 ANV 拆出的老卡维护版 |
| **NVK** | `src/nouveau/vulkan` | NVIDIA Turing+ | 新一代 N 卡驱动（配合 NVIDIA 开源内核模块）；同目录 `src/nouveau/` 下有 compiler(NAK)、nil(图像布局)、mme(宏引擎) |
| **Turnip (tu)** | `src/freedreno/vulkan` | Adreno a5xx–a7xx | 与 freedreno GL 同源，Android 主力之一 |
| **panvk** | `src/panfrost/vulkan` | Mali Midgard/Bifrost | |
| **v3dv** | `src/broadcom/vulkan` | Raspberry Pi V3D | |
| **PVR** | `src/imagination/vulkan` | Imagination PowerVR Rogue | |
| **Venus** | `src/virtio/vulkan` | virtio-gpu 虚拟化 | 把 Vulkan 命令流转发给主机 |
| **Dozen (dzn)** | `src/microsoft/vulkan` | Windows/WSL D3D12 | Vulkan 翻译到 D3D12 |
| **lavapipe (lvp)** | `src/gallium/frontends/lavapipe` | CPU（基于 llvmpipe） | 软件 Vulkan |
| **Honeykrisp (hk)** | `src/asahi/vulkan` | Apple M 系列（AGX） | Asahi 逆向的 Vulkan |
| **kosmickrisp (kk)** | `src/kosmickrisp` | Apple 平台（Metal 桥接） | 2026 新增：跑在 Metal 上的 Vulkan，`bridge/` 为 ObjC/Metal 层 |
| **gfxstream** | `src/gfxstream` | Android 模拟器 | 虚拟图形流 |

### 4.4 Vulkan 与 OpenGL 链路的差异总结

| 维度 | OpenGL | Vulkan |
|---|---|---|
| 抽象方式 | 统一状态机（classic/st）+ Gallium pipe 抽象 | 无统一抽象，各 ICD 独立实现 + vk_common 基类复用 |
| 入口分发 | glapi/mapi 按上下文跳转 | Vulkan Loader 按 ICD manifest 加载 |
| Shader 输入 | GLSL（`src/compiler/glsl`） | SPIR-V（`src/compiler/spirv`） |
| 中间表示 | NIR（经 TGSI 逐步淘汰，现已基本全 NIR） | NIR |
| 窗口对接 | GLX/EGL/ WGL 各自实现 | 统一 WSI 层 |
| 代码组织 | 集中在 `src/mesa` + `src/gallium` | 分散在各厂商目录 `src/*/vulkan` + 共享 `src/vulkan` |

---

## 5. 编译器基础设施（src/compiler）

Mesa 的通用编译器栈（GL 和 Vulkan 共用）：

| 目录 | 内容 |
|---|---|
| `glsl/` | GLSL → NIR 前端（OpenGL 用） |
| `spirv/` | SPIR-V → NIR（Vulkan / OpenCL 用） |
| `nir/` | **NIR**（NIR 是几乎所有路径的枢纽：优化 pass、lowering、调度） |
| `clc/`、`libcl/` | OpenCL C 编译支持 |
| `isaspec` | ISA 描述/反汇编框架 |

各硬件后端编译器（NIR → 机器码）分散在厂商目录：

| 后端 | 位置 | 用途 |
|---|---|---|
| **ACO** | `src/amd/compiler` | RADV/radeonsi 的新一代着色器编译器 |
| Intel backend (brw) | `src/intel/compiler` | ANV/hasvk/iris，含 EU 指令发射 |
| **NAK** | `src/nouveau/compiler` | NVK 的 Rust 编译器 |
| asahi compiler | `src/asahi/compiler` | Apple AGX |
| Microsoft | `src/microsoft/compiler`（DXIL）、`spirv_to_dxil` | d3d12 驱动/dozen |
| freedreno/ir3 | `src/freedreno/ir3` | Turnip + freedreno GL |
| Rust 基础 | `src/compiler/rust` | Rust 互操作支持 |

其他关键共享目录：`src/util`（通用数据结构、u_math、disk_cache、driconf 配置）、`src/loader`（DRI 驱动探测加载）、`src/drm-shim`（无硬件调试）。

---

## 6. 两条链路的交汇点

1. **Zink**（`src/gallium/drivers/zink`）：GL 状态机 → Gallium → Vulkan 命令。没有原生 GL 驱动的平台（如新 NVIDIA 卡）用它跑 GL。
2. **lavapipe**：反方向——Vulkan API → Gallium pipe → llvmpipe 软件执行。
3. **NIR**：GLSL 与 SPIR-V 都汇入 NIR，再进各硬件后端。
4. **EGL**：GL 和 VG 的共同入口；EGL 设备扩展也能枚举 Vulkan 设备。
5. Venus/Virgl/gfxstream：虚拟化场景下两套 API 分别走不同转发路径。

---

## 7. 速查：一条命令看懂目录

```
OpenGL:  src/glx + src/egl (入口) → src/mesa/glapi (调度)
         → src/mesa/main (状态机) → src/mesa/state_tracker (GL→pipe)
         → src/gallium/drivers/* (驱动) → src/gallium/winsys/* (内核)

Vulkan:  src/vulkan/runtime (共享基类) + src/vulkan/wsi (窗口)
         → src/amd/vulkan | src/intel/vulkan | src/nouveau/vulkan | ... (各 ICD)
         → src/compiler/spirv→nir → 各厂商后端编译器
```

---

## 8. 版本与状态备注

- 本树为 **27.0.0-devel** 主线快照（2026-11），无 `.git` 历史（zip 解压）。
- 相比旧资料的新变化：`vulkan_hasvk` 拆分、NVK 成熟为默认 N 卡 Vulkan、`kosmickrisp`（Metal 上的 Vulkan）与 `rocket`（Google NPU）等新成员、`torx`/`teflon` 等 NPU 前端、TGSI 基本被 NIR 取代。
- 构建系统为 **Meson**（根目录 `meson.build` + `meson.options`），驱动按 `-Dgallium-drivers=` / `-Dvulkan-drivers=` 选择。
