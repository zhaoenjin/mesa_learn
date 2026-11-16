# EGLSurface 创建流程与 Android Gralloc 的连接分析

> 基于本地源码 `E:\mesa\mesa-main`（Mesa 27.0.0-devel）
> 核心文件：
> - `src/egl/drivers/dri2/platform_android.c`（Android 平台 EGL 后端）
> - `src/util/u_gralloc/`（gralloc 抽象层）
> - `src/gallium/frontends/dri/dri2.c`（dmabuf 导入）

---

## 0. 先说结论（纠正常见误解）

**Mesa 不直接调用 gralloc 分配窗口缓冲区。**

gralloc 的 **分配（Allocator）** 发生在 Android 框架侧：`ANativeWindow`（底层是 BufferQueue / BLASTBufferQueue，属 libgui）在 dequeue 时通过 `GraphicBufferAllocator → IAllocator HAL` 完成分配。

Mesa 与 gralloc 体系只发生**两次握手**：

| # | 连接点 | gralloc 角色 | 代码位置 |
|---|---|---|---|
| 1 | `ANativeWindow_setUsage(gralloc_usage)` — 告诉框架侧"按这个 usage 分配" | Allocator（间接，由框架执行） | `platform_android.c:344` |
| 2 | `u_gralloc_get_buffer_basic_info()` — 拿到 gralloc buffer handle 后查询元数据（fourcc/modifier/fd/stride/offset） | **Mapper**（Mesa 直接调用） | `platform_android.c:85` |

之后 Mesa 把 mapper 查出的 fd 以 **dmabuf** 方式导入 Gallium 驱动，GPU 直接渲染到 gralloc 分配的内存上。

```
┌─────────────── App / EGL 侧（Mesa）───────────────┐   ┌──────── Android 框架侧 ────────┐
│                                                      │   │                                 │
│  eglCreateWindowSurface                              │   │                                 │
│    │                                                 │   │                                 │
│    ├─ ANativeWindow_setUsage(GRALLOC_USAGE_HW_RENDER)│──→│  记住 usage                     │
│    │                                                 │   │                                 │
│  （第一次绘制）                                        │   │                                 │
│    ├─ ANativeWindow_dequeueBuffer() ─────────────────│──→│  BufferQueueProducer            │
│    │                                                 │   │    └→ GraphicBufferAllocator   │
│    │←──── ANativeWindowBuffer{handle,format,stride} ─│───│         └→ gralloc IAllocator ★分配
│    │                                                 │   │                                 │
│    ├─ u_gralloc 查询 handle ─────────────────────────│   │                                 │
│    │    （gralloc IMapper/mapper HAL）                │──→│  GraphicBufferMapper            │
│    │←──── fourcc / modifier / fds / strides ─────────│───│                                 │
│    │                                                 │   │                                 │
│    ├─ dri2_from_dma_bufs(fds...) → resource_from_handle (Gallium 驱动导入 dmabuf)          │
│    │                                                 │   │                                 │
│  eglSwapBuffers                                      │   │                                 │
│    └─ ANativeWindow_queueBuffer(buf, fence) ─────────│──→│  SurfaceFlinger 合成            │
└──────────────────────────────────────────────────────┘   └─────────────────────────────────┘
```

---

## 1. 阶段〇：EGLDisplay 初始化时就创建了 gralloc 封装

`eglGetDisplay(EGL_DEFAULT_DISPLAY)` 在 Android 上走 `dri2_initialize_android()`：

```c
// platform_android.c:1167
EGLBoolean dri2_initialize_android(_EGLDisplay *disp)
{
   ...
   dri2_dpy->gralloc = u_gralloc_create(U_GRALLOC_TYPE_AUTO);   // :1176
   ...
   device_opened = droid_open_device(disp, disp->Options.ForceSoftware);  // 打开 DRM 设备
   // 打不开 GPU 时退回纯软件: driver_name = "swrast"
}
```

- gralloc 对象挂在 `dri2_egl_display` 上，**display 生命周期内全局复用**（`u_gralloc_create` 内部有缓存+引用计数，见 `u_gralloc.c:36`）。
- `U_GRALLOC_TYPE_AUTO` = 按优先级逐个尝试后端（见第 4 节）。

## 2. 阶段一：eglCreateWindowSurface

入口 `src/egl/main/eglapi.c` 做参数校验后分发到驱动的 `droid_create_window_surface()`（`:376`），两者都汇入 `droid_create_surface()`（`:272`）：

```c
static _EGLSurface * droid_create_surface(...)   // platform_android.c:272
{
   // 1) 读窗口属性
   format = ANativeWindow_getFormat(window);                         // :298
   ANativeWindow_query(window, ANATIVEWINDOW_QUERY_MIN_UNDEQUEUED_BUFFERS, ...); // :307
   buffer_count = min_undequeued_buffers + 2;                        // :315 本地缓存槽数
   ANativeWindow_query(window, ANATIVEWINDOW_QUERY_DEFAULT_WIDTH/HEIGHT, ...); // :330

   // 2) ★决定 gralloc usage —— Mesa 对 gralloc 分配的唯一影响手段
   dri2_surf->gralloc_usage =
      (swrast/kms_swrast 驱动)
        ? GRALLOC_USAGE_SW_READ_OFTEN | GRALLOC_USAGE_SW_WRITE_OFTEN  // CPU 访问
        : GRALLOC_USAGE_HW_RENDER;                                    // :335 GPU 渲染
   if (ActiveRenderBuffer == EGL_SINGLE_BUFFER)
      dri2_surf->gralloc_usage |= dri2_dpy->front_rendering_usage;    // :342

   // 3) ★连接点①：把 usage 告诉 BufferQueue
   ANativeWindow_setUsage(window, dri2_surf->gralloc_usage);          // :344
   //    此后框架侧每次分配 buffer 都按这个 usage 调 gralloc

   // 4) 创建 DRI drawable，注册回调（不分配任何图形缓冲）
   dri2_create_drawable(dri2_dpy, config, dri2_surf, dri2_surf);      // :357
   //    loader 回调 = droid_image_get_buffers (:549)

   ANativeWindow_acquire(window);                                     // :361
}
```

要点：
- **此时没有分配任何图形缓冲**，只完成"配置 + 回调注册"。
- `front_rendering_usage` 是驱动上报的（比如需要 frontbuffer rendering 的 usage 位），单缓冲/共享缓冲模式会用到。
- `color_buffers[buffer_count]` 数组只是本地记录 buffer 指针，用于 swap 时计算 **buffer age**（`platform_android.c:153-160`）。

## 3. 阶段二：第一次绘制 → dequeue → mapper 查询 → dmabuf 导入

DRI 驱动在拿到 drawable 后第一次取缓冲时，回调 `droid_image_get_buffers()`（`:549`）：

### 3.1 取 buffer（分配发生在框架侧）

```c
// platform_android.c:139
droid_window_dequeue_buffer(dri2_surf)
{
   ANativeWindow_dequeueBuffer(dri2_surf->window, &dri2_surf->buffer, &fence_fd);
   // ↑ 框架侧：BufferQueueProducer 不足则分配（GraphicBufferAllocator → gralloc IAllocator）
   //   返回的 ANativeWindowBuffer 包含：handle(native_handle_t*) / format / stride / width / height
   // 记录到 color_buffers[] 用于 buffer age
}
```

`ANativeWindowBuffer` 的 `handle` 就是 **gralloc buffer handle**（一个 binder/native_handle，内含 fd 数组与整数属性数组）。

### 3.2 ★连接点②：u_gralloc（mapper）解析元数据

```c
// platform_android.c:71
droid_create_image_from_native_buffer(disp, buf, priv)
{
   struct u_gralloc_buffer_handle gr_handle = {
      .handle = buf->handle,          // gralloc native handle
      .hal_format = buf->format,      // HAL_PIXEL_FORMAT_*
      .pixel_stride = buf->stride,
   };

   u_gralloc_get_buffer_basic_info(dri2_dpy->gralloc, &gr_handle, &buf_info);   // :85
   // → buf_info: drm_fourcc / modifier / num_planes / fds[] / strides[] / offsets[]

   u_gralloc_get_buffer_color_info(dri2_dpy->gralloc, &gr_handle, &color_info); // :89
   // → YUV 色彩空间 / 采样范围 / 色度采样位置

   img = droid_create_image_from_buffer_info(...);   // → dri2_from_dma_bufs()
   if (!img) {   // virgl 等虚拟化驱动的兜底：去掉 modifier 重试一次
      buf_info.modifier = DRM_FORMAT_MOD_INVALID;    // :103
      img = droid_create_image_from_buffer_info(...);
   }
}
```

mapper 后端（IMapper5）实际查询代码（`u_gralloc_imapper5_api.cpp:52`）：

```c
GraphicBufferMapper::get().getPixelFormatFourCC(hnd->handle, &drm_fourcc);
GraphicBufferMapper::get().getPixelFormatModifier(hnd->handle, &modifier);
GraphicBufferMapper::get().getAllocationSize(...);
GraphicBufferMapper::get().getPlaneLayouts(...);   // 每个 plane 的 fd/stride/offset
```

### 3.3 dmabuf 导入 Gallium 驱动

```c
// src/gallium/frontends/dri/dri2.c:1301
dri2_from_dma_bufs(dri_screen_render_gpu, width, height,
                   fourcc, modifier, fds, num_planes, strides, offsets, ...)
{
   // 校验 fourcc↔modifier 匹配的 plane 数
   // 把 fds[]/strides[]/offsets[] 组装成 winsys_handle[]（WINSYS_HANDLE_TYPE_FD）
   // → dri_image_from_handle → pipe_screen::resource_from_handle()
   //    由具体驱动（如 radeonsi/freedreno/turnip winsys）执行 DRM GEM 导入
}
```

导入成功后 `dri2_surf->dri_image_back` 指向这块与 **SurfaceFlinger 共享**的 gralloc 内存；随后 `handle_in_fence_fd()`（`:112`）把 dequeue 返回的 fence 设进 image，保证等上游写完才开始画。

### 3.4 pbuffer / front buffer（对照）

| 情形 | 是否经 gralloc |
|---|---|
| 窗口 surface 后台缓冲 | 是（框架分配 + mapper 导入） |
| pbuffer | **否**。`droid_create_pbuffer_surface` 传 `native_window=NULL`（`:387`），缓冲由 `dri_create_image()` 直接从 DRI 驱动分配（`get_front_bo`，`:484`） |
| 窗口 surface front buffer | EGL 规范不支持，仅打日志（`:474`） |

## 4. u_gralloc 抽象层（src/util/u_gralloc/）

统一接口屏蔽 gralloc 版本碎片化，按优先级自动选择后端（`u_gralloc.c:25`）：

| 优先级 | 后端 | 文件 | 说明 |
|---|---|---|---|
| 1 | `U_GRALLOC_TYPE_CROS` | `u_gralloc_cros_api.c` | ChromeOS 的 gralloc4 扩展 API，最快（避免 IMapper metadata 的 binder 往返） |
| 2 | `U_GRALLOC_TYPE_GRALLOC4` | `u_gralloc_imapper4/5_api.cpp` | AOSP 标准 IMapper metadata（`GraphicBufferMapper`），Android 11+ |
| 3 | `U_GRALLOC_TYPE_LIBDRM` | `u_gralloc_libdrm.c` | 传统 gralloc0/1 + HAL format→fourcc 映射表 |
| 4 | `U_GRALLOC_TYPE_QCOM` | `u_gralloc_qcom.c` | 高通私有 gralloc1 扩展 |
| 5 | `U_GRALLOC_TYPE_FALLBACK` | `u_gralloc_fallback.c` | 按 `HAL_PIXEL_FORMAT_*` 纯查表猜测的兜底 |

输出结构（`u_gralloc.h`）：

```c
struct u_gralloc_buffer_basic_info {
   uint32_t drm_fourcc;
   uint64_t modifier;
   int num_planes;
   int fds[4]; uint64_t strides[4]; uint64_t offsets[4];
   ...
};
struct u_gralloc_buffer_color_info { /* YUV colorspace/range/siting */ };
```

## 5. Swap 阶段（回到框架侧）

```
eglSwapBuffers
  → dri2 flush（驱动把 GPU 工作 + out fence 提交）
  → droid_window_enqueue_buffer (:187)
       ANativeWindow_queueBuffer(window, buffer, fence_fd)   // :199
       → BufferQueue 入队 → SurfaceFlinger/HWC 合成上屏
  → 下一帧 droid_window_dequeue_buffer 时按 buffer 指针更新 buffer age
```

Shared buffer mode（`EGL_KHR_mutable_render_buffer`，`droid_set_shared_buffer_mode :230`）时会切 `ANativeWindow_setSharedBufferMode(true)` 并给 usage 追加 `front_rendering_usage`，单块缓冲直接给合成器，`droid_image_get_buffers` 走 `__DRI_IMAGE_BUFFER_SHARED` 分支（`:562`）。

## 6. 一图流总结

```
eglCreateWindowSurface ─→ setUsage(HW_RENDER) ─→ [框架: BufferQueue 记下 usage]
        │
首帧 getBuffers 回调
        ├─ dequeueBuffer ─────────→ [框架: gralloc IAllocator 分配] → ANativeWindowBuffer(handle)
        ├─ u_gralloc (IMapper) ───→ fourcc/modifier/fds/strides/offsets
        ├─ dri2_from_dma_bufs ────→ resource_from_handle → pipe_resource (GPU 渲染目标)
        │
eglSwapBuffers ─→ queueBuffer(fence) ─→ [SurfaceFlinger 合成]
```

## 7. 关键文件速查

| 文件 | 内容 |
|---|---|
| `src/egl/main/eglapi.c` | EGL 入口与参数校验、API 分发 |
| `src/egl/drivers/dri2/platform_android.c` | Android 平台后端：surface 创建/dequeue/导入/swap 全流程 |
| `src/egl/drivers/dri2/egl_dri2.{c,h}` | DRI2 EGL 通用框架（display/driver 装载） |
| `src/util/u_gralloc/` | gralloc mapper 多后端抽象 |
| `src/gallium/frontends/dri/dri2.c` | `dri2_from_dma_bufs` 等 dmabuf 导入实现 |
| `src/gallium/frontends/dri/dri_drawable.c` | `dri_create_drawable` / getBuffers 回调机制 |
