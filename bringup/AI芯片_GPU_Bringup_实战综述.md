# AI 芯片 / GPU Bring-up 实战经验与资料综述

> 面向 MLSys、AI 芯片与 GPU、硬件软件协同设计的研究者与工程师。  
> 检索与核验日期：2026-11-22（Asia/Shanghai）。本文只收录作者实际打开或抓取过的公开链接；未核验的 URL 不写入。无法从原文确认的数字、引言一律不编造。

## 本文说明

「Bring-up」在不同语境里含义略有差别，但实践路径高度一致：从**第一次上电**开始，按依赖顺序证明电源、时钟、复位、调试口、固件、驱动、互连、运行时，直到系统能稳定跑真实负载，并把基线和故障证据留下来。

本文优先收录能落到操作步骤、排查清单、常见坑与 debug 手法的资料，而不是产品软文。每条资源给出标题、作者/机构（能查到则写）、日期、类型、URL，以及一段可操作的要点摘要。视频尽量写清其中的具体技巧；若主要依据视频页面说明、讲者口播转写或配套文字，会在摘要中注明信息来源。

**建议阅读方式：** 先读各类开头的「跨资源共识」，再按需要点开资源清单。官方手册（NVIDIA / AMD / Intel Gaudi / Tenstorrent）应作为现场操作的第一依据；工程博客、论坛经验帖和认证辅导视频用于补充「为什么」和「踩过的坑」，其中的阈值、命令与版本需对照当时官方文档复核。

## 分类目录

1. [硅前与 first silicon / post-silicon 验证](#1-硅前与-first-silicon--post-silicon-验证)
2. [板级 / 系统级 bring-up（供电、时钟、复位、接口）](#2-板级--系统级-bring-up供电时钟复位接口)
3. [GPU / AI 加速卡硬件与互连 bring-up（PCIe、NVLink、HBM、热设计、ECC）](#3-gpu--ai-加速卡硬件与互连-bring-uppcienvlinkhbm热设计ecc)
4. [Firmware / 驱动 / 运行时栈 bring-up](#4-firmware--驱动--运行时栈-bring-up)
5. [多卡 / 机架 / 集群上线与排查](#5-多卡--机架--集群上线与排查)
6. [调试工具与常见故障](#6-调试工具与常见故障)
7. [公司 / 开源案例与教程](#7-公司--开源案例与教程)

文末有[资源统计](#资源统计)、[检索缺口与局限](#检索缺口与局限)和[建议的现场最小清单](#建议的现场最小清单跨资源提炼)。

---

## 1. 硅前与 first silicon / post-silicon 验证

### 跨资源共识

自研 AI 加速器 / ASIC 的 bring-up，成败很大程度上在硅片回来之前就已经决定。

- **先排演，再上电。** 在仿真器、FPGA 原型或硬件辅助验证平台上把上电路线、Boot ROM、调试器访问和基本测试完整跑通。ChipVerify 把这段叫做「rehearsing bring-up」。Etched 走得更远：不只跑简化用例，而是在 ZeBu 仿真上直接跑 Llama 8B 推理，同时锻炼 DMA、HBM 通路、数据完整性和软件迭代。
- **第一次上电从最简配置开始。** 关掉电源管理、安全启动和正常 boot 流，先证明电源、时钟、复位和调试口活着。稳定的 power-on recipe 往往要数天到一周；把复杂特性逐项重新打开可能还要数周。
- **顺序不可颠倒：** 上电前对地电阻检查 → 按规格书顺序、带电流限制地上电 → 外时钟 / 旁路 PLL → 释放复位 → 读 JTAG IDCODE。电流远超预硅估计时立刻断电排查，不要继续加压。
- **DFT 必须为 post-silicon 服务。** AI 芯片门数大、重复核多，层次化 DFT、分组 core / tile 签核、包化 scan 网络，比芯片级一把扫更现实。挂死、极低概率故障时，扫描链复用（scandump）往往比只靠软件调试器更有用。
- **软件也要 0-day 就绪。** 引导加载器、安全固件、Linux 或 RTOS 应在仿真/仿真器上先跑通，并尽量上游化，这样硅片到达当天不必与 fab 同地才能开机。
- **不要按「一次硅片成功」排计划。** 复杂芯片通常至少还要一次改版；金属层修复比改基础层更快。量产决策看的是新 bug 趋势是否收敛，而不是未关闭 bug 的绝对数量。

### 资源清单

#### First Power-On: Supplies, Clocks and Reset

- 作者/机构：ChipVerify
- 日期：页面标注 2026-10-28（教程站点，抓取时仍可访问）
- 类型：文章 / 教程
- URL：<https://chipverify.com/post-silicon-validation/first-power-on>
- 要点：把 first power-on 拆成固定顺序。上电前对每路电源测对地电阻，近 0 Ω 视为短路（板、插座或芯片）。每路按规格顺序单独上电，电流限制略高于预硅估计的漏电；超限就断电，不要用电源满量程（文中练习把 60 mA 估计设成 120 mA 限制，而不是 3 A）。时钟先用外部源并旁路内部 PLL，确认后再开 PLL 并检查 lock 与频率。复位释放后若有状态脚或寄存器，这是内部逻辑活着的第一个证据。调试器读到正确的 JTAG IDCODE，说明电源、测试时钟、TAP 和调试管脚都通。调试口无响应时，从轨电流、管脚波形、换芯片/换板来定位；问题跟着芯片走就是芯片，跟着板走就是板。常见错误：不设电流限制、不遵守上电顺序（可能正偏 ESD 二极管甚至 latch-up）、只用一颗芯片和一块板。

#### Post-Silicon Validation（总览）

- 作者/机构：ChipVerify
- 日期：抓取于 2026-11-22，页面为持续维护的教程
- 类型：文章 / 教程
- URL：<https://chipverify.com/verification/post-silicon-validation>
- 要点：把 silicon bring-up 定义为「第一次上电并逐步证明芯片能工作」。步骤依次为：电源与电流、时钟与复位（硅上复位去断言会与 PLL lock、POR、真实时钟沿交互，这是和理想仿真最先分叉的地方）、冒烟测试（经 JTAG 读已知寄存器和芯片 ID，确认 UART/SPI/I2C 有响应）、再进入按验证计划的功能验证。强调硅上没有全信号波形窗口，可见性只来自管脚和调试接口。JTAG / boundary scan 通常是复位之后的第一个调试接口，用来读写寄存器、跑 BIST。

#### The Post-Silicon Timeline: First Silicon to Production Release

- 作者/机构：ChipVerify
- 日期：页面标注 2026-10-28
- 类型：文章 / 教程
- URL：<https://chipverify.com/post-silicon-validation/post-silicon-timeline>
- 要点：给出从 tapeout 到量产决策的时间线。硅片到达前应用来完成板卡、测试内容、调试软件，并在仿真器/FPGA 上预演 bring-up。稳定 power-on recipe 可能数天到一周。步进命名惯例：字母表示基础层，数字表示金属层（A0 为首硅，A1 为仅改金属，B0 为改基础层）。量产决策看 bug 趋势、功耗性能、电学裕量、可靠性认证和量产测试就绪；新 bug 发现率仍在上升时不宜按期放行。常见错误：只计划一次改版、在非量产步进上做最终分档、只看未关闭 bug 数量不看趋势。

#### Advanced DFT And Silicon Bring-Up For AI Chips

- 作者/机构：Lee Harrison / Siemens EDA，刊于 Semiconductor Engineering
- 日期：2023-12-07
- 类型：文章（厂商技术文）
- URL：<https://semiengineering.com/advanced-dft-and-silicon-bring-up-for-ai-chips/>
- 要点：AI 芯片共性是超大规模、大量重复处理核和分布式存储器。DFT 应：利用规则性做层次化签核（core / block / chip），在「单核重复 DFT」和「整片一把扫」之间用 core group / tile 做折中；用包化 scan 数据网络（文中为 Tessent SSN）把测试数据送到各块，IJTAG 负责配置；多裸片封装沿用同一层次化方法并配合 IEEE 1838。Memory BIST 控制器可在多核间共享以省面积。把 DFT 插到 RTL 而不是门级，可避免每次改 DFT 都重新综合。硅上 bring-up 的传统痛点是 DFT 工程师与 ATE 测试工程师来回迭代；若 DFT 工程师能直接做 pattern debug / 表征，测试工程师能独立做从触发器到网表级的诊断，可缩短 first silicon 周期。需注意这是 EDA 厂商视角，工具名是 Tessent 产品线，方法本身可迁移。

#### Understanding Scandump: A Key Silicon Debugging Technique

- 作者/机构：Vincent Yang，刊于 Semiconductor Engineering
- 日期：2024-06-13
- 类型：文章
- URL：<https://semiengineering.com/understanding-scandump-a-key-silicon-debugging-technique/>
- 要点：Scandump 把 DFT 扫描链复用到功能调试，在死锁或系统无响应时抽取扫描链上寄存器/锁存器的状态快照。文中给出合作方案例：一次硅上死锁要 48 小时、200 多颗器件才复现一次；另一次 hang 要一周、80 颗器件。仅靠 CoreSight 一类软件调试往往不够——外部调试器可能停不住核，trace 带宽只能覆盖部分核，PMU 只能提示某类 snoop 无响应，ELA 只能抓住预先埋好的信号。Scandump 提供接近单周期的内部寄存器全景，有时几小时就能定位，再回到仿真复现。局限：对动态过程可能抓不准「出事那一拍」；数据可达数十万到数百万比特，目前仍靠人工分析。建议与 CoreSight、PMU、ELA 组合使用。文中也指向 IEEE P2929（系统级状态提取标准）。

#### A Bumblebee Moment for AI Silicon

- 作者/机构：Johannes Stahl / Synopsys；讲述 Etched 在 Synopsys Converge Silicon Valley 的工作
- 日期：2026-08-28
- 类型：文章（会议报告转写 / 厂商博客）
- URL：<https://www.synopsys.com/blogs/chip-design/bumblebee-moment-ai-silicon.html>
- 要点：对为特定负载定制的 AI ASIC，传统「块级正确 + 简化用例通过」不够，真正的问题是：在流片前能否正确、高效地跑它要跑的负载。Etched 的 Sohu 在 ZeBu 上跑完整 Llama 8B 推理：参数经 DMA 流过 HBM，同时验证存储子系统和数据完整性，并在 32 层 transformer 上处理一百万 token。他们部署 16 台 ZeBu Server 5，外围 48 台服务器（16 台接仿真运行时，32 台做模型编译），并用常驻管理器降低模型重载，把软件迭代从 6 分钟降到 2 分钟，支撑约 40 名软件开发者和每小时 30 次以上验证。采用三层置信度：小范围组件测试服务日常开发，而不是每个测试都上全芯片全负载。关键警告：接口、存储和时序如果建模过于乐观，会得到「仿真里漂亮、硅上不成立」的信心。配套报告标题为 David Svensson（文中也拼作 Sevensson）, Etched AI，《Validating Large Language Model Inference on ZeBu: A Llama 8B Case Study》。这是厂商成功案例文，规模数字来自该文陈述，未独立复现。

#### 10 Tips for First-Pass Silicon Success in AI Chip Development

- 作者/机构：原载 Synopsys，转载于 Edge AI and Vision Alliance
- 日期：2025-11-17
- 类型：文章（方法论，带厂商工具色彩）
- URL：<https://www.edge-ai-vision.com/2025/11/10-tips-for-first-pass-silicon-success-in-ai-chip-development/>
- 要点：可操作的部分是流程而不是工具广告：尽早做架构探索以暴露计算/存储/互连瓶颈；尽量用硅已验证的 IP（高速接口、存储控制器等）；软件团队必须进入预硅规划，并在硅前用硬件模型跑真实软件；RTL 验证要组合仿真、形式化与覆盖率驱动，并用硬件辅助仿真/原型补仿真漏掉的系统级和软件集成问题；流片前用全面的系统级验证、真实接口验证板和可调试的软件环境，为 first silicon 的 bring-up 做准备；测试与生命周期管理（含多裸片）要在首硅到达前就就位。文中「千万亿次验证周期」等表述是厂商口径，不宜当硬指标引用。

#### Getting your open source software ready for 0-day SoC bringup: success stories and strategies

- 作者/机构：Bhupesh Sharma，Open Source Firmware Conference 2024
- 日期：2024-09-03
- 类型：讲座（会议页 + InfoCon 转写归档）
- URL：<https://osfc.io/2024/talks/getting-your-open-source-software-ready-for-0-day-soc-bringup-success-stories-and-strategies/>
- 转写：<https://infocon.org/cons/Open%20Source%20Firmware%20Con/OSFC%202024/OSFC%202024/Getting%20your%20open%20source%20software%20ready%20for%200-day%20SoC%20bringup%20-%20success%20stories%20and%20strategies%20-%20Bhupesh%20Sharma.eng.text>
- 要点：信息来源为会议简介和 InfoCon 英文转写，而非作者自行逐帧观看的唯一版本。讲者强调：SoC 流片成本高，多数芯片需要多次迭代；如果运行在 SoC 上的软件没有「硅片第一天就能跑」的心态，失败概率和总成本会再升高。策略是在硅前用开源软件（U-Boot、ARM Trusted Firmware、Linux 等）在仿真/仿真器上把启动路径跑通，并把平台支持上游化，从而把 0-day bring-up 与 fab 所在地解耦。讲者举了 Freescale 平台例子：硅片回来当天跑通 U-Boot、安全固件、Linux 和一个简单网络用户态程序；并指向 Linux 树中的仿真设备树（如 `fsl-ls2080a-simu.dts`）。实践建议：为引导软件选择合适的仿真/仿真器模型，目标是一直启动到 Linux/Android 用户态。

#### Post Silicon Validation DEMO: Silicon Lifecycle, Platform Validation, Debug Thinking, Industry Tools

- 作者/机构：VLSI FOR ALL（YouTube 频道）
- 日期：2026-06-12
- 类型：视频（约 59 分钟）
- URL：<https://www.youtube.com/watch?v=Lxx2iy7UZik>
- 要点：信息来源为 YouTube 自动转写。这是面向工程师的综述课，不是某一颗 AI 芯片的实验室日记。有用的部分是把产品生命周期和 post-silicon 工作方式串起来：需求与架构（接口、功耗、核数、时钟与存储结构）→ RTL 设计与验证（此时还没有真硅，靠仿真/仿真）→ DFT（为硅后读寄存器和调试预埋 JTAG 等逻辑）→ 物理设计与 tapeout → 硅后 bring-up 与平台验证。调试思维部分强调：异常关机或重启时先看系统日志/事件记录，看故障前发生了什么；系统 hang 时用调试器看卡在何处；用 JTAG 读内部寄存器，并对照规格书/数据手册判断复位后或从低功耗返回后的值是否符合预期。章节标记包括 first silicon bring-up、validation and testing、stress testing、power and performance。适合作为入门地图，具体实验室规程仍应对照 ChipVerify 与官方 DFT/ATE 文档。

#### Post Silicon Bring-up By Sundar Vedantham

- 作者/机构：Sundar Vedantham（口播中为存储/固件方向，谈到 Intel 与 Nokia 侧的器件 bring-up）
- 日期：2025-01-16
- 类型：视频（约 62 分钟）
- URL：<https://www.youtube.com/watch?v=rD-rKmKdZAo>
- 要点：信息来源为 YouTube 转写。这是一次公开上传的、偏内部培训风格的讲座，不是经过精修的公开教程，但 bring-up 节奏写得很具体。讲者按预硅验证、筛选与分档、板级设计与验证、JTAG 等调试工具、再到硅后 bring-up 的顺序讲。实践上强调硅前验证、板级准备和硅后计划必须对齐；客户侧 bring-up 应复用己方已经走通的知识。讲者对比传统 Intel 板级 bring-up 可能要两三个月，而某个（转写中作 Lionfish / Lon fish，拼写不清）案例中 Linux 提示符在己方两三天出现，到 Nokia 侧又用一两天完成，说明「预演充分 + 基本 OS 尽快起来」能显著压缩后续工作。周计划的目标包括数周内做到基本包处理、MAC/以太网交换等（这是存储/网络器件语境，不是 GPU）。结尾提到该场录音计划放到 Intel 内部视频门户。引用时请把具体周次目标当作「一种器件的计划样例」，不要直接套到 AI 加速器上。

---

## 2. 板级 / 系统级 bring-up（供电、时钟、复位、接口）

### 跨资源共识

板级 bring-up 的目标不是「立刻跑应用」，而是**在不毁掉稀缺样片的前提下，证明电源树、时钟、复位和最基本的通信口是干净的**。

- **上电前必做视觉检查和对地电阻。** 同一网络近 0 Ω，电源对地应是高阻（有上拉/分压时可能是 kΩ 级，但数百欧以下通常要怀疑）。探针接触不良（助焊剂、氧化）会造成「假开路」，应反复测。
- **必须使用限流实验室电源，且先关输出再接线。** 电源输出电容里存的能量可能在限流生效前就打坏器件。电流限制略高于估计工作电流（需计入未编程 MCU 把 LED 全拉亮、电容浪涌等）。进入恒流且输出电压明显掉下去，多半是短路。
- **热像仪比手指快且更安全。** 限流下即使温升不到 1°C 也能定位漏电元件，不必等到芯片发烫。
- **电源正常不等于板子正常。** 接着用表确认每一路稳压输出；再用示波器看时钟频率、幅度、振铃与抖动；然后才是 SPI/I2C/UART。I2C 必须有上拉；UART/SPI 最常见的是 TX/RX 接反和波特率不一致。
- **用换板/换片定位，不要凭猜测到处探。** 固定流程：外观 → 欧姆 → 电流 → 电源 → 时钟 → 信号 → 通信。
- **注意 strap 与调试口生命周期。** 某些 SoC 在 Boot ROM 成功加载镜像后会关闭 JTAG/SWD，空片才能挂调试器；成功启动后要由固件重新打开调试口。

### 资源清单

#### Common PCB Bring-Up Problems and How to Diagnose Them

- 作者/机构：Ben H / SolderStack
- 日期：2026-03-28（页面另有 2026-05-21 更新标记）
- 类型：文章
- URL：<https://solderstack.com/pcb-bring-up-problems/>
- 要点：把常见失效分成电源、过流、时钟、通信、器件方向、信号完整性、焊接、损坏器件。清单：外观检查、万用表查短路、确认电源轨、打开限流、查时钟、查通信接口。电源立即撞电流限制先断电测对地短路，并核对稳压器反馈电阻和元件位置。时钟问题表现为 MCU 不跑、总线上无数据、系统像死机——先确认振荡器是否贴对、再上示波器看频率/幅度/振铃。SPI/UART 接反时，在 TX 上触发发送应看到波形，看不到就优先怀疑交叉。I2C 上拉过小可能拉不死。焊接问题（虚焊、桥连、回流不良）常表现为间歇故障，需放大镜检查并考虑助焊剂返修。强调不要随机探测，按电源到信号逐步缩小范围。

#### Safe Circuit Board Power-Up: Troubleshoot like a Pro

- 作者/机构：Mark Harris，Altium Academy
- 日期：2023-03-30
- 类型：视频（约 9 分 37 秒）
- URL：<https://www.youtube.com/watch?v=M1B4t4Ukrwk>
- 要点：信息来源为视频口播转写。Harris 给出一套可直接上实验台的流程：（1）用万用表测输入连接器和每一路稳压轨对地电阻；同一网络近 0 Ω，不同网络应为高阻或开路；有上拉/分压时 kΩ 级可能合理，但数百欧以下要警惕。读数反复确认，因为焊点上的助焊剂或氧化会造成接触不良。（2）用限流实验室电源上电：先关掉电源输出再接线，避免输出电容在限流启动前放电击穿器件。电流限制可先设为估计值的约 1.5 倍；许多板子可以从 50–100 mA 起步，但必须看原理图。进入恒流时若电压已经接近设定值，可能只是限流偏低；若电压明显很低，更像短路。（3）热像仪是他最有价值的排障工具：限流下哪怕不到 1°C 的温升也能指出漏电元件，比用手摸更早、更安全。（4）电源进入恒压后，再用表核对所有稳压输出是否与原理图一致，电压偏低可能是过流或反馈电阻错误。

#### AN3759：Boards Bring-Up Debug Tips（摘录部分）

- 作者/机构：Microchip Technology
- 日期：修订版 DS00003759B，2021-09-20（初版 2020-12-01）
- 类型：应用笔记 / PDF
- URL：<https://ww1.microchip.com/downloads/aemDocuments/documents/CPG/ApplicationNotes/ApplicationNotes/AN3759-Application-Note-DS00003759.pdf>
- 要点：这是针对 Microchip 某类嵌入式控制器的笔记，不是 AI 加速器文档，但第 4 节「Boards bring-up debug tips」是很少见的、写进官方文档的板级检查单。建议核对：各电源轨是否按数据手册时序到达并稳定；复位脚是否按参考设计上拉；所有 strap 脚电平（文中列举了 JTAG_STRAP 必须下拉、正常启动相关 strap 的默认方向等）。对 Shared SPI 启动，可在 SHD_CLK 上探时钟：完全无时钟则回头查电源与 strap；镜像已加载但不执行，则查镜像格式，并尝试降速或改变 SPI 读写模式。对 eSPI MAF，确认 Boot ROM 释放 RSMRST#，并在 PCH SPI CLK 上观察是否取指。JTAG/SWD：JTAG_RST# 需按参考设计上拉；**若有效镜像已成功执行，Boot ROM 可能关闭 JTAG/SWD**；空片才能直接挂 ARM 调试器，已运行的固件需通过 DEBUG ENABLE 寄存器重新打开。这条「成功启动反而失去调试口」的坑，在自研 SoC 上同样常见。

---


## 3. GPU / AI 加速卡硬件与互连 bring-up（PCIe、NVLink、HBM、热设计、ECC）

### 跨资源共识

加速卡 bring-up 最常见的失败不是「卡完全不工作」，而是**静默降速、链路上报错、ECC/HBM 退化或拓扑与预期不符**。这些状态仍能跑模型，但带宽、稳定性和正确性已经受损。

- **先记基线，再加压。** 记录每张卡的序列号、VBIOS/InfoROM、固件、ECC 累计、行重映射 / 页退役、PCIe 世代与宽度（LnkCap 对 LnkSta）、NVLink 错误计数和拓扑。Ampere 及以后的 ECC 累计通常不能清零，所以「上电即记录」才能把后续变化变成证据。
- **PCIe 降速是高发且容易被忽略的故障。** 一条 `lspci` 就能暴露 LnkCap 与 LnkSta 不一致。根因包括 BIOS 把 Link Speed 锁在 Gen1、Riser 规格不够、插不到位、对端能力不足。Meta 的经验是：先计算「应有速率/宽度 = 两端能力与上游目标的最小值」，再和当前状态比较；宽度/速率降级优先重插，不可纠正错误优先换件，可纠正错误要看速率是否越过平台阈值。
- **NVLink / NVSwitch 必须按系统而不是按单卡理解。** Xid 74 既可能是本端链路问题，也可能是对端设备挂了导致本端报错。解码 Xid 报文第一个 DWORD 的位：bit 4/5 偏 ECC/Parity，同一链路多次出现要报修；bit 21/22 偏信号完整性，先查机械连接；其他若干位也建议先重插再跑诊断。HGX 类系统还要看 Fabric Manager 是否在跑、版本是否与驱动分支锁定。
- **HBM / ECC 要看序列，不要只看单条 Xid。** 48（双比特）后面是否紧跟 63/64（行重映射记录成功/失败），以及 94（已隔离）与 95（未隔离）决定了是重启应用、重置 GPU 还是立刻重启节点。Llama 3 405B 在 54 天快照里，GPU 是意外中断的最大类，其中 HBM3 单独就是主要子类之一。
- **热与功耗会在集群尺度显现。** Llama 3 训练观察到昼间 1–2% 的吞吐波动，原因是中午温度触发 GPU DVFS；万卡同时增减功耗还会在数据中心造成数十兆瓦量级的功率起伏。单机 bring-up 就要建立空闲与满载的温度/功耗基线。

### 资源清单

#### NVIDIA GPU Debug Guidelines

- 作者/机构：NVIDIA
- 日期：文档标注最近更新 2026-11-11（Release 620 的 PDF 同样给出该日期）
- 类型：官方文档
- URL：<https://docs.nvidia.com/deploy/gpu-debug-guidelines/>
- 分册：<https://docs.nvidia.com/deploy/gpu-debug-guidelines/gpu-node-triage.html> ，PDF：<https://docs.nvidia.com/deploy/pdf/GPU_Debug_Guidelines.pdf>
- 要点：这是 GPU 服务器现场排障的首要手册。报修前应收集：OS 与驱动、关键日志、已做步骤、完整日志、`nvidia-bug-report.sh`、HGX 的 Fabric Manager 日志、DCGM 诊断日志，以及（若与应用相关）框架与版本。文档用流程图区分监控告警、用户报错和调试应用三类入口。常见 Xid 的动作表（文档 Table 1）应直接当作运维手册：例如 13 先跑 DCGM/现场诊断区分硬件与应用；48 若后随 63/64 则排空、等作业结束、重置 GPU，否则补现场诊断；74 按 DWORD 位分流；79（GPU fallen off the bus）排空并报修；94 重启该应用，95 在未开 MIG 时立即重启节点。还给出不同架构下「能否单独重置 GPU」的限制：Ampere + NVSwitch 且 Fabric Manager 未运行时，不能只重置单卡，需要重置所有经 NVLink 连在一起的 GPU 和 NVSwitch；Hopper 及以后 + NVSwitch 则可在更大程度上单独重置。DCGM 长测（`-r 3`）用于充分加压，官方在该页写大约 30 分钟量级；**Field diagnostic 才是判定 GPU 健康、通常也是启动 RMA 前需要的权威工具**，须按系统厂商指示运行。网络侧建议用 NCCL tests（尤其 `all_reduce_perf`）做节点对测：成对应给出相近性能，慢的一对要换搭配重测以隔离节点；上线前应留下带宽基线。

#### Introduction — XID Errors

- 作者/机构：NVIDIA
- 日期：与 Debug Guidelines 同属常更新文档族；目录页曾标注 2025-05 左右版本信息，抓取时目录仍指向现行站点
- 类型：官方文档
- URL：<https://docs.nvidia.com/deploy/xid-errors/introduction.html>
- 相关：<https://docs.nvidia.com/deploy/xid-errors/analyzing-xid-catalog.html> ，PDF：<https://docs.nvidia.com/deploy/pdf/XID_Errors.pdf>
- 要点：Xid 是驱动写到内核/事件日志的 GPU 错误报告，**最常见原因是驱动给 GPU 编了错误命令，或发给 GPU 的命令被破坏**；它可能指向硬件、NVIDIA 软件或用户应用。编号应作为继续调查的起点，而不是结论——PCIe 或帧缓冲损坏理论上可以伪造出几乎任意 Xid。NVSwitch 有类似的 SXid，细则在 Fabric Manager 用户指南；文档注明 SXid 说明适用于 Hopper 及更早代。分析页按 Ampere 及更新 GPU 给出目录化动作，应与 Debug Guidelines 的 Table 1 对照使用。

#### DCGM Diagnostics

- 作者/机构：NVIDIA
- 日期：用户指南正文引用的版本在 2026 年仍在更新（Rillor 文引用 2026-06-17 的用户指南）
- 类型：官方文档
- URL：<https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/dcgm-diagnostics.html>
- 要点：DCGM Diagnostics（前身 NVVS）定位是生产环境里的**系统级就绪与排障工具**，不是全面硬件诊断，也不能替代 field diagnostic 或走 RMA。四个级别的用途被写得很清楚：Level 1 作就绪指标，Level 2 作失败后的收尾，Level 3/4 给管理员作事后分析。Hopper 系统上官方测得的时间上限大致为：r1 < 2.5 秒；r2 在 8 卡上 < 10.5 分钟；r3 在 8 卡上 < 35 分钟；r4 在 8 卡上 < 2.25 小时。测试覆盖软件栈（NVML/CUDA 库与冲突）、硬件（待退役页、PCIe、NVLink、framebuffer、计算引擎）、集成问题（PCIe replay、拓扑、权限/cgroup、基本功耗与热）、以及压力（targeted power/stress、吞吐、nvbandwidth、NCCL、r4 的 memtest 与 pulse）。pulse / targeted power 用来打电源轨和电流尖峰，这是稳态烤机容易漏掉的。跑 NCCL 插件需要设置 `DCGM_NCCL_TESTS_BIN_PATH`，并以 systemd override 的方式注入到 `nvidia-dcgm` 服务，否则插件会被静默关掉。

#### Basic system health checks — AMD Instinct Customer Acceptance Guide

- 作者/机构：AMD
- 类型：官方文档
- URL：<https://instinct.docs.amd.com/projects/system-acceptance/en/latest/common/health-checks.html>
- 要点：这是 Instinct 单节点验收的第一步，不通过不要进入更重的验证。操作系统必须在 ROCm 兼容矩阵内。内核启动参数要正确，文档示例包括 `pci=realloc=off`、`pci=bfsort`、`iommu=pt`、`numa_balancing=disable` 等；Intel 主机还要加 `intel_iommu=on`。用 `dmesg` / `journalctl -p err` 确认没有关键 `amdgpu` 错误。系统内存有最低建议（例如 MI300X / MI308X 为 2.0 TB，MI325X 为 2.5 TB，MI350X / MI355X 为 3.0 TB）。用产品对应的 `lspci -d 1002:<did>` 确认八张加速器都在；再用 `-vvv` 查 `LnkSta`：期望 Speed 32GT/s、Width x16，且无 `FatalErr+`。空闲时用 `amd-smi monitor -putm` 看利用率应为 0%、时钟偏低、温度远低于 85°C。找不到全部 GPU 时，先确认 `pci=realloc=off` 并检查 OAM 扭矩。更系统的检查可交给 Cluster Validation Suite，但要先配好 `host_config.json`。

#### Blackhole FAQ and Troubleshooting / Installing a Blackhole Card

- 作者/机构：Tenstorrent
- 类型：官方文档
- URL：<https://docs.tenstorrent.com/aibs/blackhole/faq.html> ，安装：<https://docs.tenstorrent.com/aibs/blackhole/installation.html>
- 要点：这是目前少数公开、可操作的非 NVIDIA AI 加速卡硬件 bring-up 文档。安装前断电并做 ESD 防护。电源必须是 ATX 3.1 或更新，使用 12V-2x6（12+4 pin）；不要用 ATX 3.0 或更旧电源。线缆必须完全插到位，避免连接器附近急弯；转接头只能用电源厂商标明允许的产品，每路输入接到电源的独立输出。旧 12VHPWR 物理上能插，但因接触不良过热风险更高，不推荐。卡要坐平、扣具自然扣上，支架螺丝不能把卡拉歪，否则不要上电；桌面系统需防塌托。多卡要在上电前按拓扑把 QSFP-DD 线接好。枚举检查：`sudo update-pciids && lspci -d 1e52:`，每颗 Blackhole 芯片应有一条记录。若枚举失败：BIOS 里不要让 PCIe 停留在 Auto，强制 Gen 4.0 或 Gen 5.0；同时检查插到位和供电。TT-SMI 异常时，把 BIOS 里的 PCIe AER Reporting Mechanism 设为 `OS First`。文档给出空闲约 75 W（墙端会因 AC-DC 损耗更高）、可持续工作到 90°C、110°C 自动关机等运行特征。枚举成功后应先升级到最新固件再做软件栈安装。

#### 三台 4090 服务器 GPU 全部 PCIe 降级到 1.0——排查实录

- 作者/机构：卡卡带我飞（掘金）
- 日期：2026-04-04
- 类型：文章（经验帖）
- URL：<https://juejin.cn/post/7624447286865068072>
- 要点：自建 3 机 6 卡 RTX 4090 集群，在准备升级网卡时顺手查 PCIe，发现全部 GPU 跑在 PCIe 1.0（多数还是 x8）。作者给出的对比是：能力为 16 GT/s x16，状态为 2.5 GT/s x8，带宽只剩约 8%。三台机器、不同批次表现一致，因此不像单卡或单槽随机硬件损坏。作者的排查顺序：排除单点硬件 → 检查消费级卡在服务器里常用的 Riser 规格 → **最可能是 BIOS 的 PCIe Link Speed 被设成 Gen1**（出厂默认或装机时统一设置，能解释「全机一模一样」）。建议优先改 BIOS 为 Auto 或 Gen4，仍不行再换 PCIe 4.0 Riser 或升级 BIOS。影响评估也写得很清楚：模型从内存加载到显存会严重变慢；4090 无 NVLink，多卡 TP/EP 走 PCIe，基准测试会全面失真；单卡、权重已在显存内的推理影响较小。推荐的检查命令：`sudo lspci -vvv | grep -A2 "NVIDIA" | grep -E "LnkCap|LnkSta"`。这是消费级卡 + 国产服务器的案例，结论「先查 BIOS 与 Riser」可以迁移，具体带宽数字以现场 `lspci` 为准。

#### How Facebook deals with PCIe faults to keep our data centers running reliably

- 作者/机构：Ashwin Poojary, Bill Holland, Makan Diarra, Ray Park / Meta Engineering
- 日期：2021-06-02
- 类型：文章
- URL：<https://engineering.fb.com/2021/06/02/data-center-engineering/how-facebook-deals-with-pcie-faults-to-keep-our-data-centers-running-reliably/>
- 要点：虽早于大规模 H100 集群，但方法仍是 GPU/加速器机群 PCIe 治理的样板。他们把故障分成：不可纠正错误（即使一次也视为坏件，换卡，必要时换主板）、可纠正错误（要看速率，超过平台阈值会带来不可接受的性能下降）、以及静默的链路降速/降宽（硬件「还能用」，只是以 1/2、1/4 甚至 1/8 能力运行）。检测必须同时看链路两端（upstream 与 endpoint），并用 `LnkSta` / `LnkCtl` 等通用寄存器计算 expected_speed = min(上游目标, 两端能力)，current_speed 必须等于 expected_speed；宽度同理。降速/降宽优先重插，少数重启能恢复；对重复犯错的部件设规则，重插若干次仍失败再标记为坏件并更换。他们推荐用通用 AER + 开源 PCIcrawler，而不是各家私有工具，以便跨供应商、跨内核维护。自动化修复必须有速率限制，防止脚本 bug 把大批机器同时抽离生产。固件回归也曾造成特定供应商的降速，要用趋势数据而不是单机现象来抓。

---

## 4. Firmware / 驱动 / 运行时栈 bring-up

### 跨资源共识

软件 bring-up 的核心不是「把驱动装上」，而是把 **固件、BMC、内核驱动、CUDA/ROCm、通信库、容器运行时当作一个必须一起验证的版本元组**。

- **先锁版本矩阵，再装软件。** 固件包（VBIOS、NVSwitch、NIC、BMC、CPLD、PSU）要与驱动主版本匹配；Fabric Manager 与驱动分支锁定。AMD 侧 OS 必须在 ROCm 兼容矩阵内，内核参数不合格就不要继续。Gaudi 侧 SPI / eROM 固件要与软件版本对齐，升级前常需处理写保护、卸载驱动。
- **更新顺序和过渡包很重要。** DGX 类文档要求更新 VBIOS 前停止一切 GPU 活动（包括 `nvidia-smi`）；BMC 更新期间风扇加速是正常现象。某些 VBIOS 不能跨不兼容的数据结构直接升级，必须先打过渡包。更新过程中不要再跑诊断或用户作业。
- **带外（OOB / Redfish / IPMI）是机架级 bring-up 的地基。** 先确认能访问 BMC、电源控制不是 Skipped、凭据正确，再谈业务网和 NVLink 域。
- **驱动「装上了」不等于栈健康。** 要用 `nvidia-smi` / `amd-smi` / `hl-smi` / `tt-smi` 核对设备数量、温度非零、固件版本、PCIe 状态；再跑厂商资格测试（DCGM、hl_qual、Instinct CVS 等）。
- **通信库环境变量会静默毁掉多机性能。** 错误的 `NCCL_SOCKET_IFNAME`、`NCCL_IB_HCA`、`NCCL_P2P_DISABLE` 或容器内 RDMA 库与主机版本不一致，会出现「单机看起来正常、跨节点极慢或失败」。

### 资源清单

#### Firmware Update Steps — NVIDIA DGX B200 Firmware Update Guide

- 作者/机构：NVIDIA
- 类型：官方文档
- URL：<https://docs.nvidia.com/dgx/dgxb200-fw-update-guide/sequence.html>
- 相关：GB300 顺序说明 <https://docs.nvidia.com/dgx/dgxb300-fw-update-guide/sequence.html>；多节点 NVLink 系统固件概述 <https://docs.nvidia.com/multi-node-nvlink-systems/nvfupd-guide/introduction.html>
- 要点：更新前停止不必要的系统活动，**停止所有 GPU 活动，包括不要运行 `nvidia-smi`**，否则可能阻断 VBIOS 更新；也不要在更新期间叠加用户作业、诊断或监控负载。BMC 更新时风扇加速是预期现象，建议先备份配置。推荐顺序：用 `nvfwupd ... show_version` 对比已装与待装版本 → 先更新 BMC 并 `ipmitool mc reset cold`，确认 BMC 恢复 → 再更新主板托盘组件（主机 BIOS、EROT、PCIe Retimer/Switch、PSU、CPLD 等），系统需处于加电状态 → GPU 托盘若 VBIOS 早于某个分界版本，必须先打过渡包，再升到最新，否则数据结构不兼容，Redfish 路径会失败。文档还覆盖随后的 NIC / BlueField 等固件，并给出更新后需要重启才能生效、以及查询失败时的报错样例。原则可迁移到同类 HGX/DGX：先带外、再主板、再 GPU 托盘，注意过渡版本。

#### Installation Guide and On-Premise System Update / Platform Upgrade（Intel Gaudi）

- 作者/机构：Intel / Habana
- 日期：抓取到的最新安装总览为 Gaudi Documentation 1.24.2；下面许多具体命令来自仍公开的 1.14.0 平台升级页（流程结构一致，版本号须按当时 Support Matrix 更新）
- 类型：官方文档
- URL：<https://docs.habana.ai/en/latest/Installation_Guide/>
- 平台升级（1.14.0，命令更细）：<https://docs.habana.ai/en/v1.14.0/Installation_Guide/Platform_Upgrade_and_Unboxing.html>
- 要点：完整系统安装默认假设机械已组装、OS 已装好。先保证 BMC 可达（HLS-2 可能有两个 RJ45/两个 IP 分别对应 CPU 与 Gaudi 子系统；某些 Supermicro 则是单口单 IP）。用 `lspci` 按设备 ID 确认八张卡都在。先核对/升级平台级组件：CPLD、平台 BIOS、BMC，以及（HLS 上）PCIe 交换机版本。器件固件用 `hl-smi -L | grep SPI` 查看 SPI 预启动版本，再用发行包安装并以 `sudo hl-fw-loader` 升级；升级 SPI 需按平台要求解除写保护。Gaudi 2 在需要时还要升级 eROM：卸载 `habanalabs*` 模块、经 BMC/CPLD 关掉 eROM 写保护、用指定 `.itb` 文件烧录、再恢复写保护。装完软件后重新打开 shell 或 `source` 环境脚本。验证：`lsmod | grep habana` 看到驱动，`hl-smi` 中驱动版本匹配且 **Temp 不为 0°C**（0°C 表示卡初始化有问题，应重启并回查驱动安装），`dmesg` 无异常，SPI 版本与软件对齐，最后用 `hl_qual` 做硬件冒烟（1.14.0 页给出的例子是 `./hl_qual -gaudi2 -c all -rmod parallel -f2 -l extreme -t 60`，具体参数以当时 Qualification Library 为准）。

#### Multi-node setup for AI workloads（ROCm）

- 作者/机构：AMD ROCm
- 类型：官方文档
- URL：<https://rocm.docs.amd.com/en/latest/how-to/rocm-for-ai/system-setup/multi-node-setup.html>
- 要点：在多节点训练前，先完成 Instinct 多节点网络配置，并用官方 Docker + Slurm 示例。按网卡类型安装 RDMA/perftest 等软件包。若用 Broadcom 网卡，必须在容器里编译与**主机完全相同版本**的 RoCE 库（`libbnxt_re`），并把主机库路径正确挂进容器，否则会出现兼容性和通信失败。关键环境变量：`MASTER_ADDR`、`NNODES`、每节点唯一的 `NODE_RANK`；`NCCL_SOCKET_IFNAME` 必须是与其他节点同网段、状态为 UP 的接口（设错会导致失败或性能严重下降）；`NCCL_IB_HCA` 按 `ibv_devices` 列出的 RDMA 口填写（Broadcom 与 Mellanox 的命名不同）；RoCE 时还要设置 `NCCL_IB_GID_INDEX`（文档示例为 3）。这些是 ROCm 多机 bring-up 最常见的「软件已安装但仍不通」来源。

#### Adapting OpenBMC for NVIDIA Platforms: Challenges and Solutions

- 作者/机构：Tom Joseph & Deepak Kodihalli，OSFC 2024
- 日期：2024-09
- 类型：讲座（InfoCon 转写归档存在，抓取到目录与文件名；未在此逐字展开全部转写）
- 会议语境：<https://osfc.io/2024/schedule/>
- 转写目录：<https://infocon.org/cons/Open%20Source%20Firmware%20Con/OSFC%202024/OSFC%202024/>
- 要点：这是少见的、公开讨论「把 OpenBMC 适配到 NVIDIA 平台」的会议报告，与 DGX/HGX 的带外管理和固件更新直接相关。因本次未能完整精读全部转写正文，这里只确认讲座与归档真实存在，建议读者直接阅读同目录下 `Adapting OpenBMC for NVIDIA Platforms Challenges and Solutions - Tom Joseph & Deepak Kodihalli.eng.text`。与已核验的 NVIDIA 固件/机架文档对照时，关注点应是：Redfish 库存与更新路径、BMC 与主机/GPU 托盘的更新顺序、以及开源 BMC 与厂商平台组件之间的差异。不把未精读的口播细节写成断言。

---


## 5. 多卡 / 机架 / 集群上线与排查

### 跨资源共识

机架和集群 bring-up 的单位不再是一张卡，而是 **电源域、带外网络、NVLink / 以太网 / IB 域、固件 SBOM、以及可重复的验收记录**。

- **先把管理面和互连域带起来，再上计算节点。** NVIDIA 对 GB200/GB300 NVL72 的明确建议是：先上 NVLink Switch，让 NVLink 域配置完成后再给计算托盘上电；否则需要重启计算节点才能正确加入 fabric。
- **上线不是一次诊断，而是分阶段验收。** 典型顺序：基线记录 → 短诊断（就绪）→ 持续加压 / 功率脉冲 → 机内带宽（nvbandwidth、单机 all_reduce）→ 机间带宽（GPUDirect RDMA、多机 NCCL）→ 对照 Xid/ECC/链路计数 → 书面记录。DCGM 官方并不把任一 level 叫作 burn-in 或验收测试，现场要把「诊断」和「合同验收/烤机时长」分开写进流程。
- **成对比较，不要看单次绝对值。** NCCL `all_reduce_perf` 报告的 busbw 用于和瓶颈链路比较；NVIDIA 建议成对节点性能应接近，慢的一对要与其他节点交叉重测。上线前留下基线，以后才能判断是「从来就慢」还是「后来退化」。
- **大规模训练的中断以硬件为主，必须靠自动化。** Llama 3 405B 在 54 天快照中有 419 次意外中断，约 78% 属已确认或疑似硬件问题，GPU 是最大类（占意外中断的 58.7%）；有效训练时间仍高于 90%，期间显著人工介入只有三次。工具包括 NCCL flight recorder、看门狗超时自动转储、以及专门抓「还活着但很慢」的掉队者。
- **柠檬节点（反复出问题但不总失败的机器）要主动剔除。** Meta 研究集群论文指出，结合健康检查和历史分析剔除这类节点，可把大作业完成率提高 30% 以上。
- **维护窗口本身也是中断源。** Llama 3 期间仅计划内固件/内核升级等就会每天至少造成一次训练中断。集群 bring-up 必须包含可滚动、可验证的固件升级流程，而不是「先跑起来再说明天补固件」。

### 资源清单

#### GB200/GB300 Rack Power On and Bring Up

- 作者/机构：NVIDIA（Mission Control / BCM 安装指南）
- 类型：官方文档
- URL：<https://docs.nvidia.com/mission-control/docs/rack-bring-up-install/latest/rack-bring-up.html>
- 要点：这是公开文档里最完整的机架级 GPU 系统上电流程之一。总建议：先 bring-up NVLink Switch。计算托盘：先确认机架设备的带外电源控制（BCM 里 `power status`，输出 Skipped 通常表示电源控制未配置；failed 通常表示能访问 BMC 但凭据错误）→ 经 BMC 或物理上电并完成供应 → 从头节点登录，确认网卡 bond 和南北向网络起来 → 对照 SBOM 检查固件并按需更新。NVLink Switch：确认 BMC 可达（BCM 电源状态或直接 SSH）→ 安装 `cm-lite-daemon`（成功后设备显示 UP，并自动配置 NMX-C / NMX-T；失败时需手工指定一台 master 配置；若启用 ZTP 则可自动完成）→ 再核对固件与 NVOS。电源架：确认上报为 on，不行则从交换机侧 bounce 端口；然后做固件更新。NIC 固件单独作为一项收尾。文档后半给出手工拉取并安装 `fm_config`、重启 nmx-controller、启动 nmx-telemetry 的命令级步骤。适合作为 NVL72 类系统的操作剧本；命令与网络平面名称以现场 BCM 配置为准。

#### Burn-in and acceptance tests for GPU servers

- 作者/机构：Rillor
- 日期：2026-11-14
- 类型：文章（方法综述，逐条引用厂商文档）
- URL：<https://rillor.com/insights/gpu-server-burn-in-and-acceptance-tests>
- 要点：作者把公开工具整理成九段式验收：基线（序列号、固件、ECC、行重映射、PCIe/NVLink 计数）→ `dcgmi diag` 1–4 级 → 持续负载（DCGM targeted power / pulse、gpu-burn、DGX 上的 NVSM）→ 机内带宽（nvbandwidth、单机 all_reduce）→ 机间（`ib_write_bw` + GPUDirect、MPI 版 NCCL tests）→ 存储与主机（带校验的 fio、内核 PCIe 日志）→ 按 Xid 目录与行重映射退货标准读错误 → 把时长和通过值写进规则 → 留下书面记录和 NVIDIA 支持包。几个必须记住的边界：NVIDIA **没有**把 DCGM 任一级称为 burn-in 或验收测试，也不认为 DCGM 能替代 field diagnostic / RMA；公开厂商文档也**没有**规定 GPU 服务器必须烤多久，常见的只是工具默认时长或示例（如 `gpu_burn -d 3600`）。作者还提醒：Azure 等云厂商公布的 NCCL 带宽期望值只适用于其虚拟机型号；买方合同里的阈值应来自该系统规格和首次接受的基线。有一处转引「某研究集群 bring-up 时带宽损失可达 50%–75%」，本文未独立核对该原始论文数字，故不转写为已核实事实。

#### Pass NVIDIA AI infrastructure (NCP-AII) Module 1: GPU Server Setup, Bring-Up & Performance Validation

- 作者/机构：FlashGenius（认证辅导频道）
- 日期：2025-11-27
- 类型：视频（约 34 分 57 秒）
- URL：<https://www.youtube.com/watch?v=iIw0F4fUDQM>
- 相关文字：<https://flashgenius.net/ncp-aii-cheat-sheet>（NCP-AII 备考提纲，含 System/Server Bring-up 等权重说明）
- 要点：信息来源为视频口播转写。这不是 NVIDIA 官方课程，而是按 NCP-AII Domain 1 整理的「从货箱到可上线节点」叙述，技巧需对照官方文档。口播强调：多 GPU + 高速互连的系统必须按固定顺序 bring-up，跳步会在数月后变成难以追溯的不稳定。他们把过程概括为七个基础步骤，并归成三阶段——管理基础（BMC、固件、安全）、物理结构（拓扑、线缆、电源与散热）、运行就绪（`nvidia-smi`、DCGM、负载验证）。反复出现的检查项包括：双路电源冗余、BMC 可达且安全配置、固件兼容、Above 4G Decoding 等 BIOS 项、是否做了烤机。核心主张是每一步都在验证前一步已经集成正确；若直接跑大规模分布式训练，将失去根因定位能力。作为入门叙事有用，具体命令、BIOS 项和通过标准以 NVIDIA 官方安装与 Debug Guidelines 为准。

#### The Llama 3 Herd of Models（可靠性与运维部分）

- 作者/机构：Llama Team, AI @ Meta
- 日期：arXiv 2407.21783，首发 2024-07-31，v3 修订 2024-11-23
- 类型：论文
- URL：<https://arxiv.org/abs/2407.21783> ，HTML：<https://arxiv.org/html/2407.21783v3>
- 要点：第 3.3.4 节是目前最完整的公开「万卡训练集群在跑起来之后还会怎样坏」记录。405B 在最多 16K 颗 H100 80GB（HBM3，700W TDP）上训练，机内 NVLink，并使用 NCCL 的分叉 NCCLX。同步训练下单卡故障就可能让整作业重启。54 天预热快照：总中断 466 次，其中 47 次为计划内（自动固件/内核升级、配置或数据集更新等），419 次为意外中断；约 78% 的意外中断属已确认或疑似硬件问题；**GPU 问题占意外中断的 58.7%**。Table 5 给出的意外中断计数包括：Faulty GPU 148、GPU HBM3 Memory 72、Software Bug 54、Network Switch/Cable 35、Host Maintenance 32、GPU SRAM 19、GPU System Processor 17，以及 NIC、NCCL Watchdog Timeouts、Silent Data Corruption（6 次）、热界面与传感器、电源、SSD 等较小类别。尽管故障很多，有效训练时间仍高于 90%，显著人工介入仅三次。实践技巧：缩短作业启动与 checkpoint 时间；用 PyTorch NCCL flight recorder 把集合通信元数据和栈追踪写入环形缓冲，并在 watchdog / heartbeat 超时自动转储；更重的追踪可在生产中通过在线配置打开，不必发版或重启作业。NVLink 失败常常表现为 CUDA kernel 里 load/store 卡住且没有清晰错误码，NCCLX 与 PyTorch 协同做超时和内部状态快照（已完成/未完成的 rank 间传输）。另外要专门工具抓「仍然工作但很慢」的 straggler，否则一张慢卡会拖住数千 GPU。环境因素：昼间吞吐波动 1–2%，来自中午温度触发 DVFS。集群同时增减功耗可造成数据中心数十兆瓦量级的功率起伏。

#### Revisiting Reliability in Large-Scale Machine Learning Research Clusters

- 作者/机构：Apostolos Kokolis, Michael Kuchnik 等，FAIR at Meta
- 日期：arXiv 2410.21680（HTML 抓取为 v2）
- 类型：论文
- URL：<https://arxiv.org/abs/2410.21680> ，HTML：<https://arxiv.org/html/2410.21680v2> ，Meta 介绍页：<https://ai.meta.com/research/publications/revisiting-reliability-in-large-scale-machine-learning-research-clusters/>
- 要点：分析两个研究集群 11 个月的数据：约 400 万个作业、超过 1.5 亿 A100 GPU 小时、合计约 24k A100，集群利用率 >80%。与「只有超大 LLM 作业」的叙事不同，4k GPU 以上作业不到作业数的 1%，却消耗约 12% 的 GPU 资源，因此可靠性目标不能只为超大作业优化。论文给出故障分类、可靠性指标，并拟合故障模型以外推不同 GPU 规模下的 MTTF，还提出用作业参数估计 Effective Training Time Ratio（ETTR），用来评估软件缓解措施。运维上强调用健康检查和历史分析发现瞬时故障和柠檬节点；文中写到剔除柠檬节点可以把大作业完成率提高 30% 以上（Observation 11）。作者还提到与 NVIDIA 合作排查故障和规模化带宽问题，并补充新的健康检查。SDC 不是该文重点；他们的经验是研究集群里这类事件很少以用户投诉的形式出现。阅读时建议同时看公式化的 ETTR / 检查点间隔讨论，它直接决定「诊断跑多久」和「检查点做多勤」的权衡。

#### How Meta keeps its AI hardware reliable

- 作者/机构：Harish Dattatraya Dixit, Sriram Sankar / Meta Engineering
- 日期：2025-07-22
- 类型：文章（并指向同期 @Scale 演讲）
- URL：<https://engineering.fb.com/2025/07/22/data-infrastructure/how-meta-keeps-its-ai-hardware-reliable/>
- 要点：把硬件故障分成静态（不上电、不枚举，好查）、瞬态、以及静默数据损坏（SDC）。结合 Llama 3 经验指出：SRAM、HBM、处理阵列和网络交换机等故障会显著影响 AI 集群可靠性，训练中断中超过 66% 来自这类故障（该数字与 Llama 3 论文 54 天快照中「约 78% 意外中断属硬件/疑似硬件」不是同一口径，引用时不要混用）。AI 集群的特殊困难包括：加速器比 CPU 更复杂、遥测更少，网络复杂导致故障归属错误，GPU 软件栈配置错误也会造成长时间排查。方法上强调从工厂到机群的检测与诊断、快速用健康节点重启训练，以及跨越固件、编译器、内核和操作系统的栈级弹性（更细的故障遥测、梯度裁剪与算法容错等）。SDC 对训练和推理的正确性尤其危险。这是战略与方法综述，具体命令级步骤仍要回到 Xid/DCGM/PCIe 那一组文档。

#### Manufacturing of AI Systems Comprehensive Testing Approach

- 作者/机构：Gotham 与同事 Somia，Meta 产品质量 / 质量工程；会议现场演讲
- 日期：YouTube 页面可访问；具体会议名未在所抓转写开头明确给出
- 类型：视频
- URL：<https://www.youtube.com/watch?v=zUSrffD8oDE>
- 要点：信息来源为视频口播转写。与实验室 post-silicon 不同，这是制造与量产测试视角：目标是减少数据中心周末故障电话。讲者对比传统服务器测试与 AI 系统测试——后者异构（GPU、CPU、ASIC、存储器与互连必须一起工作），软硬件共验证更关键，用传统方法时异常容易漏检。测试要分层：器件/服务器功能块 → 机架级（完整固件、软件、硬件验证和连通性）→ 多机架/解决方案级，并建立持续监控与反馈，动态优化测试项。问答中他们承认制造环境没有验证实验室那么长的时间窗口，压力测试常见是 2–4 小时，早期有时到 12 小时；进入数千机架量产后再根据实际失效位置（例如存储器利用率问题）缩短或调整，超过某点是收益递减。对集群 bring-up 的启发是：上线烤机时长应来自反馈回路，而不是一个永恒的固定小时数。

---

## 6. 调试工具与常见故障

### 跨资源共识

现场调试最贵的两种错误是：**把应用 bug 当成坏卡换掉**，以及 **把正在退化的 HBM/链路当成软件问题反复重启**。

- **先分流，再读码。** 三个问题能处理掉大部分工单：是单进程崩溃还是同卡全崩？是同一作业必现还是换作业也随机？把作业迁到另一张卡是否消失？Xid/SXid 是调查起点，不是判决书。
- **先存证，再重置。** 重置/重启会毁掉易失线索。至少留下：按 PCI 地址聚合的 Xid 时序、`nvidia-smi` 的 ECC/行重映射/温度功耗、`lspci` 与 AER/dmesg、NVLink 状态与错误计数、`nvidia-bug-report.sh`、HGX 的 Fabric Manager 日志、DCGM 输出。
- **链路类故障要看对端和位域。** Xid 74 不要只数出现次数；Xid 79 高度指向 PCIe 链路硬件问题。可纠正 AER 看速率，不可纠正看一次即处理。
- **工具分层：** 板级用表、限流电源、示波器、热像仪、JTAG；硅上用 scan/scandump、BIST、时钟监测；GPU 服务器用 nvidia-smi / dcgmi / amd-smi / hl-smi、NCCL tests、CUDA-GDB / Nsight；集群用成对带宽基线、flight recorder、柠檬节点历史。
- **换件后必须复验。** 诊断全绿、ECC/pending 干净、PCIe/NVLink 速率回到预期、集合通信回到原基线，并把 SN、槽位、Xid 序列和复验数据入库。

### 资源清单

#### GPU 服务器故障诊断手册：XID 错误码、ECC 与换件决策

- 作者/机构：AI-HPC 智算超算专家（ai-hpc.org）
- 类型：文档 / 手册（中文，基于 NVIDIA 官方指南的操作化改写）
- URL：<https://ai-hpc.org/zh/guide/02-hardware/gpu-fault-diagnosis>
- 要点：这是把 NVIDIA Debug Guidelines / Xid 目录转成现场决策的中文手册，命令和建议应回头核对官方原文。核心主张：不要一报错就重启；先把问题分成应用、驱动、硬件。应用类：13/31 优先用 Compute Sanitizer `memcheck` 或 CUDA-GDB 查代码；43/45 通常是应用被终止的伴随信息，45 在 A100 上还可能只是与 GPU 同机的 Fabric Manager 异常退出。显存类必须看 48 与 63/64、94 与 95 的组合，以及 `nvidia-smi -q -d ECC,PAGE_RETIREMENT,ROW_REMAPPER` 的数字。链路类给出 74 的 DWORD 位判读和「先查对端」警告，并配合 `nvidia-smi nvlink -e` 的 CRC/replay 计数区分偶发与持续退化。手册还给出取证命令清单（Xid 日志、ECC、lspci/AER、NVLink）和换件后五步复验（DCGM、ECC、链路速率、NCCL 基线、归档）。三个高频误判：把 43/45 当硬件、把 94 当 95、只看本端不看对端。

#### NVIDIA Fabric Manager（NVSwitch 系统）

- 作者/机构：NVIDIA
- 类型：官方文档
- URL：<https://docs.nvidia.com/datacenter/tesla/fabric-manager-user-guide/>
- 要点：HGX / NVSwitch 系统 bring-up 的关键守护进程：负责把 NVSwitch 存储结构配成参与 GPU 之间的统一 fabric，并监控支撑该 fabric 的 NVLink。GPU Debug Guidelines 明确要求报修时附上 FM 日志，SXid 的解释也在本手册。实践中必须与已加载驱动的主版本分支对齐（常见安装形态是按驱动主版本选择对应的 `nvidia-fabricmanager-<branch>` 包），并用 `nvidia-smi -q -d FABRIC` 等查询确认状态。FM 未运行时，部分架构下不能做单 GPU 重置。A100 上单独出现的 Xid 45，官方也提醒要怀疑 FM 与 GPU 同 OS 环境时异常退出。本文主要依据官方交叉引用与现行文档入口核验，细节操作以该用户指南正文为准。

#### NCCL Tests

- 作者/机构：NVIDIA
- 类型：工具 / 开源仓库
- URL：<https://github.com/NVIDIA/nccl-tests>
- 相关说明也见 Debug Guidelines 的 Network Testing 一节。
- 要点：集群 bring-up 和日常回归中最常用的集合通信基准与正确性检查。`all_reduce_perf` 是节点内和节点间的标准探针；其 busbw 把算法带宽乘上 2(n-1)/n，便于和硬件瓶颈链路对比。官方不设统一通过线，要通过「成对可比 + 上线基线」来用。多轨网络上可隔离到特定轨道，判断是哪一张网卡或哪一条链路的问题。DCGM 的 NCCL 插件依赖这些二进制的路径配置。

---

## 7. 公司 / 开源案例与教程

### 跨资源共识

公开、能动手照做的「第一硅 / 第一块板」日记，在闭源 GPU 厂商那边几乎不存在；更常见的是 **验收手册、驱动/固件安装、模型向硬件的逐级搬迁，以及制造与良率叙述**。开源程度最高的是 Tenstorrent 的板卡安装、排障和模型 bring-up 文档。Cerebras 公开的是晶圆级良率、分区调压与在役修复，而不是传统 JTAG 上电步骤。Google TPU、Groq、Graphcore、SambaNova 的硅后实验室流程基本没有可核验的公开操作手册（见文末缺口）。

把「芯片 bring-up」和「模型 bring-up」分开看会更清楚：前者证明硅和板是活的、接口是干净的；后者证明编译器/运行时在这台设备上能逐层复现参考精度。两者共用「由简到繁、每步对照参考、不要一上来就上全模型全系统」的纪律。

### 资源清单

#### Tenstorrent Model Bring-Up

- 作者/机构：Tenstorrent（tt-metal 仓库）
- 类型：开源文档
- URL：<https://github.com/tenstorrent/tt-metal/blob/main/models/docs/model_bring_up.md>
- 原始文件：<https://raw.githubusercontent.com/tenstorrent/tt-metal/main/models/docs/model_bring_up.md>
- 要点：这是把模型搬上自研加速器的逐步方法，可类比编译器/运行时 bring-up。先在 CPU/GPU 上跑通参考实现和权重，再改编。能放下就先用单设备、更小的模型（文档写 Wormhole 约 12 GB DRAM，BFP8 大约可到 12B 参数量级）。LLM 建议系统化、按组件进行：先 decode 再各个模块（RMSNorm、RoPE、Attention、MLP 等），同一输入分别送参考实现和 TT-NN，用 PCC 对照；再拼成单层/整网，先用 decode 跑 prefill；全模型测试用真实文本，**教师强制（teacher forcing）** 是推荐的精度核对法，并看 top1/top5 相对参考 token 的准确率。注意浮点与非线性近似会导致 token 不完全一致。多卡先按模块做 data parallel 单元测试再上全模型；放不下的模型再用 tensor parallel。性能优化（Metal Trace、async、多命令队列）放在功能正确之后。配套还有 LLM / CNN 的 tech report 链接。

#### How to Yield at Wafer Scale

- 作者/机构：Isaac Tai / Cerebras；致谢 David Greenhill、Jean-Philippe Fricker、Alvin Jee 等
- 日期：2026-11-10
- 类型：文章
- URL：<https://www.cerebras.ai/blog/how-to-yield-at-wafer-scale>
- 要点：不是传统 bench bring-up 日志，而是晶圆级 AI 芯片如何在制造和服役中「活下来」的工程说明。WSE 用大量小核加冗余应对缺陷；设计阶段采用更宽的时序签核目标（文中称 setup 最高约四 sigma、hold 最高约七 sigma），并遵循 Foundry DFM 推荐规则。硅后按 84 个裸片区域独立调压，避免整片被最差区域的电压拖死，从而在满足频率的同时节省动态功耗（文中用两区 0.70 V / 0.75 V 的例子估算量级）。每块 reticle 有大量传感器（文中为 184 个传感器块）：环形振荡器测局部速度，droop 检测、延迟链和片上采样阵列捕捉电源跌落。工作负载切换时的电流突变靠硬件限制活动变化率。服役后仍可用剩余冗余核做软件可重构修复，而不必把晶圆从系统中取出。对做自研 AI 芯片的启发是：bring-up 和可制造性、在役修复是同一套设计决策，不能等首硅到了再补。

Etched / Synopsys 的预硅真实负载案例见第 1 节《A Bumblebee Moment for AI Silicon》，此处不重复著录。

#### Bolt Graphics Test Chip Demo

- 作者/机构：Bolt Graphics（讲者口播中名为 Darwesh 的现场演示）
- 类型：视频
- URL：<https://www.youtube.com/watch?v=CgyVCNToNto>
- 要点：信息来源为 YouTube 页面说明与转写，不是完整的实验室手册。团队讲述 12 nm TSMC 测试芯片于 4 月流片，硅片回来约三四周后完成封装并上板，**24 小时内上电且未发现功能问题**，功耗好于预期；演示时芯片运行在 80 MHz。作为「首硅当天能跑 demo」的故事有参考价值，但公开材料几乎不涉及电源顺序、JTAG、失败路径或调试工具，不能当作 bring-up 教程。

---

## 建议的现场最小清单（跨资源提炼）

下面不是某一家的官方流程，而是把已核验资料里反复出现的步骤收成一张「最小集合」，实施时用对应厂商文档替换命令和阈值。

**自研芯片 / 第一块板**

1. 上电前：外观、对地电阻、strap/配置开关放到最简模式，限流电源输出先关闭再接线。
2. 按规格顺序上电，记录每路电流；异常立即断电，换板/换片定位。
3. 外时钟或旁路 PLL → 复位 → 读 JTAG IDCODE 并做寄存器读写。
4. 再开 Boot ROM / 调试器加载，跑预先在仿真器上排演过的冒烟测试。
5. 需要时准备 scandump / 跟踪 / 时钟监测；成功启动可能关闭调试口，要有重新打开的路径。

**GPU / 加速器服务器**

1. 带外：BMC 可达，电源控制可用，固件与 SBOM/兼容矩阵一致；需要时按顺序升 BMC → 主板 → GPU 托盘（含过渡包）。
2. 枚举：卡的数量、`LnkCap` vs `LnkSta`、NVLink/Infinity Fabric 拓扑、空闲温度与功耗。
3. 记录基线：序列号、固件、ECC、行重映射、PCIe/NVLink 计数、Xid 历史。
4. 短诊断（如 `dcgmi diag -r 1` 或厂商等价物）作就绪检查；更长诊断和功率脉冲/烤机按验收条款执行。
5. 单机再多机跑带宽与正确性（nvbandwidth、all_reduce_perf、ib_write_bw）；留下成对基线。
6. 故障时先分流和存证，再重置；换件后复验并归档。

**机架 / 集群**

1. 先互连域（NVLink Switch 或对等 fabric）和带外网络，再批量上计算节点。
2. 固件、驱动、CUDA/ROCm、NCCL/RCCL、容器运行时作为单一元组管理和回滚。
3. 建立柠檬节点与静默降速的持续监测（PCIe AER 速率、NVLink replay、ECC 趋势、NCCL 基线漂移）。
4. 为大作业准备自动诊断、flight recorder、快速换节点重启和检查点策略。

---

## 资源统计

统计口径：上面各节「资源清单」中的独立条目。同一文档的分册 URL 不重复计数；第 7 节对 Etched / Synopsys 一文只作交叉引用，不重复计数。共 37 条独立资源。

| 类型 | 数量 | 说明 |
|------|------|------|
| 文章（工程博客、教程、方法综述、应用笔记、中文经验帖） | 15 | ChipVerify 三篇，SemiEngineering 两篇，Synopsys 与 Edge AI/Synopsys 转载各一篇，SolderStack，Microchip AN3759，Rillor，掘金 PCIe 实录，Meta 工程博客两篇（PCIe 故障；AI 硬件可靠性），Cerebras 良率，ai-hpc 中文诊断手册 |
| 官方文档 / 手册 | 10 | NVIDIA：Debug Guidelines、Xid、DCGM、Fabric Manager、GB200/GB300 机架、DGX 固件更新；AMD Instinct 验收、ROCm 多节点；Intel Gaudi 安装/升级；Tenstorrent 安装与 FAQ（合并为 1 条） |
| 论文 | 2 | Llama 3 Herd；Revisiting Reliability |
| 开源仓库 / 工具文档 | 2 | Tenstorrent model bring-up；NCCL tests |
| 视频 / 讲座 | 8 | Altium 上电；VLSI FOR ALL post-silicon 课；Sundar Vedantham bring-up；OSFC 0-day 软件（讲座+转写）；OSFC OpenBMC（确认归档，正文未精读）；FlashGenius NCP-AII；Meta 制造测试；Bolt Graphics 测试芯片演示 |
| **合计** | **37** | |

若按「文章类（含官方文档、论文、开源文档、经验帖）」与「视频/讲座」二分：文章类 29，视频/讲座 8。

中英文分布：中文可直接阅读的操作文主要是 ai-hpc.org 手册与掘金 PCIe 降级实录；其余以英文官方文档、论文和视频为主，中文综述已把可操作要点写出。

---

## 检索缺口与局限

以下是检索中客观存在的空白，或已遇到但质量不足以当作「实操来源」的情况。

1. **Groq、Graphcore、SambaNova 缺少可核验的硅后 / 板级 bring-up 手册。** 公开材料以架构介绍、招聘职位（例如 Graphcore 的 SoC Validation / Bring-up lead）和产品对比为主，没有找到与 Tenstorrent/Gaudi/NVIDIA 文档同等级的上电、JTAG、固件与验收步骤。
2. **Google TPU 的芯片与板级 bring-up 基本不公开。** 能公开获取的是 Cloud TPU 使用、JAX/XLA/Pathways 与芯片软件共设计叙述，而不是 first silicon、DFT 或机架上电剧本。
3. **HBM PHY、UCIe / 芯粒互连的实验室 bring-up 笔记几乎都在 NDA 下。** 公开讨论停留在 DFT 方法、良率和架构层，缺少可引用的示波器/眼图/训练序列操作指南。
4. **NVIDIA Field Diagnostic 与完整 RMA 流程是厂商受限工具。** 公开文档只说明它是健康判定的权威工具，并要求按系统厂商指示运行；本文无法提供其内部步骤。
5. **中文「自研 AI 芯片 first silicon」深度经验很少。** 能核验的中文高质量材料更集中在 GPU 服务器运维（Xid、PCIe、NVLink），而不是 ASIC 实验室日记。B 站上能检索到 HGX 模组安装、维保类视频，但页面往往偏商业服务展示，技术密度不足，本文未把未抓到实质技巧的视频硬收进来。
6. **部分视频的信息来源是自动转写或会议简介**，可能有专有名词拼写错误（已在文中标明）。FlashGenius 视频是认证辅导而非官方教材；Sundar Vedantham 视频是公开上传的内部风格讲座；OSFC OpenBMC 一场只确认了归档存在，未把未精读内容写成步骤。
7. **Semiconductor Engineering 与部分 EDA/IP 博客带赞助性质。** 已选用其中确实包含方法（层次化 DFT、scandump、预硅负载）的篇目，并标明厂商视角；未把纯工具宣传当作现场规程。
8. **个别搜索结果未采用：** 例如部分芯片招聘描述、仅有摘要的 IEEE 教程条目、以及未能打开正文或内容过浅的二手转载。Rillor 文中转引的「某集群 bring-up 带宽损失 50%–75%」未在本次对原始论文的核对中确认，故未写入正文事实陈述。
9. **链接时效。** 官方文档会改版；Gaudi 1.14.0 的具体包版本和 `hl_qual` 参数尤其容易过期。建议以各文档「latest」入口和当时 Support Matrix 为准。本次抓取时上述 URL 均可打开；未发现必须剔除的 404。Microchip PDF 与部分 NVIDIA 文档在不同网络环境下可能出现中间设备拦截，但本次抓取得到了正文。

---

## 主要资料入口（便于放入仓库 README）

- NVIDIA GPU 排障与 Xid：<https://docs.nvidia.com/deploy/gpu-debug-guidelines/> ，<https://docs.nvidia.com/deploy/xid-errors/introduction.html>
- DCGM 诊断：<https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/dcgm-diagnostics.html>
- GB200/GB300 机架上电：<https://docs.nvidia.com/mission-control/docs/rack-bring-up-install/latest/rack-bring-up.html>
- AMD Instinct 验收：<https://instinct.docs.amd.com/projects/system-acceptance/en/latest/common/health-checks.html>
- Tenstorrent 板卡与模型 bring-up：<https://docs.tenstorrent.com/aibs/blackhole/installation.html> ，<https://github.com/tenstorrent/tt-metal/blob/main/models/docs/model_bring_up.md>
- 硅后上电与时间线：<https://chipverify.com/post-silicon-validation/first-power-on>
- 万卡故障数据：<https://arxiv.org/abs/2407.21783> ，<https://arxiv.org/abs/2410.21680>
