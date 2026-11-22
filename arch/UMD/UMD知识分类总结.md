# GPU 用户态驱动（UMD）知识分类总结

> 面向 MLSys / AI 芯片与 GPU 软硬件协同、编译器与框架、LLM 加速读者的分类调研。  
> 整理日期：2026 年 11 月 22 日（Asia/Shanghai）。  
> 范围：公开资料中的 UMD 实际项目经验、官方文档、开源实现，以及 XDC / FOSDEM / LPC 等可核验讲座。  
> 方法：每条关键论断尽量给出可点击来源；链接若 404 则舍弃或替换；403 / 超时若标题与内容已由其他途径核验，则保留并注明。不收录无法核验或明显注水的转载。本文是分类总结，不是链接堆砌，也不粘贴外部版权文章原文。

---

## 目录

1. [定位与总体图景](#1-定位与总体图景)
2. [UMD / KMD 的职责边界](#2-umd--kmd-的职责边界)
3. [命令提交与 command buffer / queue](#3-命令提交与-command-buffer--queue)
4. [内存管理：显存、VA、SVM / HMM 与虚拟内存 API](#4-内存管理显存va-svm--hmm-与虚拟内存-api)
5. [同步：fence、semaphore 与 syncobj](#5-同步fencesemaphore-与-syncobj)
6. [UMD 中的 shader / 编译器](#6-umd-中的-shader--编译器)
7. [各厂商实现与开源项目经验](#7-各厂商实现与开源项目经验)
8. [调试、性能与 AI 推理 / 算子栈的关联](#8-调试性能与-ai-推理--算子栈的关联)
9. [学习路径与可动手实践](#9-学习路径与可动手实践)
10. [文章与视频索引](#10-文章与视频索引)
11. [完整参考目录](#11-完整参考目录)
12. [资料边界与未完全核验项](#12-资料边界与未完全核验项)

---

## 1. 定位与总体图景

在现代 GPU 软件栈里，**用户态驱动（User Mode Driver，UMD）是把高层 API（Vulkan、OpenGL、CUDA Driver API、Level Zero、HIP/HSA 等）翻译成硬件可执行命令的那一层**。它运行在非特权的用户进程里，通常以共享库形式被加载：Linux 上常见如 `libvulkan_radeon.so`（RADV）、`libcuda.so`（NVIDIA CUDA Driver API 的实现库）、Intel compute-runtime 的 Level Zero / OpenCL 实现；Windows 上则是 WDDM 模型中由 Direct3D runtime 加载的 UMD DLL。

Collabora 的 Tyr 系列把分工说得很干脆：KMD 把一块大得多的 UMD 接到真正的 GPU 上；UMD 实现 Vulkan / OpenGL / OpenCL 等 API，负责描述几何、纹理、shader 和工作负载，并构造 `VkCommandBuffer` 这类「让 GPU 做事的指令结构」；KMD 负责把工作提交到硬件队列、在异步乱序完成时维护依赖，以及设备上电与初始化。该文并强调：**复杂度的大头通常在 UMD**——它要把高层 API 翻译成 GPU 能懂的低级命令。[Almeida, Collabora, 2025-08-06](https://www.collabora.com/news-and-blog/blog/2025/08/06/writing-a-rust-gpu-kernel-driver-a-brief-introduction-on-how-gpu-drivers-work/)

Mesa 官方 RADV 文档给出了同一图景的「可操作版本」：所有图形 API 都在用户态实现；RADV 是配合 Linux `amdgpu` KMD 工作的 UMD，经 winsys 与内核对话。KMD 负责 PCIe 访问与电源管理、显示、显存与 GTT，以及把已提交命令写入 GPU ring；UMD 负责以 GPU 能理解的二进制格式记录命令、编程寄存器、把 shader 编译到原生 ISA 并上传到 GPU 可访问内存，再通过系统调用提交。[Mesa RADV 文档](https://docs.mesa3d.org/drivers/radv.html)

对做 MLSys 的读者，可以把一次算子执行粗看成：

```
框架 / 运行时（PyTorch、ggml、Triton、CUDA Runtime、HIP…）
        │  分配、流/队列、kernel launch、图捕获
        ▼
UMD（libcuda、RADV/ANV/NVK、compute-runtime、ROCr/libhsakmt…）
        │  ISA 编译或模块加载、命令编码、VA/描述符布局、提交与同步
        ▼
KMD（nvidia.ko、amdgpu/KFD、Xe/i915、Dxgkrnl+miniport…）
        │  地址落实、调度/抢占、驻留、权限与隔离、中断
        ▼
固件 + 硬件（GSP / GuC / CP/MEC、doorbell、PBDMA、计算/拷贝引擎）
```

CUDA 官方编程指南把边界说得更「API 化」：CUDA Runtime 写在更低层的 CUDA Driver API 之上；Driver API 实现于随设备驱动安装的 `cuda.dll` / `cuda.so`，入口以 `cu` 为前缀。多数应用不必直接碰 Driver API 也能跑满性能，但部分新接口会先出现在 Driver API，**虚拟内存管理（VMM）等高级接口只在 Driver API 暴露**。[CUDA Programming Guide §3.3，v13.5](https://docs.nvidia.com/cuda/cuda-programming-guide/03-advanced/driver-api.html)

因此，框架里看到的 `cudaMalloc`、`cudaLaunchKernel`、`cudaGraphLaunch`，以及 Vulkan 计算路径上的 `vkCmdDispatch` / `vkQueueSubmit`，最终都要经过 UMD 变成命令流、地址映射和同步对象。UMD 的实现质量，会直接表现为：**提交延迟、命令条数、显存碎片、shader 编译卡顿、以及图（Graph）能否少走主机往返**。

---

## 2. UMD / KMD 的职责边界

不同操作系统和厂商的用词不同（UMD/KMD、winsys、thunk、miniport），但反复出现的边界是稳定的。

### 2.1 几乎总在 UMD 一侧

- **实现用户可见 API**，包括状态跟踪（pipeline、descriptor、context）。RADV 把 Vulkan entry point 转到 `radv_*`；ANV 则把大量动态状态的打包策略写进驱动内部文档。[RADV](https://docs.mesa3d.org/drivers/radv.html)；[ANV](https://docs.mesa3d.org/drivers/anv.html)
- **把 API 命令编码成硬件命令包**（AMD 上是 PM4 / command stream，NVIDIA 上是 pushbuffer / method stream，Intel 上是按代际打包的指令）。
- **编译或加载 shader / kernel**：SPIR-V/GLSL/DXIL/NIR → 原生 ISA，或加载预编译模块（CUDA `cuModuleLoad`、Level Zero 模块）。
- **决定资源布局**（tiling、descriptor 格式、binding table），再请求 KMD 做出分配。
- **在传统模型里发起提交**：调用 ioctl / `pfnRenderCb` 等，把 command buffer 交给内核。

### 2.2 几乎总在 KMD 一侧

- **设备初始化、电源、显示、特权寄存器**。Tyr 文把调度（决定何时执行、保证依赖）和设备 bring-up 明确归为 KMD 职责。[Almeida, 2025](https://www.collabora.com/news-and-blog/blog/2025/08/06/writing-a-rust-gpu-kernel-driver-a-brief-introduction-on-how-gpu-drivers-work/)
- **物理显存与驻留（residency）**、GTT/系统内存迁移。RADV 文档写明 video memory 与 GTT 由 KMD 负责。[RADV](https://docs.mesa3d.org/drivers/radv.html)
- **跨进程安全与校验**：传统提交路径上，KMD 会校验命令并做地址 patch。WDDM 操作流程里，KMD 的 `DxgkDdiRender` 校验 command buffer、按硬件格式写入 DMA buffer，并生成 allocation list；随后由 `Dxgkrnl` 做分页、patch 物理地址再提交。[WDDM Operation Flow](https://learn.microsoft.com/en-us/windows-hardware/drivers/display/windows-vista-and-later-display-driver-model-operation-flow)
- **中断与完成通知**，以及与 OS 调度器、内存管理器的接合（Linux 的 DRM scheduler / dma_fence，Windows 的 VidSch / VidMm）。

### 2.3 正在前移的那条线：用户态提交

边界不是冻结的。为了降低提交延迟，越来越多的栈允许 **UMD（甚至 GPU 自己）直接往固件 ring / doorbell 写**，内核只做队列创建、特权内存和生命周期管理：

- Windows WDDM 3.2（Windows 11 24H2 仍标注为开发中的特性）引入 **user-mode work submission**：应用/UMD 可从用户态直接向 GPU 提交，避免每次都经过内核；硬件队列用 doorbell。传统内核提交仍须同时保留。该设计文档还写明：其低延迟路径上「KMD 并不介入、也不感知每一次提交」；`D3DKMTNotifyWorkSubmission` 只会在少数必须通知 KMD 的场景使用，因为每次往返都会抵消低延迟目标。[User-mode work submission](https://learn.microsoft.com/en-us/windows-hardware/drivers/display/user-mode-work-submission)
- Linux `amdgpu` **User Mode Queues** 文档写得同样直接：让用户进程直接管理向 GPU engine 的提交，**绕过用于提交的 ioctl**，既降开销，也允许 GPU 向自己提交；UMD 通过每应用共享内存与固件交互，向 ring 写入 IP 特定包，更新 `wptr`，由固件按 `rptr`/`wptr` 消费。[Kernel doc: User Mode Queues](https://www.kernel.org/doc/html/latest/gpu/amdgpu/userq.html)
- NVIDIA 闭源 UMD（`libcuda`）在开源内核模块与近年论文中被描述为：把 PCIe BAR 映射进用户地址空间，从而 **直接写命令与 doorbell，不必为每次提交陷入内核**。[Yan, Karlin, Grant, arXiv:2604.26889, 2026-04-29](https://arxiv.org/abs/2604.26889)

一句话：**KMD 仍掌握信任与资源，UMD 越来越接近「命令的作者和门铃的按下者」。** 对 LLM 推理这种短 kernel、高提交频率的负载，这条路径往往比「再快一点的 kernel 计算」更先成为瓶颈。

---

## 3. 命令提交与 command buffer / queue

### 3.1 记录：API 调用变成硬件包

Vulkan 应用把命令记进 command buffer，再提交到队列。RADV 的实现是向一块称为 command stream（CS）的缓冲区发射 packet；packet 大体对应 Vulkan 命令，但若硬件与规范不完全同构，一条 API 会变成更复杂的一串包。提交时，若干 CS 作为内核所称的 **indirect buffer（IB）** 一次性交给内核——名称来自内核常用 `INDIRECT_BUFFER` 这条 PM4 命令去执行它们。现代 AMD GPU 有多条队列，大致映射到 Vulkan queue 类型，有时一次提交要同时打到多条硬件队列。包最终由该队列的 **Command Processor（CP）** 执行。[RADV：Command submission and PM4 packets](https://docs.mesa3d.org/drivers/radv.html)

Intel ANV 的文档则展示了另一类 UMD 技巧：为降低 3D pipeline 状态的重复打包，非动态状态在 `genX_pipeline.c` 预打包，动态状态在 `genX_gfx_state.c` 往已打包指令上补丁；间接绘制还可以用 **generation shader** 在 side batch 里生成主 command buffer 中的指令。[ANV 文档](https://docs.mesa3d.org/drivers/anv.html)

NVIDIA 侧没有同等的官方「UMD 如何写 pushbuffer」公开手册，但开源内核模块与独立研究给出了一致的机械图：

- 驱动先把高层 API（例如一次 `cudaMemcpy`）翻译成 copy/DMA 描述符，写入 **pushbuffer**（每条 4 字节）。
- 通过 **GPFIFO** 把这段命令流交给 PBDMA 引擎。
- 用户态可写的 **USERD** 用来更新生产者指针 `GP_PUT`（含 `GP_PUT` 的 RAMFC 位于用户态不能直接访问的特权区）。
- 写 doorbell 后，PBDMA 从 USERD 载入最新 `GP_PUT` 并开始消费。[arXiv:2604.26889](https://arxiv.org/abs/2604.26889)；开源内核模块中的 doorbell kickoff 注释同样是「先把 `GP_PUT` 写入 USERD，再向 HOST doorbell 写 channel token」，见 [NVIDIA/open-gpu-kernel-modules](https://github.com/NVIDIA/open-gpu-kernel-modules) 中 `nvidia-push.c` 的 `DoorbellKickoff` 注释。

Maxime Peim 的 `libmc` 把同一条路径做成了可运行的最小库：它不链接 `libcuda` / `libcudart`，直接走 RM/UVM 接口；在 Hopper 及以上，甚至可以让 **一个 SM 线程** 写另一个 channel 的 pushbuffer、发布 GPFIFO、推进 USERD `GP_PUT` 并敲 BAR1 doorbell，从而在没有 host 介入的情况下启动 copy engine。项目说明强调：这不是 device-side `cudaMemcpyAsync`（后者仍交给驱动调度），而是由该线程自己撰写命令流并按铃。该能力依赖 `HOPPER_USERMODE_A` / `BLACKWELL_USERMODE_A` 这类可映射的 usermode aperture；Ampere 的 `AMPERE_USERMODE_A` 没有这一概念。[libmc README](https://github.com/maxime-peim/libmc)

### 3.2 提交：ioctl 路径 vs doorbell 路径

**传统 ioctl 路径**（仍是兼容与校验的基线）：

1. UMD 在用户内存里填满 command buffer。
2. 通过 DRM uAPI、`libdrm`、KFD ioctl 或 WDDM runtime callback 提交。
3. KMD 校验、做地址 patch / 分页，把 IB 或 DMA buffer 挂到硬件/固件 ring。
4. 硬件完成后以中断或 fence 信号通知。

WDDM 流程是这一模型的完整标本：Draw 调用进入 UMD；Present/Flush 或 command buffer 满时，UMD 调 `pfnPresentCb` / `pfnRenderCb`；KMD 的 `DxgkDdiPresent` / `DxgkDdiRender` 校验并生成 DMA buffer 与 allocation list；`Dxgkrnl` 必要时构建 paging buffer，patch 物理地址，再 `DxgkDdiSubmitCommand`。每个提交的 DMA buffer 带 fence identifier，GPU 完成后中断，KMD 读回 fence 并通知 `Dxgkrnl`。[WDDM Operation Flow](https://learn.microsoft.com/en-us/windows-hardware/drivers/display/windows-vista-and-later-display-driver-model-operation-flow)

**用户态队列 / doorbell 路径**：

- Linux amdgpu：UMD 写 ring 包并更新 `wptr`，固件消费；内核文档明确这与 KFD 的用户态队列思想类似。[userq](https://www.kernel.org/doc/html/latest/gpu/amdgpu/userq.html)
- Mesa 25.0 为 RadeonSI 合并了 AMDGPU user queue 支持：在受支持的内核上通过 `AMD_USERQ=1` 启用，使作业提交不必经过 AMDGPU command submission ioctl。原文站点此次抓取返回 403，内容已由 Internet Archive 存档页核验。[Larabel, Phoronix, 2024-12-03](https://www.phoronix.com/news/Mesa-25.0-AMDGPU-User-Queue)（[存档](https://web.archive.org/web/20241204000000/https://www.phoronix.com/news/Mesa-25.0-AMDGPU-User-Queue)）
- Intel compute-runtime（NEO）架构说明里有独立的 `direct_submission/`：**绕过「每次提交都进内核」的 ring-buffer 提交路径**。[ARCHITECTURE.md](https://github.com/intel/compute-runtime/blob/master/ARCHITECTURE.md)
- NVIDIA：上述 USERD + doorbell；Yan 等人还指出，用户态虽能直接写 doorbell，但仍须由内核驱动建立到 doorbell 物理 I/O 地址的映射。[arXiv:2604.26889](https://arxiv.org/abs/2604.26889)

LPC 2026 上 Daniel Almeida（Collabora）与 Philipp Stanner（Red Hat）的幻灯片从内核调度器角度补了一句现实：许多 GPU 已不再是「有限条硬件 ring」，而是任意数量的 **firmware ring**；内核「scheduler」往往只做依赖与生命周期跟踪，真正跑什么由固件决定，内核几乎失去任务优先级的控制。[《GPU Job Scheduling: Current Development》幻灯片](https://lpc.events/event/20/contributions/2400/attachments/2138/4732/LPC2026%20Gaming%20on%20Linux%20JobQueue.pdf)

对算子栈的含义是：优化「提交」不再只是少几次 `ioctl`，而是要问——这条 launch 有没有变成更短的命令足迹、有没有避开主机往返、有没有把依赖留给固件/GPU 自己串起来（CUDA Graphs、Device-Generated Commands、GPU 自提交都属于同一方向）。

---

## 4. 内存管理：显存、VA、SVM / HMM 与虚拟内存 API

### 4.1 物理显存在谁手里，虚拟地址在谁手里

RADV 文档把 **video memory 与 GTT** 归给 KMD；UMD 决定需要什么样的 buffer，再通过 DRM uAPI 创建与映射。[RADV](https://docs.mesa3d.org/drivers/radv.html)

Windows WDDM 更加制度化：UMD 创建 resource 时调用 runtime 的 `pfnAllocateCb`，最终进到 KMD 的 `DxgkDdiCreateAllocation`；驻留与换入换出由 `Dxgkrnl` 的 VidMm 和 paging DMA 处理。[WDDM Architecture](https://learn.microsoft.com/en-us/windows-hardware/drivers/display/windows-vista-and-later-display-driver-model-architecture)；[Operation Flow](https://learn.microsoft.com/en-us/windows-hardware/drivers/display/windows-vista-and-later-display-driver-model-operation-flow)

Linux DRM 则提供 GEM 与 TTM 两套内核内存管理基础设施。TTM 面向带专用显存的加速器，管理 buffer object 的生命周期、迁移和 CPU mapping。[DRM memory management](https://www.kernel.org/doc/html/latest/gpu/drm-mm.html)

Intel NEO 把用户态内存子系统写得很明确：`MemoryManager`、`GraphicsAllocation`、GPU 地址空间划分、host pointer 跟踪，并经 `gmm_helper/` 对接 GmmLib（tiling、cache、页表）。[ARCHITECTURE.md](https://github.com/intel/compute-runtime/blob/master/ARCHITECTURE.md)

### 4.2 CUDA 的低级虚拟内存管理（VMM）

CUDA 10.2 起，Driver API 提供一套把「物理句柄」和「虚拟地址」拆开的接口。Cory Perry 与 Nikolay Sakharnykh 的开发者博客（2020-04-15，页面 JSON-LD 标注 `datePublished` 为 2020-04-15）介绍了四件套：

- `cuMemCreate`：创建物理内存句柄
- `cuMemAddressReserve`：保留一段 VA
- `cuMemMap`：把物理句柄映射到 VA
- `cuMemSetAccess`：设置各设备的访问权

这使应用可以做可扩展的 GPU buffer、避免 `cudaMalloc` + `memcpy` 式的再分配，并能把可共享句柄交给 Vulkan / OpenGL 等其他用户态驱动。[Introducing Low-Level GPU Virtual Memory Management](https://developer.nvidia.com/blog/introducing-low-level-gpu-virtual-memory-management/)；编程指南专章：[Virtual Memory Management](https://docs.nvidia.com/cuda/cuda-programming-guide/04-special-topics/virtual-memory-management.html)

与此配套的是流序分配器 `cudaMallocAsync`（CUDA 11.4+）。Vivek Kini 等在 2021-07-27 的博客中讨论如何在流的生命周期内分配与释放，以降低分配器与同步的互相牵制。[Using the NVIDIA CUDA Stream-Ordered Memory Allocator, Part 1](https://developer.nvidia.com/blog/using-cuda-stream-ordered-memory-allocator-part-1/)（`datePublished`: 2021-07-27）

这些接口正是框架分配器「长在 UMD 之上」的钩子：框架不必修改 KMD，只要调用 Driver API，就能实现自己的 VA 策略。

### 4.3 HMM / SVM：共享虚拟地址

Linux 内核的 **Heterogeneous Memory Management（HMM）** 官方文档开宗明义：提供基础设施，把非常规内存（如 GPU 板载显存）纳入常规内核路径，核心是设备内存也可以有 `struct page`，从而让 CPU 与设备共享虚拟地址空间、并支持设备内存的迁移。[HMM](https://www.kernel.org/doc/html/latest/mm/hmm.html)

ROCm 用户态 thunk（`libhsakmt`）面向 KFD，导出包括 `hsaKmtAllocMemory`、`hsaKmtMapMemoryToGPU`、`hsaKmtSVMSetAttr` / `hsaKmtSVMGetAttr` 以及 doorbell 初始化等接口；README 说明它是与 ROCk 内核驱动交互的用户态 API，仓库现已并入 `ROCR-Runtime`。[ROCT-Thunk-Interface README](https://github.com/ROCm/ROCT-Thunk-Interface)；[ROCR-Runtime](https://github.com/ROCm/ROCR-Runtime)

需要避免的过度推断：HMM 是内核基础设施，HSA/SVM、CUDA Unified Memory、IOMMU ATS 等相关但不自动等同。公开邮件列表里长期有人提问它们的差别，本身说明术语容易混用。本文只在资料直接支持的范围内使用这些词。

### 4.4 和 AI 框架分配器的交接

PyTorch 官方 CUDA 笔记（docs 2.15）说明：缓存分配器由 `PYTORCH_ALLOC_CONF`（别名 `PYTORCH_CUDA_ALLOC_CONF`）控制；`backend` 可选 `native` 或 `cudaMallocAsync`（需 CUDA 11.4+）。`expandable_segments`（文档标注 experimental，默认 False）让分配器先创建一段、之后再扩展，以应对 batch size 轻微变化时「老段末尾留下无法回收的碎片、最终 `cudaMalloc` 失败」的问题——它试图（按 stream）维护一段按需增长的大 segment，而不是为每次分配单独开段。[PyTorch CUDA semantics：Memory management](https://docs.pytorch.org/docs/2.15/notes/cuda.html)

同一页还警告：对能装进 GPU 显存的深度学习负载，显式放置通常优于 CUDA Unified Virtual Memory，因为没有缺页；显存饱和时 UVM 会在 CPU 与 GPU 之间做昂贵的双向搬运。这是框架层对「UMD/运行时提供的自动共享 VA」的实践判断，而不是 HMM 实现细节。

**未在该选项的官方说明段落中写明的事**：本文不把 `expandable_segments` 直接断言为「内部调用了 `cuMemMap`」。同页稍后出现的 `cuMemMap` 示例属于多 GPU 虚拟内存代码，不宜自动等同。

---

## 5. 同步：fence、semaphore 与 syncobj

GPU 是大规模异步设备，命令完成可以乱序。Tyr 文把「编码 job 依赖、并由内核保证这些依赖被遵守」列为 KMD 的关键责任。[Almeida, 2025](https://www.collabora.com/news-and-blog/blog/2025/08/06/writing-a-rust-gpu-kernel-driver-a-brief-introduction-on-how-gpu-drivers-work/)

在 Linux DRM 中，用户态显式同步的核心抽象是 **syncobj**：

> DRM synchronisation objects（`drm_syncobj`）为同步原语提供容器，供用户态显式同步 GPU 命令，可在进程之间、以及不同 DRM 驱动之间共享。其首要用例是实现 Vulkan 的 fence 与 semaphore。

用户态 ioctl 支持创建/销毁、以 fd 导入导出、与 sync file 互换底层 fence、reset、signal，以及等待 fence 出现并被通知；timeline syncobj 则建立在 `dma_fence_chain` 上，可对时间线上的 point 做 signal / wait / 导入导出。syncobj 本质上是对可能为空的 `dma_fence *` 的包装。[Kernel doc: DRM Sync Objects](https://www.kernel.org/doc/html/latest/gpu/drm-mm.html#drm-sync-objects)

这解释了 Vulkan UMD 的典型做法：`VkFence` / `VkSemaphore`（尤其是 timeline semaphore）在 Linux 上往往架在 syncobj 之上；跨进程、跨驱动（例如图形队列与视频引擎、或计算进程与显示合成）也走同一套导入导出。

Windows 传统 WDDM 路径则使用提交时写入 DMA buffer 的 **fence identifier**：GPU 完成后发中断，KMD 在 `DxgkDdiInterruptRoutine` 中读回该编号，再 `DxgkCbNotifyInterrupt` 通知 `Dxgkrnl`。[WDDM Operation Flow](https://learn.microsoft.com/en-us/windows-hardware/drivers/display/windows-vista-and-later-display-driver-model-operation-flow)

用户态队列并不能「随便写一个完成标志」。amdgpu userq 文档描述一类由固件执行的包：向**仅特权客户端可写**的内存位置写入信号值，用户队列只有读权限；内核负责准备这块特权内存，包的提交本身在用户态完成。这样，其他 kernel/user queue 才能安全看见完成。[User Mode Queues](https://www.kernel.org/doc/html/latest/gpu/amdgpu/userq.html)

对框架的启示：`cudaEventSynchronize`、`torch.cuda.synchronize()`、Vulkan 的 queue 间依赖、以及 CUDA Graphs 内部的边，最终都会变成 UMD 写入的等待/信号包，或对 syncobj / 事件对象的等待。多余的主机侧同步会强迫 UMD/KMD 走完完整的完成通知路径，把异步流水线打成串行。

---

## 6. UMD 中的 shader / 编译器

「用户态驱动」里往往藏着完整的编译器，而不是只做 ioctl 封装。

### 6.1 Mesa：NIR + 后端，复杂度在「资源怎么接到 shader」

Faith Ekstrand 在 2024-01-16 的 Collabora 文章中指出，今日写 Mesa 驱动的一个高阶难题是：如何在 Vulkan 与 Gallium（OpenGL 等）之间共享代码。[How to share code between Vulkan and Gallium](https://www.collabora.com/news-and-blog/blog/2024/01/16/how-to-share-code-between-vulkan-and-gallium/)

XDC 2022 演讲 *How to write a Vulkan driver in 2022*（演讲者当时署名 Jason Ekstrand；后续公开资料多用 Faith Ekstrand）把「2022 年写一个 Vulkan 驱动」拆成 Mesa 运行时、NIR、winsys 与硬件相关后端等可复用层。原 Collabora 博文（LWN 于 2022-03-24 报道）目前原 URL 会重定向到博客首页，故本文以 [XDC 演讲页与幻灯片](https://indico.freedesktop.org/event/2/contributions/57/) 和 [YouTube 录像](https://www.youtube.com/watch?v=v0uUEePI1IA) 为准，并以 [LWN 报道](https://lwn.net/Articles/889176/) 佐证博文曾于 2022 年 3 月发布。

XDC 2025 的 *Descriptors are Hard*（Faith Ekstrand）把编译器之外、却仍属于 UMD 核心的问题单独拎出来：**访问「住在 shader 之外的资源」**。演讲摘要写：shader 编译大体是已被理解的问题，但 shader 与 API 其余部分的接口仍困难；OpenGL、Vulkan、D3D12 的资源映射策略差异很大。[YouTube，上传时间 2025-11-10](https://www.youtube.com/watch?v=TpwjJdkg2RE)

ANV 文档用大量篇幅描述 bindless / descriptor set 如何变成硬件 binding table，以及 shader 如何先读 descriptor 再发访存消息——这正是「描述符模型」在 UMD 里落地的实例。[ANV](https://docs.mesa3d.org/drivers/anv.html)

### 6.2 ACO：放在 Mesa 里的 AMD 后端

ACO（AMD compiler）是面向 GCN/RDNA 的后端，基于 NIR，把 shader 从 NIR 译成 GPU 可执行的二进制。其 README 给出的动机有三条：优先编译速度（减少游戏卡顿）、做好 divergence 分析以优化运行时、以及缺陷修复可以跟 Mesa 发版走、不必绑在其他项目的时间表上。ACO 维护逻辑 CFG 与线性 CFG（后者按 Whole-Function Vectorization 的思路表示硬件上的实际执行），指令选择以 divergence 分析为中心，以便更好使用标量单元、标量访存和标量寄存器文件。本文使用的 README 文本来自 AOSP 镜像的 Mesa 源码副本（gitlab.freedesktop.org 原文此次被反爬拦截）。[ACO README（AOSP mesa3d 镜像）](https://android.googlesource.com/platform/external/mesa3d/+/refs/heads/main/src/amd/compiler/README.md)

相关讲座：

- [XDC 2020 | A year of ACO: from prototype to default](https://www.youtube.com/watch?v=TFYNUNt3WeE)（X.Org Foundation 频道，oembed 标题已核验）
- [XDC 2021 | Workshop: SSA-based Register Allocation | Connor Abbott and Daniel Schürmann, Valve](https://www.youtube.com/watch?v=UqcbNyN4ix4)

对 AI 编译器读者：ACO 关注的「编译延迟 vs 代码质量」与 JIT 出 kernel（Triton、框架动态 shape、shader 特化）是同一类压力；而 descriptor / binding 则对应算子里的 tensor 参数、workspace 指针、scale/zero-point 如何变成硬件可寻址的资源表。

### 6.3 CUDA 与 Intel：模块加载 vs IGC

CUDA Driver API 把「加载计算模块、取入口、启动」暴露为 `cuModuleLoad*` / `cuLaunchKernel` 等；Runtime 隐藏了 context 与模块生命周期，Driver API 则允许更早或更细的控制，也包括惰性加载等策略。[CUDA Driver API 章](https://docs.nvidia.com/cuda/cuda-programming-guide/03-advanced/driver-api.html)；模块管理参考手册：[Module Management](https://docs.nvidia.com/cuda/cuda-driver-api/group__CUDA__MODULE.html)

Intel compute-runtime 通过 `compiler_interface/` 对接 **Intel Graphics Compiler（IGC）**，并有 builtin kernel（copy/fill）和用于调试的 System Routine（SIP）；程序二进制格式包括 zebin 等。[ARCHITECTURE.md](https://github.com/intel/compute-runtime/blob/master/ARCHITECTURE.md)

摩尔线程公开的 MUSA Driver API 指南采取了与 CUDA 类似的分层：Driver API 提供显式上下文、模块加载与更细的设备控制，面向多 GPU、多进程、库开发和需要精细控制的场景；并与 Runtime API 对照。[MUSA Driver API 开发者指南](https://docs.mthreads.com/musa-sdk/version-5.1.0/programming_guide/api_guides/driver_api_guide)

---

## 7. 各厂商实现与开源项目经验

### 7.1 Mesa：可阅读的 UMD 主战场

Mesa 是 Linux 上 OpenGL / Vulkan / OpenCL 等用户态实现的集合。FOSDEM 2026 演讲 *Mesa3D: the heart of the linux graphics stack*（Juan A. Suarez，2026-02-01，Graphics 开发室）专门面向「有图形基础、但没写过 GPU 驱动」的听众，讲 Mesa 如何把 Vulkan/OpenGL 调用变成硬件指令、shader 编译流程，以及一次 draw 背后发生了什么。官方日程页提供录像与幻灯片。[FOSDEM 2026 日程](https://fosdem.org/2026/schedule/event/HX9XAY-mesa3d_the_heart_of_the_linux_graphics_stack/)

常见开源 UMD 与对应 KMD（据 Mesa 项目页与各驱动文档）：

| UMD | API | 典型 KMD / 备注 |
| --- | --- | --- |
| RADV | Vulkan | `amdgpu`；Steam Deck 默认 Vulkan 驱动之一 |
| RadeonSI | OpenGL（及部分 OpenCL 路径） | `amdgpu`；与 RADV 共享大量 AMD 公共代码 |
| ANV | Vulkan | Intel 官方 Linux Vulkan 驱动；新硬件需 GuC firmware |
| Iris / Crocus | OpenGL | Iris 面向 Gen8+；Crocus 面向更老的 Gen |
| NVK | Vulkan | NVIDIA 开源 Vulkan；与 nouveau / 新内核驱动协同演进 |
| panvk 等 | Vulkan | Mali 等；Tyr 文以 panvk 作为 UMD、Tyr 作为 Rust KMD 的例子 |

**RADV 工作流经验**：合并请求需熟悉代码者审批、首审后通常等 24 小时（周末除外）；文档明确 LLM 辅助 MR 需要提交者真正理解代码，维护者可酌情关闭；不鼓励 LLM 代写的 issue。[RADV 文档](https://docs.mesa3d.org/drivers/radv.html)

**NVK 项目经验**：Faith Ekstrand 于 2023-08-04 宣布 NVK 合入 Mesa 主线，并作为 23.3 的实验驱动发布。[NVK Has landed!](https://www.collabora.com/news-and-blog/news-and-events/nvk-has-landed.html) 之后的进度见 XDC 2023（与 Karol Herbst 的 Nouveau/NVK 更新）和 [XDC 2024 NVK Update](https://www.youtube.com/watch?v=JvtxlwedXzE)。对想读「闭源 NVIDIA UMD 在做什么」的人，NVK 是目前最完整的开源对照物之一；`libmc` 也写明其 QMD 构造参考了 NVK。

调试开关（属于可动手的项目经验，而非八卦）：Mesa 文档列出 `RADV_DEBUG` 的 `dumpibs`（转储 IB）、`bo_history` 等，以及 RadeonSI 侧与 userqueue 相关的 `userqjoblog`、`userqnoshadowregs` 等。[Mesa Environment Variables](https://docs.mesa3d.org/envvars.html)

### 7.2 NVIDIA：闭源 UMD + 开源 KMD 的缝

CUDA Driver API 的实现库随驱动安装（`libcuda`）。开源的是内核模块 [NVIDIA/open-gpu-kernel-modules](https://github.com/NVIDIA/open-gpu-kernel-modules)，它暴露了通道、映射、doorbell 等「搭建」用的 ABI，但**真正把 `cudaMemcpy` / launch 变成 pushbuffer 并按铃的代码仍在闭源用户态**。

Yan、Karlin、Grant（2026-04-29 提交，arXiv:2604.26889）的方法是：利用开源内核模块，在映射路径上插桩，并在用户态映射的 doorbell 上设硬件 watchpoint，从而在提交提交瞬间抓住完整、未撕碎的命令流。两个案例研究值得算子/框架读者直接看：

1. **数据搬运**：识别驱动选择的 DMA 提交模式，并用「绕过 CUDA、受控下发命令」的方式，把硬件原始带宽与驱动开销分开。
2. **CUDA Graphs**：较新 CUDA 版本里 launch 开销下降，与**更小的命令足迹和更高效的提交模式**相关。

[arXiv:2604.26889](https://arxiv.org/abs/2604.26889) · [HTML 全文](https://arxiv.org/html/2604.26889v1)

Maxime Peim 的公开研究仓库与系列文章沿着同一条缝往下做：用 ftrace / LD_PRELOAD / 内核插桩观察 `libcuda`，并指出一个实践细节——`libcuda` 可能用原始 syscall 打开并 mmap 设备文件，从而绕过 glibc 包装，普通 hook 会漏掉关键映射。Medium 原文此次抓取返回 **403**，标题与部分技术点已由搜索摘要及项目 README 交叉核验，阅读时建议同时打开 GitHub。[libcuda-research](https://github.com/maxime-peim/libcuda-research)；系列例如 [Part 3](https://medium.com/ai-infrastructure/part-3-watching-libcuda-without-its-source-245df57c9307)、[Part 4](https://medium.com/ai-infrastructure/part-4-catching-the-doorbell-before-the-gpu-sees-it-aaa5060aadc8)、[Part 7](https://medium.com/ai-infrastructure/part-7-the-sm-authors-the-submission-f2f642befabf)（403 时可用研究仓库对照）。

### 7.3 AMD ROCm：图形 UMD 与计算 thunk 是两条线

计算栈上，ROCr（HSA runtime）经 **libhsakmt（thunk）** 与内核 KFD/ROCk 通信：打开 KFD、查询拓扑、分配/映射内存、创建队列与事件、设置 SVM 属性、初始化 process doorbells。Thunk README 强调它不是独立产品，必须搭配正确的 ROCk 或兼容上游内核，并且从 1.7 起使用 DRM render 节点、用户需加入 `video` / `render` 组。[ROCT README](https://github.com/ROCm/ROCT-Thunk-Interface)；API 文档见 [ROCR-Runtime docs](https://rocm.docs.amd.com/projects/ROCR-Runtime/en/latest/)

图形栈上的 RADV/RadeonSI 走 `amdgpu` DRM uAPI，与 KFD 用户态队列在「用户态 ring + doorbell」思想上汇合，但代码路径和包格式（图形 PM4 vs 计算 AQL/PM4）并不自动相同。读 ROCm 时不要把 `libhsakmt` 误认为 Mesa 的 winsys，也不要把 RADV 误认为 HIP runtime。

### 7.4 Intel：ANV（图形）与 compute-runtime / Level Zero（计算）

ANV 是 Intel 在 Mesa 里的官方 Linux Vulkan 驱动。文档特别指出：从第 12 代 / Alder Lake-P 和 Arc Alchemist 起，3D 驱动栈需要 **GuC firmware** 才能正常工作。[ANV](https://docs.mesa3d.org/drivers/anv.html)

计算侧，[intel/compute-runtime](https://github.com/intel/compute-runtime)（NEO）同时实现 oneAPI Level Zero 与 OpenCL。`ARCHITECTURE.md` 给出进程级对象树：`ExecutionEnvironment` → 共享的 `MemoryManager`、`DirectSubmissionController`、按设备的 `RootDeviceEnvironment`（含 `HardwareInfo` 与 Linux DRM / Windows WDDM 的 `OSInterface`）。命令侧有 `CommandContainer` 与 `Encode*` 编码器；提交侧有可绕过每趟内核调用的 `direct_submission/`。

Level Zero 规范把自己定位为更贴近金属的 offload 接口，覆盖设备抽象、内存、命令表、同步与内核提交。[Level Zero Core Programming Guide](https://oneapi-src.github.io/level-zero-spec/level-zero/latest/core/PROG.html)

### 7.5 Windows WDDM：把 UMD 写成操作系统合同

即便主要工作在 Linux 上，WDDM 仍值得读，因为它把 UMD/KMD/OS 的调用顺序写成了可引用的合同：UMD 由 D3D runtime 加载；`Dxgkrnl` 含显示端口、VidMm、VidSch；厂商必须同时提供 UMD 与 KMD。[Architecture](https://learn.microsoft.com/en-us/windows-hardware/drivers/display/windows-vista-and-later-display-driver-model-architecture)（页面标注最后更新 2025-11-07）

用户态提交、以及「UMD 与 KMD 共享 allocation backing store」（WDDM 3.1 起）等扩展，说明 Windows 同样在把热点路径推回用户态，同时保留内核的安全与兼容模型。[Sharing the Backing Store with KMD](https://learn.microsoft.com/en-us/windows-hardware/drivers/display/sharing-backing-store-with-kmd)

### 7.6 国产 GPU：公开到「编程接口」，很少公开到「UMD 内部」

在已核验的公开材料里，**摩尔线程官方文档**把 MUSA 明确分成 Runtime API 与 Driver API，Driver API 负责显式上下文与模块加载，定位接近 CUDA Driver API。[MUSA Driver API 指南](https://docs.mthreads.com/musa-sdk/version-5.1.0/programming_guide/api_guides/driver_api_guide)；Host/Device 编程模型文档则讨论内存、传输、流与事件，仍是编程模型层而非 UMD 源码级描述。[Host/Device 编程模型](https://docs.mthreads.com/musa-sdk/version-5.1.0/programming_guide/programming_model/host_device_model)

海光 DCU、沐曦 MXMACA、壁仞 SUPA 等在公开网络上有产品与框架适配报道，但**缺少与本文其它章节同等级的、可核验的 UMD 内部实现文档或开源树**（命令编码、doorbell、winsys、用户态编译器结构等）。本文不根据二手商业评论推断其 UMD 架构。若后续有官方源码或驱动开发者文档公开，应再单列补遗。

---

## 8. 调试、性能与 AI 推理 / 算子栈的关联

### 8.1 调试与可观察性

- **RADV / Mesa**：`RADV_DEBUG=dumpibs` 转储 command stream；`bo_history` 记录 BO 操作。这是「看 UMD 到底提交了什么」的第一工具。[envvars](https://docs.mesa3d.org/envvars.html)
- **UMR（User Mode Register Debugger）**：AMD 开源的 Linux 工具，用于检查 AMDGPU 设备状态：寄存器读写、ring 解码、虚拟内存解码与内存读写、寄存器活动跟踪、IP block 监控。由 AMD all-open Linux 团队维护，只使用已公开的 IP 寄存器定义。[UMR 文档导言](https://umr.readthedocs.io/en/main/intro.html)；代码：[gitlab.freedesktop.org/tomstdenis/umr](https://gitlab.freedesktop.org/tomstdenis/umr)
- **NVIDIA 闭源路径**：官方提供 Nsight 等（本文不展开产品手册）；研究侧则用 doorbell watchpoint / 内核插桩捕获完整命令流。[arXiv:2604.26889](https://arxiv.org/abs/2604.26889)；`libcuda-research`
- **RenderDoc / 厂商 GPU profiler** 更偏向图形帧调试与计数器，对「提交路径」的可见性通常不如 IB dump 或 doorbell 捕获来得直接。未在本文逐项核验其与 UMD 内部结构的对应关系，故不写成依据。

### 8.2 对 AI 推理和算子性能真正重要的几点

这些结论都锚定在上文已引用的资料，而不是经验口号：

1. **提交开销是一等公民。** 用户态队列、WDDM UM submission、NVIDIA doorbell、NEO direct submission，全部在减少「每次 launch 进内核」的往返。短 kernel、动态 shape、频繁的小拷贝会放大这条路径。
2. **命令足迹可以被测量、也可以被优化。** Yan 等人显示，CUDA Graphs 的 launch 加速与更小的命令足迹、更高效的提交模式相关；这把「图优化」从框架典故落实到了 UMD 发出的字节。
3. **分配器策略活在 UMD 的内存 API 上。** CUDA VMM 与 `cudaMallocAsync` 是框架缓存分配器的底层；PyTorch 的 `expandable_segments` 明确针对动态 batch 造成的显存碎片。OOM 未必是「显存不够」，也可能是 UMD/框架分配策略留下了无法回收的 sliver。
4. **自动统一内存不是训练/推理的默认最优。** PyTorch 官方笔记对装得下的 DL 负载不推荐 UVM。共享 VA（HMM/SVM/UVM）有真实的系统价值，但缺页和双向迁移的代价要算进端到端。
5. **编译与描述符属于 UMD 热路径。** ACO 把编译速度当作一等目标；Ekstrand 把 descriptor 映射称作现代驱动的主要挑战之一。对 Triton/动态 kernel、以及「每个 shape 一份特化 kernel」的栈，UMD 或用户态编译器的缓存、特化和绑定模型会变成可见延迟。
6. **Vulkan 计算正在成为「绕开专有 UMD API、仍复用成熟 UMD」的路径。** FOSDEM 2025 上 Ruben Ortlam 的 *Expanding GGML Hardware Support using the Vulkan API* 指出：多数机器学习应用走 CUDA/ROCm 这类厂商 API；若改用游戏领域持续投入的 Vulkan 计算 shader，就可以借助更广泛的驱动支持。[FOSDEM 2025 日程](https://archive.fosdem.org/2025/schedule/event/fosdem-2025-6772-expanding-ggml-hardware-support-using-the-vulkan-api/) 这条路并不是「不用 UMD」，而是把依赖从闭源 CUDA UMD 转到 Mesa 等可调试的 Vulkan UMD。
7. **设备生成命令 / GPU 自提交** 把「谁写 command buffer」继续往下推。Igalia 的短视频介绍 Vulkan Device-Generated Commands；amdgpu userq 文档也写明允许 GPU 向自己提交。这对解码器、moe 路由、以及依赖数据的 launch 图有直接含义。[Device-Generated Commands in Vulkan](https://www.youtube.com/watch?v=IvwCF4TGwvs)

### 8.3 虚拟化与多租户（简记）

Rob Clark 在 XDC 2022 的 *moar fps in a vm, with this one little trick* 讨论 VM 中的图形性能；WDDM 的 UM submission 文档则明确提到：在 GPU 半虚拟化场景中，客户机 UMD 直接提交可避免每次向 host 发消息。[YouTube](https://www.youtube.com/watch?v=9sFP_yddLLQ)；[UM work submission](https://learn.microsoft.com/en-us/windows-hardware/drivers/display/user-mode-work-submission) 对云端推理多租户，提交路径上的 guest/host 往返同样是隐藏延迟。

---

## 9. 学习路径与可动手实践

下面按「读得懂、看得到、改得动」排列，均指向上文已列出的公开树或工具，不依赖未公开的内部课件。

**第一阶段：建立正确的分层图（约 1–2 天）**

1. 读 [Tyr 文：UMD, KMD, and the path to hardware](https://www.collabora.com/news-and-blog/blog/2025/08/06/writing-a-rust-gpu-kernel-driver-a-brief-introduction-on-how-gpu-drivers-work/)，只抓住职责边界与「复杂度在 UMD」。
2. 读 [RADV 文档的 Responsibilities / PM4](https://docs.mesa3d.org/drivers/radv.html) 和 [WDDM Operation Flow](https://learn.microsoft.com/en-us/windows-hardware/drivers/display/windows-vista-and-later-display-driver-model-operation-flow)，对照 Linux 与 Windows 的提交序列。
3. 看 [XDC 2022: How to write a Vulkan driver](https://www.youtube.com/watch?v=v0uUEePI1IA)（约 50 分钟）和/或 [FOSDEM 2026: Mesa3D 总览](https://fosdem.org/2026/schedule/event/HX9XAY-mesa3d_the_heart_of_the_linux_graphics_stack/)。

**第二阶段：跟着开源 UMD 走完一次提交（1–2 周）**

4. 在 Mesa 里选一条路径：RADV（`src/amd/vulkan` + winsys）或 ANV 或 NVK。不必先改代码，先画「`vkCmdDispatch` → 发包 → `QueueSubmit` → ioctl/userq」的调用链。
5. 用 `RADV_DEBUG=dumpibs` 跑一个最小 Vulkan compute 或 `vkcube`，对照 PM4/IB 概念。
6. 若有 AMDGPU：装 [UMR](https://umr.readthedocs.io/en/main/intro.html)，对 ring 做一次解码，把「用户态写的包」和「硬件正在执行的包」对上。

**第三阶段：计算栈与框架交接（1–2 周）**

7. 读 [CUDA Driver API](https://docs.nvidia.com/cuda/cuda-programming-guide/03-advanced/driver-api.html) 与 [VMM 博客](https://developer.nvidia.com/blog/introducing-low-level-gpu-virtual-memory-management/)，自己写一个「保留 VA + 映射物理句柄 + 扩展」的小例子，体会它与 `cudaMalloc` 的差别。
8. 读 PyTorch 的 [CUDA memory 笔记](https://docs.pytorch.org/docs/2.15/notes/cuda.html)，用 `PYTORCH_ALLOC_CONF` 比较 `native`、`cudaMallocAsync`、`expandable_segments` 在动态 batch 下的显存曲线（只记录实验，不把结果写成未经测量的通解）。
9. 读 [CUDA Graphs 章](https://docs.nvidia.com/cuda/cuda-programming-guide/04-special-topics/cuda-graphs.html)，再对照 arXiv:2604.26889 的 Graphs 案例：关注「捕获后提交形态变了什么」，而不是只看框架开关。

**第四阶段：深入一条厂商缝（按兴趣）**

10. NVIDIA：浏览 `open-gpu-kernel-modules` 的通道/doorbell 代码；进阶再看 `libcuda-research` / `libmc`（注意其 pinned 的驱动版本，私有 RM/UVM 接口不稳定，不能当生产 ABI）。
11. AMD 计算：从 `libhsakmt` 的队列创建、内存映射、doorbell、SVM 属性往上读到 ROCr。
12. Intel：读 NEO `ARCHITECTURE.md`，对照 Level Zero 命令表与 `direct_submission`。
13. 描述符与编译：看 *Descriptors are Hard*、ACO README，以及 XDC 2020/2021 的 ACO / 寄存器分配讲座。

**实践时建议坚持的纪律**

- 先在自己的机器或明确授权的实验室环境观察提交与内存，不把闭源驱动的逆向结果包装成「官方 ABI」。
- 记录驱动版本：NVIDIA 开源内核、libcuda、Mesa、内核 `amdgpu` 的组合会改变 userq / doorbell 是否可用。
- 性能结论必须带测量语境（是否开启 userq、是否 Graphs、分配器后端），避免把 KMD 调度问题写成 UMD 编码问题，或相反。

---

## 10. 文章与视频索引

下列条目均已核验标题与 URL（视频用 YouTube oembed 或官方日程页）。简评只写与 UMD 学习相关的用途。

### 10.1 架构与职责（必读文章 / 文档）

| 标题 | 出处 | 日期 | 简评 |
| --- | --- | --- | --- |
| [Building Tyr in Rust: UMD, KMD, and the path to hardware](https://www.collabora.com/news-and-blog/blog/2025/08/06/writing-a-rust-gpu-kernel-driver-a-brief-introduction-on-how-gpu-drivers-work/) | Daniel Almeida，Collabora | 2025-08-06 | 当前最清晰的 UMD/KMD 分工短文，以 panvk + Tyr 为例。 |
| [RADV 官方文档](https://docs.mesa3d.org/drivers/radv.html) | Mesa | 随 Mesa 滚动 | 把 UMD 职责、PM4、IB、winsys 写进可引用的官方段落。 |
| [WDDM Architecture](https://learn.microsoft.com/en-us/windows-hardware/drivers/display/windows-vista-and-later-display-driver-model-architecture) | Microsoft Learn | 页标注更新 2025-11-07 | Windows 上 UMD/KMD/`Dxgkrnl`/VidMm/VidSch 合同。 |
| [WDDM Operation Flow](https://learn.microsoft.com/en-us/windows-hardware/drivers/display/windows-vista-and-later-display-driver-model-operation-flow) | Microsoft Learn | （官方流程页） | 从 CreateDevice 到 fence 中断的逐步调用序。 |
| [NEO Architecture Overview](https://github.com/intel/compute-runtime/blob/master/ARCHITECTURE.md) | intel/compute-runtime | 仓库主线 | 计算 UMD 的对象树、编码器、direct submission、IGC。 |
| [Mesa3D: the heart of the linux graphics stack](https://fosdem.org/2026/schedule/event/HX9XAY-mesa3d_the_heart_of_the_linux_graphics_stack/) | Juan A. Suarez，FOSDEM 2026 | 2026-02-01 | 总览型讲座，官方页提供录像。 |

### 10.2 命令提交与用户态队列

| 标题 | 出处 | 日期 | 简评 |
| --- | --- | --- | --- |
| [User Mode Queues](https://www.kernel.org/doc/html/latest/gpu/amdgpu/userq.html) | Linux 内核文档 | 跟随内核（此次抓取为 7.4.0-rc3 文档） | 用户态 ring、`rptr`/`wptr`、绕过提交 ioctl、GPU 自提交。 |
| [User-mode work submission](https://learn.microsoft.com/en-us/windows-hardware/drivers/display/user-mode-work-submission) | Microsoft Learn | WDDM 3.2 / Win11 24H2 语境 | Windows 的 doorbell 低延迟提交，以及为何仍要保留 KM 路径。 |
| [AMD User Queue Mesa Support Merged…](https://www.phoronix.com/news/Mesa-25.0-AMDGPU-User-Queue) | Michael Larabel，Phoronix | 2024-12-03 | Mesa 25.0 RadeonSI 以 `AMD_USERQ=1` 启用；原站 403，见[存档](https://web.archive.org/web/20241204000000/https://www.phoronix.com/news/Mesa-25.0-AMDGPU-User-Queue)。 |
| [Revealing NVIDIA Closed-Source Driver Command Streams…](https://arxiv.org/abs/2604.26889) | Yuang Yan, Ian Karlin, Ryan Grant | 提交 2026-04-29 | 完整捕获闭源 UMD 命令流；含 memcpy 模式与 CUDA Graphs 案例。 |
| [libmc](https://github.com/maxime-peim/libmc) | Maxime Peim | 跟随仓库；README 针对驱动 610.43.02 | 不经 libcuda 的最小提交栈；Hopper 上 SM 自撰写 copy 提交。 |
| [GPU Job Scheduling: Current Development](https://lpc.events/event/20/contributions/2400/attachments/2138/4732/LPC2026%20Gaming%20on%20Linux%20JobQueue.pdf) | Almeida & Stanner，LPC 2026 | 2026（LPC 该届） | 固件 ring 普及后，内核调度器角色的变化。幻灯片已核验；未单独核验是否有官方录像页。 |

### 10.3 内存与同步

| 标题 | 出处 | 日期 | 简评 |
| --- | --- | --- | --- |
| [Introducing Low-Level GPU Virtual Memory Management](https://developer.nvidia.com/blog/introducing-low-level-gpu-virtual-memory-management/) | Cory Perry, Nikolay Sakharnykh，NVIDIA | 2020-04-15 | `cuMemCreate/Map` 四件套与可扩展 buffer。 |
| [CUDA Virtual Memory Management](https://docs.nvidia.com/cuda/cuda-programming-guide/04-special-topics/virtual-memory-management.html) | CUDA Programming Guide | 随 CUDA 滚动 | 官方接口规范。 |
| [Using the NVIDIA CUDA Stream-Ordered Memory Allocator, Part 1](https://developer.nvidia.com/blog/using-cuda-stream-ordered-memory-allocator-part-1/) | Vivek Kini 等，NVIDIA | 2021-07-27 | `cudaMallocAsync` 与流序生命周期。 |
| [HMM](https://www.kernel.org/doc/html/latest/mm/hmm.html) | Linux 内核文档 | 滚动 | 设备内存进入内核页模型、共享 VA 的基础设施。 |
| [DRM Sync Objects](https://www.kernel.org/doc/html/latest/gpu/drm-mm.html#drm-sync-objects) | Linux 内核文档 | 滚动 | Vulkan fence/semaphore 在 Linux 上的用户态容器。 |
| [PyTorch CUDA semantics（Memory management）](https://docs.pytorch.org/docs/2.15/notes/cuda.html) | PyTorch 2.15 文档 | 跟随 2.15 | 缓存分配器、`cudaMallocAsync`、`expandable_segments`、对 UVM 的态度。 |

### 10.4 编译器、描述符与写驱动经验

| 标题 | 出处 | 日期 | 简评 |
| --- | --- | --- | --- |
| [How to share code between Vulkan and Gallium](https://www.collabora.com/news-and-blog/blog/2024/01/16/how-to-share-code-between-vulkan-and-gallium/) | Faith Ekstrand，Collabora | 2024-01-16 | Mesa 里 Vulkan/Gallium 代码共享的高阶问题。 |
| [NVK Has landed!](https://www.collabora.com/news-and-blog/news-and-events/nvk-has-landed.html) | Faith Ekstrand，Collabora | 2023-08-04 | NVK 合入 Mesa 主线的宣告与定位。 |
| [ANV 官方文档](https://docs.mesa3d.org/drivers/anv.html) | Mesa | 滚动 | Binding model、动态状态打包、generation shader。 |
| [ACO README](https://android.googlesource.com/platform/external/mesa3d/+/refs/heads/main/src/amd/compiler/README.md) | Mesa / ACO 作者（Valve 等，README 本文） | 跟随源码 | 编译速度、divergence、双 CFG、指令选择。gitlab 原文被反爬，用 AOSP 镜像。 |
| [Ekstrand: How to write a Vulkan driver in 2022](https://lwn.net/Articles/889176/) | LWN 对 Collabora 博文的报道 | 2022-03-24 | 用于佐证博文发布；正文请看 XDC 演讲。 |

### 10.5 调试、ROCm、国产公开接口

| 标题 | 出处 | 日期 | 简评 |
| --- | --- | --- | --- |
| [UMR Introduction](https://umr.readthedocs.io/en/main/intro.html) | AMD all-open Linux 团队 | 页注 © 2025 | ring / VM 解码的正统工具说明。 |
| [Mesa Environment Variables](https://docs.mesa3d.org/envvars.html) | Mesa | 滚动 | `RADV_DEBUG`、`AMD_DEBUG`、userq 相关开关。 |
| [ROCT Thunk README](https://github.com/ROCm/ROCT-Thunk-Interface) | AMD / ROCm | 仓库已标记 deprecated，指向 ROCR-Runtime | libhsakmt 的定位与 KFD/render 节点要求。 |
| [MUSA Driver API 指南](https://docs.mthreads.com/musa-sdk/version-5.1.0/programming_guide/api_guides/driver_api_guide) | 摩尔线程文档中心 | 文档版本 5.1.0 | 国产 GPU 中少见的、把 Driver API 单独写成指南的公开材料。 |
| [CUDA Driver API 章](https://docs.nvidia.com/cuda/cuda-programming-guide/03-advanced/driver-api.html) | NVIDIA | CUDA 13.5（此次抓取） | Runtime vs Driver，以及 VMM 只在 Driver API 暴露。 |
| [CUDA Graphs](https://docs.nvidia.com/cuda/cuda-programming-guide/04-special-topics/cuda-graphs.html) | NVIDIA | 滚动 | 与「减少提交足迹」对照阅读。 |
| [Level Zero Core Programming Guide](https://oneapi-src.github.io/level-zero-spec/level-zero/latest/core/PROG.html) | oneAPI / Level Zero 规范 | 滚动 | 设备、内存、命令、同步的规范层。 |

### 10.6 视频与讲座（已核验）

| 标题 | 讲者 / 频道 | 日期或来源 | 时长或形式 | 简评 |
| --- | --- | --- | --- | --- |
| [XDC 2022 \| How to write a Vulkan driver in 2022](https://www.youtube.com/watch?v=v0uUEePI1IA) | Jason Ekstrand，X.Org Foundation | oembed 标题核验；会议 2022-10-04 | 约 50 分钟（搜索索引为 49:35） | 写 Mesa Vulkan UMD 的主线讲座；配[幻灯片](https://indico.freedesktop.org/event/2/contributions/57/attachments/57/145/2022-10-04%20XDC%202022.pdf)。 |
| [XDC 2025 \| Descriptors are Hard](https://www.youtube.com/watch?v=TpwjJdkg2RE) | Faith Ekstrand，X.Org Foundation | 上传 2025-11-10 | 2979 秒（约 50 分钟） | 资源映射 / descriptor，UMD 里最容易被算子人忽略的部分。 |
| [XDC 2020 \| A year of ACO: from prototype to default](https://www.youtube.com/watch?v=TFYNUNt3WeE) | X.Org Foundation | oembed 已核验 | 搜索索引 43:30 | ACO 从原型到默认后端的项目经验。 |
| [XDC 2021 \| SSA-based Register Allocation](https://www.youtube.com/watch?v=UqcbNyN4ix4) | Connor Abbott, Daniel Schürmann（Valve） | 上传 2021-09-22 | 搜索索引约 2:02:41 | UMD 编译器后端的寄存器分配工作坊。 |
| [XDC 2023 \| Nouveau/NVK update](https://www.youtube.com/watch?v=SwBe7l9tqro) | Faith Ekstrand, Karol Herbst | 上传 2023-12-12 | 搜索索引 28:57 | 开源 NVIDIA 栈的年度进展。 |
| [XDC 2024 \| Nouveau/NVK Update](https://www.youtube.com/watch?v=JvtxlwedXzE) | Faith Ekstrand | 上传 2024-10-30 | 搜索索引 20:07 | 续篇。 |
| [XDC 2022 \| moar fps in a vm…](https://www.youtube.com/watch?v=9sFP_yddLLQ) | Rob Clark | 上传 2022-10-12 | 搜索索引 35:53 | 虚拟化下的提交/显示路径，补多租户视角。 |
| [FOSDEM 2026 \| Mesa3D: the heart of the linux graphics stack](https://fosdem.org/2026/schedule/event/HX9XAY-mesa3d_the_heart_of_the_linux_graphics_stack/) | Juan A. Suarez | 2026-02-01 | 官方提供 AV1/WebM 与 MP4 | 从零理解 Mesa 结构与一次 draw。 |
| [FOSDEM 2025 \| Expanding GGML Hardware Support using the Vulkan API](https://archive.fosdem.org/2025/schedule/event/fosdem-2025-6772-expanding-ggml-hardware-support-using-the-vulkan-api/) | Ruben Ortlam | FOSDEM 2025 | 官方日程（附件/录像以该页为准） | 用 Vulkan UMD 生态支撑机器学习后端的实践。 |
| [Device-Generated Commands in Vulkan](https://www.youtube.com/watch?v=IvwCF4TGwvs) | Igalia | 上传 2025-01-07 | oembed 已核验（搜索索引约 7:02） | 设备侧生成命令，与「谁来写 CB」直接相关。 |

未把 GPU MODE 的通用 CUDA 编程讲座列入：此次检索未发现以「用户态驱动内部」为主题、且已核验的 GPU MODE 场次。其 [lectures 仓库](https://github.com/gpu-mode/lectures) 仍可作为 CUDA 编程补充，但不应当成 UMD 教材。

Medium 上 Maxime Peim 的 libcuda 系列（Part 3/4/7 等）属于文章而非视频；原站 403，见 §7.2。

---

## 11. 完整参考目录

下列与正文引用对应，按主题粗排。访问日期统一为 2026-11-22（Asia/Shanghai），除非条目自行标明发布或提交日期。

### 官方文档与规范

1. Mesa. *RADV*. https://docs.mesa3d.org/drivers/radv.html
2. Mesa. *ANV*. https://docs.mesa3d.org/drivers/anv.html
3. Mesa. *Environment Variables*. https://docs.mesa3d.org/envvars.html
4. Mesa. *Home*. https://www.mesa3d.org/
5. Microsoft Learn. *WDDM architecture*. https://learn.microsoft.com/en-us/windows-hardware/drivers/display/windows-vista-and-later-display-driver-model-architecture （页标 Last updated 2025-11-07）
6. Microsoft Learn. *WDDM Operation Flow*. https://learn.microsoft.com/en-us/windows-hardware/drivers/display/windows-vista-and-later-display-driver-model-operation-flow
7. Microsoft Learn. *User-mode work submission*. https://learn.microsoft.com/en-us/windows-hardware/drivers/display/user-mode-work-submission
8. Microsoft Learn. *Sharing the Backing Store with KMD*. https://learn.microsoft.com/en-us/windows-hardware/drivers/display/sharing-backing-store-with-kmd
9. The Linux Kernel. *User Mode Queues*. https://www.kernel.org/doc/html/latest/gpu/amdgpu/userq.html
10. The Linux Kernel. *Heterogeneous Memory Management (HMM)*. https://www.kernel.org/doc/html/latest/mm/hmm.html
11. The Linux Kernel. *DRM Memory Management*（含 DRM Sync Objects）. https://www.kernel.org/doc/html/latest/gpu/drm-mm.html ；锚点 https://www.kernel.org/doc/html/latest/gpu/drm-mm.html#drm-sync-objects
12. The Linux Kernel. *DRM Userland Interfaces*. https://www.kernel.org/doc/html/latest/gpu/drm-uapi.html （含 `DRM_CAP_SYNCOBJ` / `DRM_CAP_SYNCOBJ_TIMELINE`）
13. NVIDIA. *CUDA Programming Guide*, §3.3 The CUDA Driver API (v13.5). https://docs.nvidia.com/cuda/cuda-programming-guide/03-advanced/driver-api.html
14. NVIDIA. *CUDA Programming Guide*, Virtual Memory Management. https://docs.nvidia.com/cuda/cuda-programming-guide/04-special-topics/virtual-memory-management.html
15. NVIDIA. *CUDA Programming Guide*, CUDA Graphs. https://docs.nvidia.com/cuda/cuda-programming-guide/04-special-topics/cuda-graphs.html
16. NVIDIA. *CUDA Driver API：Module Management*. https://docs.nvidia.com/cuda/cuda-driver-api/group__CUDA__MODULE.html
17. Level Zero Specification. *Core Programming Guide*. https://oneapi-src.github.io/level-zero-spec/level-zero/latest/core/PROG.html
18. ROCm. *ROCR-Runtime documentation*. https://rocm.docs.amd.com/projects/ROCR-Runtime/en/latest/
19. 摩尔线程. *MUSA Driver API 开发者指南*（文档树 version-5.1.0）. https://docs.mthreads.com/musa-sdk/version-5.1.0/programming_guide/api_guides/driver_api_guide
20. 摩尔线程. *Host/Device 编程模型*. https://docs.mthreads.com/musa-sdk/version-5.1.0/programming_guide/programming_model/host_device_model
21. PyTorch. *CUDA semantics*（2.15，含 Memory management）. https://docs.pytorch.org/docs/2.15/notes/cuda.html
22. UMR. *Introduction — UMR: User Mode Register Debugger*. https://umr.readthedocs.io/en/main/intro.html

### 开源项目

23. NVIDIA. *open-gpu-kernel-modules*. https://github.com/NVIDIA/open-gpu-kernel-modules
24. Intel. *compute-runtime*（含 `ARCHITECTURE.md`）. https://github.com/intel/compute-runtime ；https://github.com/intel/compute-runtime/blob/master/ARCHITECTURE.md
25. ROCm. *ROCT-Thunk-Interface*（已 deprecated，指向 ROCR-Runtime）. https://github.com/ROCm/ROCT-Thunk-Interface
26. ROCm. *ROCR-Runtime*. https://github.com/ROCm/ROCR-Runtime
27. Maxime Peim. *libmc*. https://github.com/maxime-peim/libmc
28. Maxime Peim. *libcuda-research*. https://github.com/maxime-peim/libcuda-research
29. AMD / Tom St Denis. *umr*. https://gitlab.freedesktop.org/tomstdenis/umr
30. ACO README（AOSP mesa3d 镜像，因 gitlab.freedesktop.org 反爬）. https://android.googlesource.com/platform/external/mesa3d/+/refs/heads/main/src/amd/compiler/README.md

### 文章与论文

31. Daniel Almeida. *Building Tyr in Rust: UMD, KMD, and the path to hardware*. Collabora, 2025-08-06. https://www.collabora.com/news-and-blog/blog/2025/08/06/writing-a-rust-gpu-kernel-driver-a-brief-introduction-on-how-gpu-drivers-work/
32. Faith Ekstrand. *How to share code between Vulkan and Gallium*. Collabora, 2024-01-16. https://www.collabora.com/news-and-blog/blog/2024/01/16/how-to-share-code-between-vulkan-and-gallium/
33. Faith Ekstrand. *NVK Has landed!*. Collabora, 2023-08-04. https://www.collabora.com/news-and-blog/news-and-events/nvk-has-landed.html
34. LWN.net. *Ekstrand: How to write a Vulkan driver in 2022*. 2022-03-24. https://lwn.net/Articles/889176/
35. Cory Perry, Nikolay Sakharnykh. *Introducing Low-Level GPU Virtual Memory Management*. NVIDIA Technical Blog, 2020-04-15. https://developer.nvidia.com/blog/introducing-low-level-gpu-virtual-memory-management/
36. Vivek Kini 等. *Using the NVIDIA CUDA Stream-Ordered Memory Allocator, Part 1*. NVIDIA Technical Blog, 2021-07-27. https://developer.nvidia.com/blog/using-cuda-stream-ordered-memory-allocator-part-1/
37. Yuang Yan, Ian Karlin, Ryan Grant. *Revealing NVIDIA Closed-Source Driver Command Streams for CPU–GPU Runtime Behavior Insight*. arXiv:2604.26889, submitted 2026-04-29. https://arxiv.org/abs/2604.26889 ；https://arxiv.org/html/2604.26889v1
38. Michael Larabel. *AMD User Queue Mesa Support Merged For Linux - Submitting Work Directly To The GPU*. Phoronix, 2024-12-03. https://www.phoronix.com/news/Mesa-25.0-AMDGPU-User-Queue （抓取 403；存档 https://web.archive.org/web/20241204000000/https://www.phoronix.com/news/Mesa-25.0-AMDGPU-User-Queue ）
39. Maxime Peim. *Part 3. Watching libcuda Without Its Source*. Medium / AI Infrastructure. https://medium.com/ai-infrastructure/part-3-watching-libcuda-without-its-source-245df57c9307 （此次抓取 403）
40. Maxime Peim. *Part 4. Catching the Doorbell Before the GPU Sees It*. https://medium.com/ai-infrastructure/part-4-catching-the-doorbell-before-the-gpu-sees-it-aaa5060aadc8 （403）
41. Maxime Peim. *Part 7. The SM Authors the Submission*. https://medium.com/ai-infrastructure/part-7-the-sm-authors-the-submission-f2f642befabf （403）

### 演讲、幻灯片与录像页

42. Faith / Jason Ekstrand. *How to write a Vulkan driver in 2022*. XDC 2022, 2022-10-04. 日程 https://indico.freedesktop.org/event/2/contributions/57/ ；幻灯片 https://indico.freedesktop.org/event/2/contributions/57/attachments/57/145/2022-10-04%20XDC%202022.pdf ；录像 https://www.youtube.com/watch?v=v0uUEePI1IA
43. Faith Ekstrand. *Descriptors are Hard*. XDC 2025. https://www.youtube.com/watch?v=TpwjJdkg2RE
44. *A year of ACO: from prototype to default*. XDC 2020. https://www.youtube.com/watch?v=TFYNUNt3WeE
45. Connor Abbott, Daniel Schürmann. *SSA-based Register Allocation*. XDC 2021. https://www.youtube.com/watch?v=UqcbNyN4ix4
46. Faith Ekstrand, Karol Herbst. *Nouveau/NVK update*. XDC 2023. https://www.youtube.com/watch?v=SwBe7l9tqro
47. Faith Ekstrand. *Nouveau/NVK Update*. XDC 2024. https://www.youtube.com/watch?v=JvtxlwedXzE
48. Rob Clark. *moar fps in a vm, with this one little trick*. XDC 2022. https://www.youtube.com/watch?v=9sFP_yddLLQ
49. Juan A. Suarez. *Mesa3D: the heart of the linux graphics stack*. FOSDEM 2026, 2026-02-01. https://fosdem.org/2026/schedule/event/HX9XAY-mesa3d_the_heart_of_the_linux_graphics_stack/
50. Ruben Ortlam. *Expanding GGML Hardware Support using the Vulkan API*. FOSDEM 2025. https://archive.fosdem.org/2025/schedule/event/fosdem-2025-6772-expanding-ggml-hardware-support-using-the-vulkan-api/
51. Igalia. *Device-Generated Commands in Vulkan*. 2025-01-07. https://www.youtube.com/watch?v=IvwCF4TGwvs
52. Daniel Almeida, Philipp Stanner. *GPU Job Scheduling: Current Development*. LPC 2026 slides. https://lpc.events/event/20/contributions/2400/attachments/2138/4732/LPC2026%20Gaming%20on%20Linux%20JobQueue.pdf

---

## 12. 资料边界与未完全核验项

- **Collabora 博文 *How to write a Vulkan driver in 2022***：LWN 于 2022-03-24 报道其存在；2026-11-22 访问原推测 URL `https://www.collabora.com/news-and-blog/blog/2022/03/24/how-to-write-a-vulkan-driver-in-2022/` 会重定向到博客首页，Internet Archive 精确快照也未取到。故不把该博文正文当作已阅读来源，改引 XDC 演讲、幻灯片与 LWN。
- **Phoronix 原文 403**，已用 2024-12-04 附近的 Wayback 存档核对标题、作者、日期与要点。
- **Medium（Peim 系列）403**：URL 与标题由检索与仓库交叉确认，正文细节以 GitHub README / 论文为准，不转述未读到的段落为事实。
- **ACO README**：gitlab.freedesktop.org 返回反爬页；改用 AOSP `platform/external/mesa3d` 镜像。内容应与 Mesa 主线高度接近，但不保证与某一 Mesa 发版逐字相同。
- **国产 GPU**：仅收录摩尔线程官方编程指南中可核对的 Driver/Runtime 分层。海光 / 沐曦 / 壁仞等未找到与 RADV 文档或 NEO `ARCHITECTURE.md` 同等级的 UMD 内部公开材料，故不编造架构。
- **未采用的中文博客**：检索中出现若干 2026 年前后的 CSDN「UMD 实战」帖，行文与结构高度模板化，无法核验独立经验，一律不引。
- **GPU MODE**：未列入未核验的「驱动内部」场次，避免把 CUDA 编程讲座误标成 UMD 资料。
- **PyTorch `expandable_segments`**：官方说明了「可扩展 segment 以减少碎片」，但该段落未写实现是否调用 `cuMemMap`；本文不强行挂钩。
- **HMM 与 CUDA Unified Memory**：二者都与共享 VA 有关，但本文不把实现关系写成已被某一文档证明的等价物。
- **LPC 2026**：已核验 PDF 幻灯片与讲者；未单独确认官方录像链接，故不在视频表中冒充「已看录像」。
- **部分讲座时长**：以 YouTube 页面或公开检索元数据为准，在表中标明「搜索索引」的数字可能与播放器显示有数秒之差。

---

*本文只做调研与分类总结，不构成对任何闭源接口的再实现说明，也不替代各厂商当前驱动版本的官方文档。*
