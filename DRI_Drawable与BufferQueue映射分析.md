# dri_drawable 概念及其 texture 与 BufferQueue 的映射分析

> 基于本地源码 `E:\mesa\mesa-main`（Mesa 27.0.0-devel）
> 前置阅读：《EGL初始化流程分析.md》《EGLSurface与Gralloc连接分析.md》
> 核心文件：
> - `src/gallium/frontends/dri/dri_drawable.{c,h}`（drawable 定义与回调）
> - `src/gallium/frontends/dri/dri2.c`（DRI2/DRI3 路径实现、invalidate 机制）
> - `src/egl/drivers/dri2/egl_dri2.c`（EGL 侧创建）
> - `src/egl/drivers/dri2/platform_android.c`（dequeue/queue/导入）

---

## 0. 一句话总结

`dri_drawable` 是 DRI 前端中"一个可渲染目标"（窗口/pbuffer/pixmap）的代表，是 **st/mesa 状态机与窗口系统之间的适配器**；它的 `textures[ST_ATTACHMENT_BACK_LEFT]` 槽位与 gralloc BufferQueue 的 N 块轮转 buffer **不是静态绑定，而是每帧重绑定**——任意时刻该槽位指向"当前 dequeue 到的那一块"。

---

## 1. dri_drawable 是什么：分层定位

```
EGLSurface (libEGL, src/egl/drivers/dri2)          ← EGL API 层的对象
    │  内含 dri2_surf->dri_drawable
    ▼
dri_drawable (src/gallium/frontends/dri/dri_drawable.h:44)
    │  继承 pipe_frontend_drawable（st/mesa 认的抽象基类）
    ▼
st/mesa 状态机 (src/mesa/state_tracker)             ← GL 状态机在这层跑
    │  只认 pipe_frontend_drawable 接口
    ▼
Gallium 驱动 (pipe_context/pipe_resource)
```

st/mesa 渲染时需要"往哪儿画"的缓冲，但它**不认识** ANativeWindow/X window/Wayland surface。`dri_drawable` 就是中间的转换层：向上面向 st/mesa 提供统一回调接口，向下通过 `loaderPrivate` 指回 EGL 侧的 `dri2_egl_surface`。

### 1.1 结构体定义（dri_drawable.h:44）

```c
struct dri_drawable {
   struct pipe_frontend_drawable base;   // st/mesa 侧的接口（虚函数表）
   struct st_visual stvis;               // 颜色格式/深度模板配置

   struct dri_screen *screen;

   struct pipe_resource *textures[ST_ATTACHMENT_COUNT];        // ★真正的渲染缓冲槽位
   struct pipe_resource *msaa_textures[ST_ATTACHMENT_COUNT];   // MSAA 版本
   unsigned texture_mask, texture_stamp;   // 哪些 attachment 有效 / 是否需重取

   void *loaderPrivate;    // ★反向指回 EGL 侧的 dri2_egl_surface
   int refcount;           // 被 N 个 context 绑定就 N 个引用
   unsigned lastStamp;     // 失效计数：loader 调 invalidate 就 ++
   int w, h;
   struct pipe_fence_handle *throttle_fence;  // 帧节流（swap interval 用）
   ...
};
```

关键成员：
| 成员 | 作用 |
|---|---|
| `textures[ST_ATTACHMENT_COUNT]` | front/back/depth/stencil 等 `pipe_resource` 槽位，GL 最终画进的 GPU 内存 |
| `loaderPrivate` | 指回 EGL 的 `dri2_egl_surface`（Android 上含 ANativeWindow、dequeue 出的 gralloc buffer） |
| `lastStamp` | 失效通知：resize/换 buffer 时递增，触发下次 validate 重取 |
| `texture_mask` | 位图：哪些 attachment 当前有效 |
| `refcount` | 多 context 共享同一 drawable 的引用计数 |

### 1.2 创建：eglCreateWindowSurface 阶段只建骨架

```
eglCreateWindowSurface
  → dri2_create_drawable (egl_dri2.c:1413)
  → dri_create_drawable (dri_drawable.c:210)
       ├─ loaderPrivate = dri2_surf（EGL surface）
       ├─ 挂三个回调给 st/mesa：
       │    base.validate          = dri_st_framebuffer_validate   ★核心
       │    base.flush_front       = dri_st_framebuffer_flush_front
       │    base.flush_swapbuffers = dri_st_framebuffer_flush_swapbuffers
       └─ 按 screen 类型分发初始化 (dri_drawable.c:240)
            DRI_SCREEN_DRI3      → dri2_init_drawable   (Android/桌面 DRI3)
            DRI_SCREEN_SWRAST    → drisw_init_drawable  (纯软件)
            DRI_SCREEN_KOPPER    → kopper_init_drawable (Zink/Vulkan)
```

**此时不分配任何缓冲**。缓冲获取推迟到绘制发生时的 validate 回调（惰性）。

### 1.3 生命周期

| 阶段 | 动作 |
|---|---|
| 创建 | `dri_create_drawable`，refcount=1，纯骨架 |
| 绑定 | `eglMakeCurrent` → `driBindContext`，refcount++ |
| 失效 | resize/换 buffer → loader 调 `dri_invalidate_drawable` → `lastStamp++` |
| 销毁 | refcount 归零 → 释放所有 `textures[]`、fence、通知 st manager |

---

## 2. 核心机制：validate 回调（惰性取缓冲）

st/mesa 在绘制前调 `validate` 说"我要 BACK_LEFT 和深度缓冲"，DRI 侧实现（`dri_drawable.c` 的 `dri_st_framebuffer_validate`）：

```c
do {
   lastStamp = drawable->lastStamp;
   new_stamp = (drawable->texture_stamp != lastStamp);

   if (new_stamp || new_mask) {
      if (new_stamp)
         dri_drawable_update_info(drawable);      // 向 loader 查尺寸等

      dri_drawable_allocate_textures(ctx, drawable, statts, count);
      //  ↑ Android 场景内部最终调到 loader 的 getBuffers
      //    = platform_android.c:549 droid_image_get_buffers

      drawable->texture_stamp = lastStamp;
      drawable->texture_mask = statt_mask;
   }
} while (lastStamp != drawable->lastStamp);       // 期间又失效则重来
```

---

## 3. texture ↔ BufferQueue buffer 的映射：每帧重绑定

BufferQueue 默认三缓冲（A/B/C 轮转），而 `textures[BACK_LEFT]` 只有一个槽位。对应关系靠 **"queue → 销毁 image → invalidate → 下次 validate 重新 dequeue+导入+bind"** 的循环维持，每帧执行一次。

### 3.1 帧内绑定链（frame N）

```
st/mesa validate(BACK_LEFT)
  → dri2_allocate_textures                    dri2.c:160
  → dri_image_drawable_get_buffers            dri2.c:94
       （把 ST_ATTACHMENT_* 翻译成 __DRI_IMAGE_BUFFER_* 掩码）
  → loader 回调 droid_image_get_buffers       platform_android.c:549
       ├─ update_buffers (:440)
       │    dri2_surf->buffer == NULL 时才 dequeue
       │      → 拿到轮转中某一块 ANativeWindowBuffer（gralloc handle）
       ├─ get_back_bo (:497)
       │    dri_image_back == NULL 时才导入：
       │      ANativeWindowBuffer.handle → u_gralloc 查 fourcc/fd/stride/offset
       │        → dri2_from_dma_bufs (dri2.c:1301)
       │           → resource_from_handle → pipe_resource ★
       │              包进 dri_image->texture
       └─ images->back = dri_image_back
  → dri_drawable_bind_images                   dri_drawable.c ★绑定发生处
       pipe_resource_reference(&drawable->textures[ST_ATTACHMENT_BACK_LEFT],
                               images->back->texture);
       // 槽位指向新 texture；旧 texture 引用计数 -1
```

对应关系是三层包装：

```
textures[ST_ATTACHMENT_BACK_LEFT]      (dri_drawable 槽位)
   └─ 引用 images->back->texture       (dri_image，EGL 侧创建)
        └─ resource_from_handle 返回值  (Gallium 驱动对 dmabuf fd 的包装)
             └─ 底层 = dequeue 到的那块 gralloc buffer 的内存
```

### 3.2 swap 时的换槽（frame N → N+1）

`droid_swap_buffers`（platform_android.c:616）三步完成轮转：

```c
① droid_window_enqueue_buffer (:184)
     ANativeWindow_queueBuffer(buffer, out_fence);   // :199 还给 BufferQueue
     dri2_surf->buffer = NULL;                        // :201
     dri2_destroy_image(dri2_surf->dri_image_back);   // :205 ★image 直接销毁
     dri2_surf->dri_image_back = NULL;

② dri_invalidate_drawable (dri2.c:85，由 platform_android.c:663 调用)
     drawable->lastStamp++;
     drawable->texture_mask = 0;    // ★所有 color attachment 标记失效
     p_atomic_inc(&drawable->base.stamp);

③ （下一帧）validate 发现 texture_stamp != lastStamp
     → 重走 3.1 整条链 → dequeue 拿到下一块（B）
     → bind_images 把 textures[BACK_LEFT] 换成 B 的 texture
```

三缓冲下的实际轮转效果：

```
BufferQueue（SurfaceFlinger 侧）      Mesa 侧
┌────┐ ┌────┐ ┌────┐
│ A  │ │ B  │ │ C  │   三块 gralloc buffer 轮转
└─┬──┘ └─┬──┘ └────┘
  │dequeue│
  ▼      │
 textures[BACK_LEFT] ←── 每帧 bind_images 重绑
  frame N: 指向 A ──queue A──→ 指向 B ──queue B──→ 指向 C ──→ 指向 A ...
```

### 3.3 stamp 在 Android 上的特殊性

`dri_image_drawable_get_buffers`（dri2.c:94）的注释明确说明：**Android/Wayland/Device/Surfaceless 路径 stamp 参数不使用**（DRI3 桌面路径才用）。Android 的失效完全靠 `droid_swap_buffers` 主动调 `dri_invalidate_drawable`（`texture_mask = 0`）驱动。

---

## 4. 三个关键细节

### 4.1 引用计数保证安全交接

`pipe_resource_reference` 换槽时旧 texture 引用 -1，但 GPU 可能还在画它。内存安全由内核 dmabuf/GEM 引用保证：EGL 侧销毁 `dri_image` 只是丢掉包装，底层内存在 GPU 工作（out fence 签收）前不会被回收或复用，SurfaceFlinger 收到 buffer 后按 fence 等待。

### 4.2 depth/stencil 与 gralloc 无关

`dri2_allocate_textures`（dri2.c:196 附近）明确注释 **"depth-stencil buffer 可以复用，不删除"**：

| attachment | 来源 | 跨帧行为 |
|---|---|---|
| BACK_LEFT（color） | BufferQueue 的 gralloc buffer（dequeue+导入） | 每帧换 |
| FRONT_LEFT（color） | 同上（仅 pbuffer/特殊场景） | 每帧换 |
| DEPTH_STENCIL | Gallium 驱动自 `resource_create` 分配 | 复用到 resize |

BufferQueue 里只有 color buffer；深度模板是驱动私有内存。

### 4.3 一帧没画也不出错

如果一帧内没有任何 GL 绘制命令，validate 不会被触发，`dri2_surf->buffer == NULL`。`droid_swap_buffers`（platform_android.c:656 附近注释）专门处理了这种情况：跳过 enqueue，不报错。

### 4.4 同步：in/out fence

- dequeue 返回 in_fence_fd → `handle_in_fence_fd`（platform_android.c:112）设进 image，GPU 等 HWC/上一个消费者写完才动笔
- flush 产生 out_fence_fd → 随 `queueBuffer` 交给 SurfaceFlinger（:199）

---

## 5. 与其他 screen 类型的对照

同一套 `dri_drawable` 框架，三种后端的 `textures[]` 来源不同（`dri_drawable.c:240` 分发）：

| screen 类型 | 实现文件 | textures 来源 | 对应窗口系统机制 |
|---|---|---|---|
| `DRI_SCREEN_DRI3` | `dri2.c` | dmabuf 导入 | Android: BufferQueue+gralloc；桌面: DRI3/Present |
| `DRI_SCREEN_SWRAST` | `drisw.c` | CPU 内存 | 软件拷贝交换 |
| `DRI_SCREEN_KOPPER` | `kopper.c` | Vulkan image | Zink 路径 |

---

## 6. 关键文件速查

| 文件 | 内容 |
|---|---|
| `src/gallium/frontends/dri/dri_drawable.h:44` | `struct dri_drawable` 定义 |
| `src/gallium/frontends/dri/dri_drawable.c:210` | `dri_create_drawable`（创建+挂回调） |
| `src/gallium/frontends/dri/dri_drawable.c` | `dri_st_framebuffer_validate`（惰性取缓冲）、`dri_drawable_bind_images`（★槽位绑定） |
| `src/gallium/frontends/dri/dri2.c:85` | `dri_invalidate_drawable`（lastStamp++/texture_mask=0） |
| `src/gallium/frontends/dri/dri2.c:160` | `dri2_allocate_textures`（DRI3 路径分配） |
| `src/gallium/frontends/dri/dri2.c:94` | `dri_image_drawable_get_buffers`（stamp 语义注释） |
| `src/egl/drivers/dri2/egl_dri2.c:1413` | `dri2_create_drawable`（EGL 侧入口） |
| `src/egl/drivers/dri2/platform_android.c` | dequeue/enqueue/swap/invalidate 全流程 |
