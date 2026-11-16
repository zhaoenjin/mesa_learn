# EGL 初始化流程代码走读：eglGetDisplay → eglInitialize → eglCreateContext → eglMakeCurrent

> 基于本地源码 `E:\mesa\mesa-main`（Mesa 27.0.0-devel）
> 以 Android 平台为主线（其他平台在 switch 分支对照）
> 关联文档：《EGLSurface与Gralloc连接分析.md》《DRI_Drawable与BufferQueue映射分析.md》

---

## 0. 总览：四个 API 各自生产什么

```
eglGetDisplay(native)      → _EGLDisplay          （骨架，不碰驱动）
eglInitialize(dpy)         → pipe_screen + EGLConfig   （Gallium 驱动实例诞生）
eglCreateWindowSurface     → _EGLSurface + dri_drawable（骨架，不分配缓冲）
eglCreateContext(...)      → _EGLContext + st_context + pipe_context + gl_context
eglMakeCurrent(...)        → 绑定线程 + 绑定 drawable + 首次 validate
```

完整调用树（行号均为源码实际位置）：

```
eglGetDisplay                     eglapi.c:414
└─ _eglGetNativePlatform          egldisplay.c:162
└─ _eglFindDisplay                egldisplay.c:253

eglInitialize                    eglapi.c:676
└─ _eglDriver.Initialize          = dri2_initialize (egl_dri2.c:888)
   └─ dri2_initialize_android     platform_android.c:1170
      ├─ u_gralloc_create()                    :1176
      ├─ droid_open_device()                   :1093
      │   └─ loader_get_driver_for_fd()        :1020
      ├─ dri2_setup_screen → dri2_create_screen   egl_dri2.c:798
      │   └─ driCreateNewScreen3
      │       └─ pipe_loader_drm_probe_fd      pipe_loader_drm.c:126
      │           └─ driver_descriptors[] 查表  :96 → pipe_screen ★
      ├─ 扩展开关 + u_gralloc_get_front_rendering_usage :1216-1264
      └─ droid_add_configs_for_visuals()       :785 → EGLConfig ★

eglCreateContext                 eglapi.c:882
└─ dri2_create_context           egl_dri2.c:1217
   ├─ _eglInitContext（解析 attribs：API/版本/profile/flags）
   ├─ EGL API → __DRI_API_* 映射                   :1240-1272
   └─ driCreateContextAttribs → dri_create_context dri_context.c:46
       └─ st_api_create_context                  st_manager.c:985
           ├─ pipe_screen->context_create()      ★pipe_context（GPU 命令流）
           └─ st_create_context                  st_context.c:808
               ├─ st_init_driver_functions       （Gallium→swrast 驱动函数表）
               ├─ _mesa_initialize_context       ★gl_context（GL 状态机）
               └─ st_create_context_priv         :478（cso_cache 等）
       └─ （可选）_mesa_glthread_init             dri_context.c:116

eglMakeCurrent                   eglapi.c:937
└─ dri2_make_current             egl_dri2.c
   ├─ _eglBindContext（线程槽位交换，取回 old 绑定）
   ├─ driUnbindContext(old)                     dri_util.c:722
   └─ driBindContext(cctx, ddraw, rdraw)        dri_util.c:688
       └─ dri_make_current                      dri_context.c
          ├─ draw->texture_stamp = lastStamp - 1   ★强制下次 validate 重取
          └─ st_api_make_current                 st_manager.c
              ├─ st_framebuffer_reuse_or_create   （drawable→gl_framebuffer）
              ├─ st_framebuffer_validate          ★首次触发取缓冲
              └─ _mesa_make_current               （GL 状态机上线程绑定）
```

---

## 1. eglGetDisplay：只建骨架，不碰驱动

```c
// src/egl/main/eglapi.c:414
eglGetDisplay(EGLNativeDisplayType nativeDisplay)
{
   // ① 平台探测
   plat = _eglGetNativePlatform(nativeDisplay);        // egldisplay.c:162
   // ② 查重 / 新建
   disp = _eglFindDisplay(plat, native_display_ptr, NULL);  // egldisplay.c:253
   return _eglGetDisplayHandle(disp);
}
```

### 1.1 平台探测（egldisplay.c）

```c
// :93  优先级 1：环境变量强制
_eglGetNativePlatformFromEnv()   // 读 "EGL_PLATFORM"

// :126 优先级 2：按原生指针特征识别
_eglNativePlatformDetectNativeDisplay(nativeDisplay)
//   XOpenDisplay 的 Display*、wl_display、gbm_device、
//   Android 的 EGL_DEFAULT_DISPLAY(NULL)... 各有指纹

// :162 汇总 → _EGL_PLATFORM_ANDROID 等
```

Android 专用入口（`egldisplay.c:697`）：`_eglGetAndroidDisplay()`。

### 1.2 display 对象登记（egldisplay.c:253）

`_eglFindDisplay()` 按 **平台 + 原生句柄** 二元组查全局链表：

- 已存在 → 直接复用（同一 display 只初始化一次）；
- 不存在 → `calloc(_EGLDisplay)` 挂入 `_eglGlobal.DisplayList`。

此时 `disp->Initialized == false`，**驱动加载推迟到 eglInitialize**（懒加载）。

---

## 2. eglInitialize：单一静态驱动 + 三级降级

现代 Mesa 只有一个**编译进 libEGL 的驱动对象**（废弃了旧的"多驱动 .so + dlopen"机制）：

```c
// src/egl/main/eglapi.c:200
extern const _EGLDriver _eglDriver;

// src/egl/drivers/dri2/egl_dri2.c:3170
const _EGLDriver _eglDriver = {
   .Initialize  = dri2_initialize,
   .CreateContext = dri2_create_context,     // :3173
   .MakeCurrent   = dri2_make_current,
   ...
};
```

```c
// src/egl/main/eglapi.c:676
eglInitialize(EGLDisplay dpy, EGLint *major, EGLint *minor)
{
   _eglDeviceRefreshList();                                  // :683 扫描 DRM 设备

   disp->Options.ForceSoftware =                             // :691
      debug_get_bool_option("LIBGL_ALWAYS_SOFTWARE", false);
   disp->Options.Zink =                                      // :697
      (MESA_LOADER_DRIVER_OVERRIDE == "zink");

   if (!_eglDriver.Initialize(disp)) {                       // :708 尝试①：硬件
      // 三级降级链：全部失败才报 EGL_NOT_INITIALIZED
      disp->Options.Zink = true;
      success = _eglDriver.Initialize(disp);                 // :715 尝试②：zink
      if (!success) {
         disp->Options.ForceSoftware = true;
         _eglDriver.Initialize(disp);                        // :720 尝试③：纯软件
      }
   }
   disp->Initialized = TRUE;                                 // :726
}
```

**zink 兜底**的意义：新硬件只有 Vulkan 驱动（如新 NVIDIA 卡）时，OpenGL 走 GL-on-Vulkan 自动续命。

---

## 3. dri2_initialize：按平台分发（egl_dri2.c:888）

```c
// src/egl/drivers/dri2/egl_dri2.c:888
static EGLBoolean dri2_initialize(_EGLDisplay *disp)
{
   // eglTerminate 后再 eglInitialize 的场景：结构还在，只加引用计数
   if (dri2_dpy) { p_atomic_inc(&dri2_dpy->ref_count); return EGL_TRUE; }

   dri2_dpy = dri2_display_create(disp);

   switch (disp->Platform) {                       // eglGetDisplay 阶段就确定了
   case _EGL_PLATFORM_SURFACELESS:  dri2_initialize_surfaceless(disp); break;
   case _EGL_PLATFORM_DEVICE:       dri2_initialize_device(disp);      break;
   case _EGL_PLATFORM_X11/XCB:      dri2_initialize_x11(disp);         break;
   case _EGL_PLATFORM_DRM:          dri2_initialize_drm(disp);         break;
   case _EGL_PLATFORM_WAYLAND:      dri2_initialize_wayland(disp);     break;
   case _EGL_PLATFORM_ANDROID:      dri2_initialize_android(disp);     break;  // ← 主线
   }

   // 硬性成功条件：至少生成一个 EGLConfig
   if (_eglGetArraySize(disp->Configs) == 0) → EGL_NOT_INITIALIZED;
}
```

---

## 4. dri2_initialize_android（platform_android.c:1170）

九个步骤，按源码顺序：

### ① gralloc 抽象层（:1176）

```c
dri2_dpy->gralloc = u_gralloc_create(U_GRALLOC_TYPE_AUTO);
// 后端优先级：CrOS → IMapper4/5 → libdrm → QCOM → fallback
// 详见《EGLSurface与Gralloc连接分析.md》第 4 节
```

### ② 打开 DRM 设备：droid_open_device（:1093）

```c
droid_open_device(disp, ForceSoftware)
{
   // 设备列表来自 eglInitialize 里的 _eglDeviceRefreshList()
   node_type = swrast ? DRM_NODE_PRIMARY : DRM_NODE_RENDER;

   // Android 系统属性：多 GPU 机型指定厂商
   __system_property_get("drm.gpu.vendor_name", vendor_buf);

   遍历 dev_list:
      dri2_dpy->fd_render_gpu = loader_open_device(device->nodes[node_type]);
      //   ↑ src/loader/loader.c:89，实际就是 open()
      若指定 vendor → droid_filter_device() 过滤，匹配后不再回退其他设备
}
```

### ③ 识别驱动名：droid_probe_device（:1075）

```c
dri2_dpy->driver_name = loader_get_driver_for_fd(dri2_dpy->fd_render_gpu);  // :1020
// 内部 ioctl 读内核驱动名并映射：
//   "amdgpu"   → "radeonsi"
//   "i915"/"xe"→ "iris"
//   "msm"      → "freedreno"
//   "panfrost" → "panfrost" ...
```

### ④ 无 GPU 兜底：纯软件（:1195）

```c
if ((!device_opened && ForceSoftware) || force_pure_swrast) {
   dri2_dpy->driver_name = "swrast";
   dri2_dpy->loader_funcs = &droid_swrast_image_loader_funcs;
   dri2_dpy->pure_swrast = true;
   dri2_create_screen(disp);          // 无 fd，软件 screen
}
// force_pure_swrast 由系统属性 MESA_ANDROID_NO_KMS_SWRAST 控制 (:1182)
```

### ⑤ 挂 EGLDevice（:1210）

`dri2_setup_device(disp)`：把 DRM 设备信息挂到 `_EGLDevice`，支撑 `EGL_EXT_device_enumeration`。

### ⑥ 核心：dri2_setup_screen → 创建 pipe_screen

```c
// src/egl/drivers/dri2/egl_dri2.c:798
dri2_create_screen(_EGLDisplay *disp)
{
   enum dri_screen_type type = DRI_SCREEN_DRI3;   // Android 用这个
   //   kopper→KOPPER, swrast_not_kms→SWRAST, KMS swrast→KMS_SWRAST

   dri2_dpy->dri_screen_render_gpu = driCreateNewScreen3(
      0, screen_fd, dri2_dpy->loader_funcs, type, &dri2_dpy->driver_configs, ...);
}
```

`driCreateNewScreen3`（`src/gallium/frontends/dri/dri_screen.c`）内部：

```
driCreateNewScreen3
  → pipe_loader_drm_probe_fd(fd)                    // pipe_loader_drm.c:126
       ├─ get_driver_descriptor(driver_name)        // :96 查静态描述符表
       │    driver_descriptors[] = { radeonsi, iris, freedreno, panfrost,
       │                              zink, asahi, lima, kmsro, ... }
       │    （Android 单体库：驱动全部静态链入 libgallium_dri.so）
       │    或 Linux 发行版布局：dlopen("<name>_dri.so")   // loader.c:925
       └→ 驱动描述符的 create_screen()
            └→ struct pipe_screen ★（Gallium 驱动实例）
  → __DRI 扩展协商（__DRI_IMAGE / __DRI2_renderbuffer 等版本匹配）
  → driver_configs 返回 DRI 层的 GL 配置（供第⑧步转成 EGLConfig）
```

**驱动选路两种布局**：

| 布局 | 机制 | 位置 |
|---|---|---|
| Android（单体） | 所有 Gallium 驱动静态链入一个库，按 driver_name 查 `driver_descriptors[]` 表 | `pipe_loader_drm.c:96` |
| Linux 发行版 | 查不到表时按搜索路径 `dlopen("<driver>_dri.so")` | `loader.c:925` |

### ⑦ Android 扩展开关（:1216 起）

```c
dri2_setup_swap_interval(disp, 1);        // Android 最大 swap interval 固定 1

disp->Extensions.ANDROID_framebuffer_target  = TRUE;
disp->Extensions.ANDROID_image_native_buffer = TRUE;
disp->Extensions.ANDROID_recordable          = TRUE;

// buffer_age 依赖 native fence：否则 QuerySurface 里 dequeue 会死锁
if (ANDROID_native_fence_sync)  EXT_buffer_age = TRUE;
else                            KHR_partial_update = FALSE;

// :1264 前向渲染 usage 要问 gralloc 要
if (!u_gralloc_get_front_rendering_usage(gralloc, &front_rendering_usage)) {
   dri2_dpy->front_rendering_usage = front_rendering_usage;
   disp->Extensions.KHR_mutable_render_buffer = TRUE;   // 仅 ES 场景
}
```

### ⑧ 生成 EGLConfig：droid_add_configs_for_visuals（:785）

按 Android 的 `HAL_PIXEL_FORMAT_*`（RGBA_8888 / RGBX_8888 / RGB_565 ...）遍历：

```
每个 visual × {ES1, ES2, ES3, GL} × {sRGB, 线性}
  → 从 driver_configs 匹配 DRI config
  → 填 _EGLConfig（NativeVisualID = HAL format）
  → 加入 disp->Configs
```

### ⑨ 挂 vtable（:1298）

```c
dri2_dpy->vtbl = &droid_display_vtbl;   // 最后才挂，防止初始化中途被虚函数误调
```

---

## 5. eglCreateContext：三层上下文对象的诞生

### 5.1 EGL 入口（eglapi.c:882）

```c
eglCreateContext(dpy, config, share_list, attrib_list)
{
   _EGLConfig *conf   = _eglLookupConfig(config, disp);
   _EGLContext *share = _eglLookupContext(share_list, disp);

   // 校验：config 合法 / share 必须同 display（EGL_BAD_MATCH）

   context = disp->Driver->CreateContext(disp, conf, share, attrib_list);
   return _eglLinkContext(context);          // 挂进 display 的资源表
}
```

### 5.2 dri2 层（egl_dri2.c:1217）

```c
dri2_create_context(disp, conf, share_list, attrib_list)
{
   dri2_ctx = malloc(...);
   _eglInitContext(&dri2_ctx->base, disp, conf, share_list, attrib_list);
   //  ↑ 解析 attribs → ClientAPI / 版本 / profile / flags / reset策略 / no_error
   //    存入 _EGLContext

   // ① EGL API → __DRI API 映射 (:1240)
   switch (dri2_ctx->base.ClientAPI) {
   case EGL_OPENGL_ES_API:
      版本1→__DRI_API_GLES; 2→__DRI_API_GLES2; 3→__DRI_API_GLES3;
   case EGL_OPENGL_API:
      ≥3.1 或 core profile → __DRI_API_OPENGL_CORE;
      其他 → __DRI_API_OPENGL;
   }

   // ② 选 DRI config（优先双缓冲变体 dri_config[1][0]）(:1274)

   // ③ 填 __DRI 上下文属性（版本/flags/reset/no_error/优先级...）
   dri2_fill_context_attribs(dri2_ctx, dri2_dpy, ctx_attribs, &num_attribs);

   // ④ 进入 DRI 核心
   dri2_ctx->dri_context = driCreateContextAttribs(
      dri2_dpy->dri_screen_render_gpu, api, dri_config, shared,
      num_attribs/2, ctx_attribs, &error, dri2_ctx, thread_safe);   // :1309
}
```

### 5.3 DRI 核心层（dri_context.c:46）

```c
dri_create_context(screen, attribs, visual, &error, sharedContextPrivate, ...)
{
   ctx = CALLOC_STRUCT(dri_context);
   ctx->loaderPrivate = loaderPrivate;        // 反指 dri2_ctx

   // MESA_NO_ERROR 环境变量/driconf 处理 (:76)

   ctx->st = st_api_create_context(&screen->base, attribs, &ctx_err, st_share);
   //  ↑ :85 关键一跳，进入 st/mesa

   ctx->st->frontend_context = ctx;

   // GALLIUM_HUD（若启用）
   ctx->hud = hud_create(ctx->st->cso_context, ...);   // :104

   // glthread：driconf 开启且 loader 线程安全才起 (:109-116)
   if (enable_glthread)
      _mesa_glthread_init(ctx->st->ctx);
}
```

### 5.4 st/mesa 层（st_manager.c:985）：两个重量级对象诞生

```c
st_api_create_context(fscreen, attribs, error, shared_ctx)
{
   _mesa_initialize(attribs->options.mesa_extension_override);  // 全局一次

   // ★① pipe_context —— Gallium 驱动的命令流上下文
   pipe = fscreen->screen->context_create(fscreen->screen, NULL,
                                          PIPE_CONTEXT_PREFER_THREADED |
                                          lod_bias_flag | context_flags);
   //   由 pipe_screen 的驱动实现（如 radeonsi/iris）创建
   //   PIPE_CONTEXT_PREFER_THREADED → 可能包成 tc（threaded context）

   // ★② st_context → gl_context —— GL 状态机
   return st_create_context(api, pipe, mode, share, options, no_error, ...);
   //   ↓ st_context.c:808
}

st_create_context(...)                          // st_context.c:808
{
   st_init_driver_functions(pipe->screen, &funcs, ...);
   //  ↑ 把 Gallium 能力翻译成 swrast（软件光栅）可调的函数表

   ctx = align_malloc(sizeof(struct gl_context), 16);
   ctx->pipe = pipe;                            // gl_context 反指 pipe_context

   _mesa_initialize_context(ctx, api, no_error, visual, shareCtx, &funcs, options);
   //  ★③ struct gl_context —— classic Mesa 的核心 GL 状态机
   //    初始化全部 GL 状态、扩展表（按驱动 caps 裁剪）、共享列表（shareCtx）

   ctx->Cache = pipe->screen->get_disk_shader_cache(...);   // 磁盘着色器缓存

   st = st_create_context_priv(ctx, pipe, options);          // :478
   //    cso_cache（状态对象缓存）、vbo 模块、ATI_fragment 等
}
```

**三层上下文关系**（创建完成时）：

```
_EGLContext (EGL API 层对象, eglapi 层管理)
  └─ dri2_egl_context
       └─ dri_context (dri_context.c)
            ├─ loaderPrivate ──→ 反指 dri2_egl_context
            └─ st ──→ st_context (st/mesa 桥)
                 ├─ ctx ──→ gl_context        ★GL 状态机（所有 GL 状态在这）
                 │    └─ pipe ──→ pipe_context ★Gallium 命令流（提交 GPU 工作）
                 │         └─ screen ──→ pipe_screen（初始化阶段产物）
                 ├─ cso_context               状态对象缓存
                 └─ frontend_drawable 绑定区（MakeCurrent 填）
```

---

## 6. eglMakeCurrent：绑定与首次取缓冲

### 6.1 EGL 入口（eglapi.c:937）

```c
eglMakeCurrent(dpy, draw, read, ctx)
{
   // 查对象 + 一长串校验：
   //  - display 未初始化时只允许全部为 NULL（释放场景）
   //  - surfaceless 检查（无 surface 绑 context 需要 KHR_surfaceless_context）
   //  - draw/read 的 Lost 检查（EGL_BAD_NATIVE_WINDOW）
   //  - protected content 匹配检查（EGL_BAD_ACCESS）
   //  - API 匹配检查：context 的 ClientAPI 必须等于 eglBindAPI 设的值

   // 同一个 context 已在别的线程 → EGL_BAD_ACCESS

   ret = disp->Driver->MakeCurrent(disp, draw_surf, read_surf, context);
   //  成功后线程槽位 _eglGlobal.CurrentDisplay/Context/DrawSurface/ReadSurface 更新
}
```

### 6.2 dri2 层（egl_dri2.c dri2_make_current）：先解绑旧的再绑新的

```c
dri2_make_current(disp, dsurf, rsurf, ctx)
{
   // ① EGL 层槽位交换：新绑定生效，取回 old_ctx/old_dsurf/old_rsurf
   _eglBindContext(ctx, dsurf, rsurf, &old_ctx, &old_dsurf, &old_rsurf);

   // 幂等优化：完全相同的绑定直接成功返回

   if (old_ctx) {
      // ② 旧 surface 若在 shared buffer mode → 先退出
      if (old_dsurf 在共享缓冲模式)
         old_dri2_dpy->vtbl->set_shared_buffer_mode(old_disp, old_dsurf, false);

      // ③ 解绑旧 context
      driUnbindContext(old_cctx);              // dri_util.c:722

      // ④ 旧 surface 的 out_fence 同步
      if (old_dsurf) dri2_surf_update_fence_fd(old_ctx, old_disp, old_dsurf);
   }

   // ⑤ surface → dri_drawable
   ddraw = dsurf ? dri2_surface_get_dri_drawable(dsurf) : NULL;
   rdraw = rsurf ? dri2_surface_get_dri_drawable(rsurf) : NULL;
   cctx  = ctx   ? dri2_ctx->dri_context : NULL;

   // ⑥ 真正绑定
   if (!driBindContext(cctx, ddraw, rdraw)) {
      // 失败回滚：恢复旧绑定（_eglBindContext 反向 + driBindContext(old...)）
      // 仍失败则置线程无 context 状态
   }
}
```

### 6.3 DRI 核心层（dri_context.c dri_make_current）

```c
dri_make_current(ctx, draw, read)
{
   _mesa_glthread_finish(ctx->st->ctx);     // glthread 收尾（不能跨线程用 st）

   if (!draw && !read)                       // surfaceless：只绑 context
      return st_api_make_current(ctx->st, NULL, NULL);

   ctx->draw = draw;  ctx->read = read;
   dri_get_drawable(draw);                   // drawable->refcount++ ★

   draw->texture_stamp = draw->lastStamp - 1;   // ★★关键：强制失效
   //  下次 st_framebuffer_validate 必然发现 texture_stamp != lastStamp
   //  → 重走 getBuffers → dequeue + 导入 + bind_images
   //  （详见《DRI_Drawable与BufferQueue映射分析.md》）

   return st_api_make_current(ctx->st, &draw->base, &read->base);
}
```

### 6.4 st/mesa 层（st_manager.c st_api_make_current）：GL 状态机上线程

```c
st_api_make_current(st, stdrawi, streadi)
{
   // ① drawable → gl_framebuffer（一个 drawable 可被多个 context 绑过，
   //    各 context 各有一份 gl_framebuffer 视图，哈希表复用/新建）
   stdraw = st_framebuffer_reuse_or_create(st, stdrawi);
   stread = (不同) ? st_framebuffer_reuse_or_create(st, streadi) : 复用 stdraw;

   // ② ★首次 validate：此刻真正触发取缓冲
   //    （dequeue gralloc buffer → 导入 → 绑进 textures[BACK_LEFT]）
   st_framebuffer_validate(stdraw, st);
   if (stread != stdraw)
      st_framebuffer_validate(stread, st);

   // ③ classic Mesa 的线程绑定：gl_context 成为当前线程的 GL 上下文
   ret = _mesa_make_current(st->ctx, stdraw, stread);
   //   内部：绑定 glapi 调度表、初始化默认状态、
   //   首绑时设置初始 viewport/scissor = surface 尺寸

   // ④ 记录 stamp 供下次比对
   st->draw_stamp = stdraw->stamp - 1;
   st->read_stamp = stread->stamp - 1;

   // ⑤ 清理已销毁 drawable 的残留 framebuffer
   st_framebuffers_purge(st);
}
```

无 surface 的 surfaceless 分支：绑一个全局 incomplete framebuffer，纯离屏渲染用。

---

## 7. 全流程时序（一次完整的 Android GL 启动）

```
App                          libEGL                 DRI/Gallium              框架侧
 │ eglGetDisplay ────────────→ 登记 _EGLDisplay                                 │
 │ eglInitialize ────────────→ pipe_screen★, EGLConfigs★                        │
 │ eglChooseConfig ──────────→ 选一个 config                                     │
 │ eglCreateWindowSurface ───→ dri_drawable（骨架，setUsage）────────→ BufferQueue记usage
 │ eglCreateContext ──────────→ pipe_context★, gl_context★（st_context 桥接）   │
 │ eglMakeCurrent ───────────→ 解绑旧/绑新 drawable                             │
 │        │                    首次 validate ──────────────── dequeue ────────→ gralloc分配
 │        │                    ←── dri_image(pipe_resource) ←── u_gralloc(fd) ─│
 │        │                    _mesa_make_current（GL 状态机就绪）              │
 │ glDrawArrays... ──────────→ st/mesa → pipe_context → GPU                    │
 │ eglSwapBuffers ───────────→ flush + queueBuffer(fence) ──────────────────→ SurfaceFlinger
```

---

## 8. 环境变量 / 系统属性速查

| 变量 | 作用 | 位置 |
|---|---|---|
| `EGL_PLATFORM` | 强制平台类型 | egldisplay.c:93 |
| `LIBGL_ALWAYS_SOFTWARE` | 强制软件渲染 | eglapi.c:692 |
| `MESA_LOADER_DRIVER_OVERRIDE` | 指定驱动名（如 zink） | eglapi.c:697 |
| `GALLIUM_DRIVER` | 指定 Gallium 驱动 | dri2 内部 |
| `MESA_NO_ERROR` | 开 KHR_no_error（非 setuid 时） | dri_context.c:76 |
| driconf `mesa_glthread` | 启用 glthread | dri_context.c:109 |
| Android 属性 `drm.gpu.vendor_name` | 多 GPU 选厂商 | platform_android.c:1093 |
| Android 属性 `MESA_ANDROID_NO_KMS_SWRAST` | 禁 KMS 软件渲染 | platform_android.c:1182 |

---

## 9. 关键文件速查

| 文件 | 内容 |
|---|---|
| `src/egl/main/eglapi.c` | `eglGetDisplay`(:414) / `eglInitialize`(:676) / `eglCreateContext`(:882) / `eglMakeCurrent`(:937) |
| `src/egl/main/egldisplay.c` | 平台探测、display 登记/查重 |
| `src/egl/drivers/dri2/egl_dri2.c` | `_eglDriver`(:3170)、`dri2_initialize`(:888)、`dri2_create_screen`(:798)、`dri2_create_context`(:1217)、`dri2_make_current` |
| `src/egl/drivers/dri2/platform_android.c` | `dri2_initialize_android`(:1170) 等 Android 全部逻辑 |
| `src/loader/loader.c` | `loader_open_device`(:89)、`loader_get_driver_for_fd`、`dlopen` 驱动(:925) |
| `src/gallium/auxiliary/pipe-loader/pipe_loader_drm.c` | `driver_descriptors[]`(:96)、`probe_fd`(:126) |
| `src/gallium/frontends/dri/dri_screen.c` | `driCreateNewScreen3`：扩展协商、pipe_screen 创建 |
| `src/gallium/frontends/dri/dri_context.c` | `dri_create_context`(:46)、`dri_make_current`（stamp 强制失效） |
| `src/gallium/frontends/dri/dri_util.c` | `driBindContext`(:688)、`driUnbindContext`(:722) |
| `src/mesa/state_tracker/st_manager.c` | `st_api_create_context`(:985)、`st_api_make_current` |
| `src/mesa/state_tracker/st_context.c` | `st_create_context`(:808)、`st_create_context_priv`(:478) |

---

## 10. 一句话总结

`eglGetDisplay` 建骨架；`eglInitialize` 经静态 `_eglDriver` 按"硬件→zink→软件"三级降级初始化出 **pipe_screen + EGLConfig**；`eglCreateContext` 自上而下穿过 EGL→DRI→st/mesa 三层，产出 **pipe_context + gl_context**（由 st_context 桥接）；`eglMakeCurrent` 完成"先解旧、再绑新"，把 drawable 的 `texture_stamp` 置为失效强制下次 validate 重取缓冲，最终 `_mesa_make_current` 让 GL 状态机在当前线程就绪——此后 `glDrawArrays` 即可一路下行走 st/mesa → pipe_context → GPU。
