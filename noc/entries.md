# AI 芯片 / GPU 片上网络（NoC）条目清单

截至 2026-11-22（Asia/Shanghai）。用于持续追加，不必先改综述页。本清单只收与**片上网络**直接相关的材料；片间 / 机柜级互联（NVLink、NVSwitch、UALink、TPU ICI、NeuronLink、Cerebras SwarmX、Dojo 的 Tile 间与 Z-plane 等）仅在条目里作界限说明，不作为主体。

摘要中的事实与数字均来自已打开的公开源（论文摘要或正文、官方文档、Hot Chips 报道、公司技术文），并标明出处。未核实的数字不写。

- **2019-06** · 论文 · 背景 · Eyeriss v2: A Flexible Accelerator for Emerging Deep Neural Networks on Mobile Devices（IEEE JETCAS / arXiv:1807.07928）
  Chen、Yang、Emer、Sze 提出层次化 mesh（HM-NoC）：簇内 all-to-all、簇间 mesh，可在高带宽单播、高复用广播以及两种组播模式间静态切换；iact / weight / psum 走独立网络。65nm 上稀疏 MobileNet、batch=1 时为 1470.6 inf/s、2560.3 inf/J，相对原版 Eyeriss 跑 MobileNet 快 12.6 倍、能效高 2.5 倍。HM-NoC 合计只占逻辑面积约 2.6%。
  https://arxiv.org/abs/1807.07928

- **2019-09** · 论文 · 背景 · CD-Xbar: A Converge-Diverge Crossbar Network for High-Performance GPUs（IEEE Transactions on Computers）
  指出 mesh 适合 manycore CPU，但 GPU 是 many-to-few-to-many 流量（大量 SM 汇聚到少量 LLC/内存控制器再散开），纯交叉开关又难以在高 SM 数下高频工作。CD-Xbar 用本地交叉开关把 SM 收敛后再用全局交叉开关散到 LLC/MC。文中相对同等对剖带宽的 mesh，面积/功耗分别降 52.5%/48.5%，性能平均高 13.9%，距理想全连接交叉开关 2.9%。
  https://www.computer.org/csdl/journal/tc/2019/09/08673612/1ciKE7LYgr6

- **2019-10** · 论文 · 背景 · Simba: Scaling Deep-Learning Inference with Multi-Chip-Module-Based Architecture（MICRO 2019）
  NVIDIA 等实现并流片 36 个 chiplet 的推理 MCM：层次化互联明确分成同一 chiplet 上的 NoC 与封装上的 NoP。单 chiplet 峰值 4 TOPS，整包最高 128 TOPS、6.1 TOPS/W；ResNet-50 batch=1 为 1988 images/s、延迟 0.50 ms。用三种 tiling 优化缓解 chiplet 间通信，相对基线映射最高加速 16%。是把「片上」和「封装内」分开讨论的经典对照。
  https://research.nvidia.com/publication/2019-10_simba-scaling-deep-learning-inference-multi-chip-module-based-architecture

- **2019-12-07** · 论文 · 背景 · Dissecting the Graphcore IPU Architecture via Microbenchmarking（arXiv:1912.03413）
  Citadel 对初代 IPU（1216 tile、每 tile 256 KiB 本地内存）做互连实测：片上 exchange 在全 tile 随机目的地的 all-to-all 负载下聚合带宽 7.7 TB/s，每 tile 约 6.3 GB/s 且几乎不随并发数下降。同列 tile 点到点延迟约 37–59 ns（59–95 拍）。IPU-Link 是片外（同板单向 55 GB/s，跨板约 28 GB/s），不要算进片上 NoC。编程模型是 BSP，exchange 为通信相位提供硬件 all-to-all。
  https://arxiv.org/abs/1912.03413

- **2020-05** · 论文 · 背景 · Think Fast: A Tensor Streaming Processor (TSP) for Accelerating Deep Learning Workloads（ISCA 2020）
  Groq 把芯片按功能切成存储与向量/矩阵运算条带，用确定性张量流而不是传统包交换 NoC 做片上搬运；指令由编译器静态调度，避免运行时仲裁带来的抖动。这是「编译器当互连控制面」的极端例子，后续 LPU 宣传多由此架构延伸，但公开微观细节仍以这篇为准。
  https://www.computer.org/csdl/proceedings-article/isca/2020/09138986/1lsasZYR61O

- **2022-03-22** · 业界 · GPU · NVIDIA Hopper Architecture In-Depth（NVIDIA 技术博客）
  官方确认 H100 用分区交叉开关结构让 L2 就近缓存与 GPC 直接相连的访存；并引入 thread block cluster 与 distributed shared memory，允许同一 GPC 内多个 SM 的共享内存之间直接 load/store/atomic，而不必绕 L2。L2 容量写为 50 MB，HBM3 带宽超过 3 TB/s。NVLink / NVLink Switch 是片外 scale-up，文中需与片上路径分开读。
  https://developer.nvidia.com/blog/nvidia-hopper-architecture-in-depth/

- **2022-08** · 演讲 · 加速器 · Tesla Dojo Microarchitecture（Hot Chips 34；IEEE Micro 2023 扩展版）
  D1 在 TSMC 7nm、645 mm² 上集成 354 个节点、440 MB SRAM，峰值 362 TFLOPS（FP16/BF16/CFP8 口径，官方摘要），时钟约 2 GHz。片上是 2D mesh；Chips and Cheese 根据演讲整理为每个方向 2×64B 链路。节点本地 SRAM 1.25 MB。5×5 个 D1 组成 Training Tile，ServeTheHome 现场记录边带宽 4.5 TB/s，这已是晶圆级/模块级互连，不是单芯片 NoC。IEEE Micro 正式文本：The Microarchitecture of DOJO, Tesla’s Exa-Scale Computer。
  https://www.computer.org/csdl/magazine/mi/2023/03/10078146/1LH8HyZD3ws

- **2023-05** · 论文 · 加速器 · Cerebras Architecture Deep Dive: First Look Inside the Hardware/Software Co-Design for Deep Learning（IEEE Micro / 源自 Hot Chips 34）
  Sean Lie 介绍晶圆级引擎：细粒度数据流核、分布式 SRAM，以及专为 ML 训练设计的片上与片外互连。官方博客给出 WSE-2 为 2.6 万亿晶体管、85 万核。集群级的 SwarmX / MemoryX 是把片上 fabric 的广播–规约语义延伸到系统，本身不是片上 NoC。
  https://www.computer.org/csdl/magazine/mi/2023/03/10123162/1N3MpzfExos

- **2023-04** · 论文 · 组播 · URMP: using reconfigurable multicast path for NoC-based deep neural network accelerators（The Journal of Supercomputing）
  针对 NoC 型 DNN 加速器中权重/激活的一对多分发，提出可重构组播路径，减少重复单播占用的链路与延迟。属于「把 DNN 规整流映射成可配置组播」这条工程线。
  https://link.springer.com/article/10.1007/s11227-023-05255-7

- **2024-04-10** · 业界 · 加速器 · Our next generation Meta Training and Inference Accelerator（Meta 官方博客）
  下一代 MTIA 仍是 8×8 PE 阵列，官方写明「改进的 NoC 把带宽翻倍，并允许 PE 之间低延迟协同」。片上 SRAM 从 128 MB / 800 GB/s 提到 256 MB / 2.7 TB/s，PE 本地存储 384 KB、1 TB/s/PE，主频 1.35 GHz（上一代 800 MHz）。这是少数由厂商直接承认「改 NoC」的公开材料；路由器微结构未披露。
  https://ai.meta.com/blog/next-generation-meta-training-inference-accelerator-AI-MTIA/

- **2024-05-13** · 论文 · 加速器 · SambaNova SN40L: Scaling the AI Memory Wall with Dataflow and Composition of Experts（arXiv:2405.07518；Hot Chips 36 同期演讲）
  可重构数据流加速器，片上分布式 SRAM + 封装内 HBM + 封装外 DDR 三层存储；RDU 之间另有专用互连做 scale-up。文中 8 路 RDU 相对未融合基线在若干基准上加速 2–13 倍。片上是数据流图上的近邻/流式搬运，RDU 间网络不要写成 NoC。
  https://arxiv.org/abs/2405.07518

- **2024-09-26** · 论文 · 设计思路 · FlooNoC: A 645 Gbps/link 0.15 pJ/B/hop Open-Source NoC…（arXiv:2409.17606；IEEE TVLSI 2025）
  ETH Zurich 面向 AI 加速器的**大块数据搬运**（相对 cache-line 相干流量）开源 NoC：宽 AXI4 链路、多流 DMA 做端到端排序、短消息另走专用物理链路。12nm 上 8×4 mesh、288 个 RISC-V 核，单链路 645 Gbps，整网聚合 103 Tbps，每跳 0.15 pJ/B（0.8 V），计算 tile 面积开销 3.5%。相对传统多层 AXI 互连面积少 30%，同 floorplan 算力高 47%。后续 Collective-Capable NoC 即在此上扩展。
  https://arxiv.org/abs/2409.17606

- **2025-02-04** · 教程 · 设计思路 · How to Think About TPUs（Google DeepMind *How to Scale Your Model*）
  把 TensorCore（MXU 脉动阵列、VPU、VMEM）和 HBM 讲清楚，并明确 ICI、光交换、3D torus 是**芯片之间**的互连。TPU7x 去掉 megacore，改为两个核之间的高带宽片内链路。读 TPU 时不要把 ICI 当成片上 NoC；脉动阵列本身是运算数据通路，也不是包交换网络。
  https://jax-ml.github.io/scaling-book/tpus/

- **2025-01** · 论文 · GPU · Dissecting the NVIDIA Hopper Architecture through Microbenchmarking and Multiple Level Analysis（arXiv:2501.12084）
  用微基准测量 Hopper，专章讨论 distributed shared memory（DSM）等 SM 间通信，以及 Tensor Core / TMA 等。提供了官方博客之外、可复现的片上通信行为数据，适合和「L2 交叉开关 + GPC 内 SM-to-SM」这一分层对照着读。
  https://arxiv.org/abs/2501.12084

- **2025-02-26** · 论文 · 组播 · A Multicast-Capable AXI Crossbar for Many-core Machine Learning Accelerators（arXiv:2502.19215）
  Colagrande 与 Benini：在 AXI 交叉开关上做轻量组播，最大可物理实现的 16×16 交叉开关面积/时序开销约 12% 和 6%；接到开源 288 核加速器后，GEMM 加速 29%。文中也把 Hopper 的 SM-to-SM 网络作为工业界需要组播/近邻交换的旁证。这是后续 mesh 上集合通信 NoC 的前置工作。
  https://arxiv.org/abs/2502.19215

- **2025-04** · 论文 · 仿真 · NoCDAS: A Cycle-Accurate NoC-Based Deep Neural Network Accelerator Simulator（ACM TOMACS 35(4)）
  周期精确的 NoC-DNN 加速器模拟器，针对已有工具在微结构与执行时间上不够准的问题。综述部分回顾了 mesh 上的映射、组播以及把 Eyeriss 一类阵列扩成层次 2D mesh 的做法。适合做设计空间探索，而不是某一款商用芯片的规格来源。
  https://doi.org/10.1145/3729169

- **2025-04** · 论文 · 拓扑 · HT-NoC: Reconfigurable High Throughput Network-on-Chip for AI Dataflow Accelerators（ARC 2025）
  针对全连接层权重分发带宽瓶颈，用可重构路由器连接在 mesh 基线上把延迟降到 1/4、能量降到约 1/2.7；接入 dataflow 加速器后，Transformer 的 FFN 相对所比较的脉动阵列加速器快 3 倍。说明「为权重广播改拓扑」仍是 2025 年的活题目。
  https://doi.org/10.1007/978-3-031-87995-1_2

- **2024** · 文档 · 加速器 · AWS Trainium2 Architecture（AWS Neuron 官方文档）
  每颗 Trainium2 有 8 个 NeuronCore-v3、96 GiB HBM、2.9 TB/s、DMA 3.5 TB/s。官方把 NeuronLink-v3（每芯片 1.28 TB/s）和 16 个 CC-Core 明确写成**芯片之间**的集合通信与扩容。公开文档几乎不展开 8 个 NeuronCore 之间的片上互连微结构；引用时不要把 NeuronLink 写成 NoC。
  https://awsdocs-neuron.readthedocs-hosted.com/en/latest/about-neuron/arch/neuron-hardware/trainium2.html

- **2025-08-30** · 论文 · 功耗 · Bit Transition Reduction by Data Transmission Ordering in NoC-based DNN Accelerator（arXiv:2509.00500）
  在 NoC 型 DNN 加速器上通过传输排序减少位翻转，从而降动态功耗。说明除了拓扑与组播，链路翻转率也已成为 AI NoC 的优化目标。
  https://arxiv.org/abs/2509.00500

- **2026-03-24** · 论文 · 经验分享 · Numerical Kernels on a Spatial Accelerator: A Study of Tenstorrent Wormhole（arXiv:2603.23343，HPEC 2026）
  把稀疏数值核与共轭梯度映射到 Wormhole 的空间计算与 2D torus NoC 上，并与 NVIDIA GPU 对比。重点是：片上显式核间通信取代「经共享缓存/片外存储转弯」，以及把科学计算核塞进为神经网络设计的空间互连时会遇到的映射问题。
  https://arxiv.org/abs/2603.23343

- **2026-03-27** · 论文 · 集合通信 · A Lightweight High-Throughput Collective-Capable NoC for Large-Scale ML Accelerators（arXiv:2603.26438v2，MLSys 2026）
  在 FlooNoC 上加入硬件组播、规约和屏障，并用 Direct Compute Access 让路由器借用 cluster 的 FPU 做网内规约。TSMC 7nm、1 GHz 下全功能路由器面积开销 16.9%，tile 级 <1%。1–32 KiB 上组播/规约几何平均加速 5.3× / 2.8×；SUMMA GEMM 在大 mesh 上最高估得 3.8×，FusedConcatLinear 最高 2.4×，能量最高约 1.17×。开源实现基于 pulp-platform/FlooNoC。本目录另有深度笔记。
  https://arxiv.org/abs/2603.26438

- **2026-09** · 论文 · GPU · Dissecting How Die Scaling Breaks GPU Fine-grained Scheduling（arXiv:2609.24270）
  用微基准揭示 H200/B200 上 L2 被分成与内存亲和的分区，中间由 LTC fabric 相连；远端 HBM 带宽受这块 fabric 限制，而不是 HBM 本身。跨分区加载会把数据复制进本地 L2（H200 再访问约 310 拍、B200 约 360 拍，接近本地 L2 命中）。说明现代 GPU「片上互连」已是多分区、甚至多 die 的 NUMA 结构，和教科书交叉开关不是一回事。
  https://arxiv.org/abs/2609.24270

- **2026-11** · 文档 · 加速器 · Cerebras WSE-3 / WSE-3 Turbo 产品页与 SDK：Wafer-Scale Engine Architecture（官方）
  产品页（本轮抓取，页面同时介绍 WSE-3 Turbo）写明 WSE-3 有 44 GB 片上 SRAM、43.2 PB/s 存储带宽，wafer-scale fabric 内部带宽 53.5 PB/s；WSE-3 Turbo 为 46,225 mm²、4 万亿晶体管、90 万核、250 petaflops。SDK 把机制写死：2D mesh，邻居之间单周期发送 32-bit wavelet，路由走 24 条 color 虚通道，CE 只能经本 PE 的 router（RAMP）与四邻居通信，不能直接读写别人的内存。缺陷通过冗余核与冗余路由绕开。
  https://cerebras.ai/product-chip
  https://sdk.cerebras.ai/computing-with-cerebras

- **2026-11** · 文档 · 加速器 · Tenstorrent Blackhole / Wormhole NoC ISA 文档（官方开源）
  目前公开资料里对商用 AI 芯片 NoC 写得最细的一套：双 2D torus、方向相反；Blackhole 上 flit 为 512 bit、时钟 1.35 GHz（Wormhole 为 256 bit @ 1 GHz），路由器到邻居 每拍 1 flit、延迟 9 拍，NIU–router 约 5 拍。支持矩形广播、posted 写、L1 原子；虚通道由 dateline / class / buddy 三位组成。硬件保证常见事务无死锁，但 VC_LINKED、不预约路径的广播、非 0 仲裁优先级要把部分责任交给软件。本目录另有深度笔记。
  https://github.com/tenstorrent/tt-isa-documentation
  https://raw.githubusercontent.com/tenstorrent/tt-isa-documentation/main/BlackholeA0/NoC/README.md

- **2022-09-01** · 经验分享 · 加速器 · Hot Chips 34 – Tesla’s Dojo Microarchitecture（Chips and Cheese）
  基于 HC34 演讲的独立整理：mesh「每个方向 2×64B 链路」、节点 5 个 64B SRAM 端口、D1 无片上 HBM/PCIe，经 DIP 出系统，die 边延迟约 100 ns。适合与 IEEE Micro 正文对照；数字以演讲原片和正式论文为准，这篇是可读的工程转述。
  https://old.chipsandcheese.com/2022/09/01/hot-chips-34-teslas-dojo-microarchitecture/

- **2022-08-23** · 经验分享 · 加速器 · Tesla Dojo AI Tile Microarchitecture（ServeTheHome 现场）
  HC34 当场记录：354 节点/片、1.25 MB SRAM/节点、2D mesh、Tile 边 4.5 TB/s、不保证端到端排序、目的地靠计数同步、编译器要建同步树、软件绕死节点。把「片上网」和「Tile / Z-plane」分层讲清楚了。
  https://www.servethehome.com/tesla-dojo-ai-system-microarchitecture/

- **2022-08-23** · 经验分享 · 加速器 · Cerebras Wafer Scale Engine WSE-2 and CS-2 at Hot Chips 34（ServeTheHome）
  现场记录 WSE-2 / CS-2 演讲，可与 IEEE Micro 2023 深潜文对照。注意把晶圆上 PE mesh 与后来的 SwarmX 集群织物分开。
  https://www.servethehome.com/cerebras-wafer-scale-engine-wse-2-and-cs-2-at-hot-chips-34/

- **2025-11-18** · 经验分享 · 加速器 · Moreh：Tenstorrent AI Data Center Solution System Architecture（合作方公开 PDF）
  把 Wormhole 写成 10×12 网格（80 Tensix + GDDR/Ethernet/PCIe 等），统一 2D torus NoC；Wormhole 每方向 32 B/cycle，Blackhole 64 B/cycle；两套方向相反的 router。片间 Ethernet 与 Galaxy 机柜 mesh 是另一层。数字需与官方 ISA（时钟、flit 宽度）交叉验证，本文提供了「谁连到 NoC 上」的系统图。
  https://moreh.io/assets/moreh-tenstorrent-ai-data-center-solution-system-architecture-251118.pdf
