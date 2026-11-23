# AI for Chip 条目清单

截至 2026-11-23（Asia/Shanghai）。用于每周追加，不必先改叙述页。

## 2026-11-23 周报新增

- **2026-11-20** · 论文 · 前端 · From Seed to BLOOM (MiracleGro)（arXiv:2611.26517）
  Berkeley：智能体 MiracleGro 在无人工介入下分析周期级 RTL 与综合结果，改写开源乱序核 BOOM。论文称 SkyWater 130nm 上 IPC +85%、频率 +27%，完整 SPEC CPU2017 intspeed（约 30 万亿条指令）墙钟加速 2.34 倍。
  https://arxiv.org/abs/2611.26517

- **2026-11-20** · 论文 · 前端 · Verify the Verifier（arXiv:2611.26588）
  德州农工：用 LLM 交叉检查对照规格，捕捉通过 UVM scoreboard 但仍违反规格的缺陷。向四个 OpenTitan IP 插入 564 个缺陷，其中 68 个为候选 scoreboard escape，框架确认 41 个（断言在有缺陷设计上失败、在原设计上得证）。
  https://arxiv.org/abs/2611.26588

- **2026-11-20** · 论文 · 后端 · SGLayout（ICCAD 2026 / arXiv:2611.25410）
  北大、ChatEDA 与港中文：用 schema 把几何评估结果反投影到可修改的代码位置，闭环生成参数化版图。112 题 GDSFactory 基准上通过率 76.79%，最强基线为 50.89%。
  https://arxiv.org/abs/2611.25410

- **2026-11-20** · 论文 · 前端 · GRODE-RTL（arXiv:2611.25661）
  港中文与中科院计算所：从网表提取可疑寄存器的单时钟级扇入锥并映射回源码，做成按跳数折叠的依赖视图，辅助 LLM 修非局部 RTL 故障。30 个形式验证过的非局部故障上，相对源码级探索，三个模型的 pass@1 分别提高 16.7%、26.7%、20.0%。
  https://arxiv.org/abs/2611.25661

- **2026-11-20** · 论文 · 前端 · BOOMerang（arXiv:2611.25438）
  Berkeley：智能体闭环把 BOOM 的 RTL 与 gem5 模型对齐。论文称两天内做出 IPC 误差 3% 以内的 gem5 模型，并把 72 项微架构特性译回 RTL；组合后 SPECint 2017 性能提升 25%。
  https://arxiv.org/abs/2611.25438

- **2026-11-20** · 论文 · 后端 · ReLay（arXiv:2611.25328）
  布朗与密歇根：从已有模拟版图恢复生成器程序，并让库函数随版图积累演化。基准含 27 个电路族、三个开源 PDK 上的 1050 张手绘版图；学到的函数占留出集布局调用的 53–66%。前沿 LLM 智能体用 ReLay 完成任务的 API 成本比直接生成几何低 64–77%，完成率与电学质量相当。
  https://arxiv.org/abs/2611.25328

- **2026-11-19** · 论文 · 前端 · CRAFT（arXiv:2611.25908）
  三星与首尔大学：LLM 智能体通过预计算查找表服务 DeX 按手算方式定参，循环中不跑 SPICE，最终用 HSPICE 独立验证。商业 28nm 上四个电路（最大为 37 管三级放大器）共 20 次全部满足规格与约束；该放大器上相对 HSPICE 闭环，10/10 对 6/10 通过，中位快 2.0 倍。
  https://arxiv.org/abs/2611.25908

- **2026-11-19** · 论文 · 前端 · Meta-RTL（arXiv:2611.25891）
  港中文：在固定优化模型、EDA 流程和预算下演化 RTL 优化智能体外壳。相对 Dr. RTL，20 个开发设计上聚合 WNS/TNS 违例再降 7.8%/11.0%，70% 设计面积下降；迁移到 10 个未见的 IC-RTL 设计后，聚合 WNS/TNS 再降 20.5%/14.0%，总综合面积 -12.6%。
  https://arxiv.org/abs/2611.25891

- **2026-11-19** · 论文 · 跨层 · SiliconGym（arXiv:2611.25658）
  加州大学圣地亚哥分校、佐治亚理工与 NVIDIA：跨任务、跨工具链的硬件设计与验证强化学习环境，覆盖模块级 RTL/测试台/PPA、仓库级修复和全层次 SoC。流式调度使 256 条回答的 RL 批次比原生 ChipSeek 快 21.9%；并比较开源训练成果迁移到商业仿真与综合时的功能正确性和 PPA。
  https://arxiv.org/abs/2611.25658

- **2026-11-18** · 论文 · 前端 · DVBench（arXiv:2611.25066）
  面向真实端到端 UVM 验证智能体的基准，含 95 题并保留各阶段依赖。13 种智能体–模型配置中，最强的 Codex 与 GPT-6 Astra 组合总成功率为 21.05%。
  https://arxiv.org/abs/2611.25066

- **2026-11-18** · 论文 · 前端 · Inference-Time Scaling in Agentic Hardware Verification Workflows（ICLAD 2026 / arXiv:2611.23905）
  Arm：在 AMBA 桥的智能体 UVM 测试台生成中比较四种推理时缩放策略。结构化诊断反馈相对原始 EDA 反馈，每美元 PVC 大约高 15 倍；双环策略在全部四个目标上达到 PVC ≥ 0.95 且无 scoreboard 错误，并在四个参考设计中的两个里暴露了潜伏 RTL 缺陷。输出由 Mentor Questa 周期精确工具链判定。
  https://arxiv.org/abs/2611.23905

- **2026-11-09** · 论文 · 跨层 · CASCoDe（arXiv:2611.11710）
  德州大学奥斯汀分校 David Z. Pan 组：把模拟定参、结构 floorplan、行内 refinement 和布线尝试做成层次化离散动作，闭环做到经验证的 GDS。20 个 SKY130 放大器全部给出 DRC 与 LVS 干净的布线版图，是对照中唯一全部合法的；相对常规 P&R，面积降 59–85%（均值 71%），平均器件密度从 0.39 提到 0.74，平均后仿 FoM 高于 APPlace、MAGICAL、ALIGN（后三者 DRC 干净数分别为 15、0、1）。打赢全部基线的版图平均 11 分钟智能体时间出现。
  https://arxiv.org/abs/2611.11710

- **2026-11-09** · 论文 · 后端 · Evaluating AI Agents for Industrial-Level Chip Floorplanning（arXiv:2611.12048（v2 2026-11-12））
  Nova-Silicon：把工业级宏块 floorplan 接到规定的下游实现流程再评分。一个 99 个宏块、约占固定 L 形区域 71% 的工业设计上，40 次评估里专用系统 Nova-SI 的综合质量均分 57.97，最强通用配置 GPT-6 Astra 为 31.17。
  https://arxiv.org/abs/2611.12048

- **2026-11-08** · 论文 · 后端 · ICLayout-Bench（arXiv:2611.12982）
  中科大与中科院计算所：评测 LLM 智能体从晶体管级网表做到 GDSII 版图，含物理验证、寄生抽取和后仿。165 题、13 个 PDK（3nm 到 180nm）；7 个模型共 22 种配置在公开/私有任务上的平均版图分分别为 4.22–80.26 和 1.43–67.01（100 分制）。公开集最高为 GPT-6 Astra medium 的 80.26，私有集最高为 GPT-6.1 Sol high 的 67.01；提高推理力度并不稳定提升均分。
  https://arxiv.org/abs/2611.12982

## 既有条目

- **2025-06-10** · 论文 · 后端 · ORFS-agent: Tool-Using Agents for Chip Design Optimization（arXiv:2506.08332（v3 2026-04-30））
  Kahng 等在 OpenROAD 流程用 LLM 智能体调参；相对 OR-AutoTuner 迭代少约 40%，若干目标最多改善约 1–2.7%。
  https://arxiv.org/abs/2506.08332

- **2025-06-17** · 论文 · 前端 · CVDP: Comprehensive Verilog Design Problems（arXiv:2506.14074）
  NVIDIA 783 题、13 类 RTL/验证基准；当时代码生成 pass@1 不超过 34%。
  https://arxiv.org/abs/2506.14074

- **2025-07-20** · 论文 · 跨层 · MMCircuitEval（ICCAD 2025 / arXiv:2507.19525）
  3614 题多模态电路基准，覆盖前端到后端；现有模型在后端和复杂计算上差距明显。
  https://arxiv.org/abs/2507.19525

- **2025-07-22** · 论文 · 前端 · RealBench（arXiv:2507.16200）
  真实开源 IP 级 Verilog 基准；文中 o1-preview 模块级 pass@1 为 13.3%，系统级为 0%。
  https://arxiv.org/abs/2507.16200

- **2025-10-21** · 创业公司 · 前端 · ChipAgents Series A $21M（ChipAgents 新闻稿）
  Bessemer 领投 2100 万美元 A 轮，累计 2400 万美元；Micron、MediaTek、Ericsson 等参投。
  https://chipagents.ai/newsroom/chipagents-series-a-21m

- **2025-11-10** · 业界 · 前端 · Cadence 收购 ChipStack（Cadence 官方博客）
  验证智能体创业公司 ChipStack 加入 Cadence，成为 2026 年 Super Agent 产品线前序。
  https://community.cadence.com/cadence_blogs_8/b/corporate-news/posts/cadence-welcomes-chipstack

- **2026-01-29** · 论文 · 前端 · ChipBench（arXiv:2601.21448）
  生成、调试与参考模型三项任务；文中 Claude-4.5-opus 的 Verilog 生成为 30.74%。
  https://arxiv.org/abs/2601.21448

- **2026-02-16** · 创业公司 · 跨层 · Ricursive 种子轮与 A 轮共 $335M（TechCrunch）
  Goldie 与 Mirhoseini 创立；Sequoia 种子轮 3500 万、Lightspeed A 轮 3 亿，估值 40 亿美元；NVIDIA 为投资方。报道日期 2026-02-16，融资发生在此前约四个月。
  https://techcrunch.com/2026/02/16/how-ricursive-intelligence-raised-335m-at-a-4b-valuation-in-4-months/

- **2026-02-10** · 论文 · 前端 · ACE-RTL（arXiv:2602.10218（v2 2026-03-21））
  NVIDIA：领域 RTL 模型与推理模型通过上下文演化协同；CVDP 上相对 14 个对照最多 +41.02 个百分点通过率。
  https://arxiv.org/abs/2602.10218

- **2026-02-10** · 业界 · 前端 · Cadence ChipStack AI Super Agent 发布（Cadence 新闻稿）
  前端设计与验证智能体；官方称最多约 10 倍生产力。早期客户包括 Altera、NVIDIA、Qualcomm、Tenstorrent。
  https://www.cadence.com/en_US/home/company/newsroom/press-releases/pr/2026/cadence-unleashes-chipstack-ai-super-agent-pioneering-a-new.html

- **2026-02-17** · 创业公司 · 前端 · ChipAgents Series A1 $50M（ChipAgents 新闻稿）
  Matter VP 领投 5000 万美元，累计 7400 万美元。客户与倍速数字为公司自述。
  https://chipagents.ai/newsroom/chipagents-series-a1-74m

- **2026-03-16** · 业界 · 跨层 · Siemens Fuse EDA AI Agent（Siemens 新闻稿 / GTC 2026）
  跨半导体、3D IC 与 PCB 的编排代理；稿件称 NVIDIA 在芯片研发中使用 Fuse。
  https://news.siemens.com/en-us/siemens-fuse-eda-ai-agent/

- **2026-04-21** · 论文 · 前端 · ChipCraftBrain（arXiv:2604.19856）
  验证优先的多智能体 RTL 生成，针对更难工业基准与综合意识不足的问题。
  https://arxiv.org/abs/2604.19856

- **2026-05-13** · 论文 · 前端 · ChipMATE（arXiv:2605.12857）
  可自训练的多智能体：Verilog 与 Python 参考模型互相校验，面向隔离部署。
  https://arxiv.org/abs/2605.12857

- **2026-06-01** · 业界 · 前端 · Cadence Level-5 虚拟工程师（Cadence 新闻稿 / Computex 2026）
  ChipStack 延伸到 Level-5，跑在 NVIDIA OpenShell 上；官方称 NVIDIA 侧 RTL 验证超过 40 倍、五周环缩到不到一天。计划 2026 下半年早期提供。
  https://www.cadence.com/en_US/home/company/newsroom/press-releases/pr/2026/cadence-unveils-industrys-first-fully-autonomous-virtual.html

- **2026-06-15** · 论文 · 后端 · PDAgent-Bench（arXiv:2606.17253）
  353 题物理设计智能体基准；工具执行仍弱，Innovus 脚本生成约 42.2%。
  https://arxiv.org/abs/2606.17253

- **2026-07-20** · 论文 · 后端 · MAGE（arXiv:2607.18536）
  Kahng 等：多模态多智能体按专家规则做类人宏块放置，并提出人类风格指标。
  https://arxiv.org/abs/2607.18536

- **2026-07-26** · 业界 · 跨层 · Synopsys DAC 长程验证代理（Synopsys 新闻稿 / DAC 2026）
  与 NVIDIA 合作：最多 50 倍到已验证 RTL、覆盖 +20%；AMS 最多约 3 倍。计划 2026 下半年提供。
  https://news.synopsys.com/2026-07-26-Synopsys-Showcases-Comprehensive-Autonomous-Engineering-Workflows-from-Silicon-to-Systems,-Developed-with-NVIDIA-Technology

- **2026-07-26** · 业界 · 跨层 · Siemens DAC 自验证代理（Siemens 新闻稿 / DAC 2026）
  Fuse 接入 Intelligence Center X；Solido 特征化周转超过 10 倍。
  https://news.siemens.com/es-mx/siemens-nvidia-dac-2026/

- **2026-07-29** · 创业公司 · 前端 · ChipAgents Series A2，A 轮扩至 $134M（ChipAgents 新闻稿）
  B Capital 加入，追加 6000 万美元；自述部署超过 120 家，上半年 ARR 增 6 倍。
  https://chipagents.ai/newsroom/chipagents-series-a2-134m

- **2026-09-22** · 业界 · 前端 · Cadence RTL Generation Agent（Cadence 新闻稿）
  规格到 RTL 与早期 PPA；早期评估相对纯基础模型面积 -24%、功耗 -18%。Honda 在评估。计划 2026 Q4 早期提供。
  https://www.cadence.com/ko_KR/home/company/newsroom/press-releases/pr/2026/cadence-expands-chipstack-ai-super-agent-with-a-new-agent-for.html

- **2026-09-23** · 业界 · 跨层 · Synopsys 与 TSMC A14 及智能体合作（Synopsys 新闻稿）
  A14 认证流程启用智能体；3DIC Compiler 增加芯片粒 floorplan 协同优化。
  https://news.synopsys.com/2026-09-23-Synopsys-and-TSMC-Partner-to-Accelerate-AI-Systems-Innovation-with-Agentic-AI-and-Advanced-Design

- **2026-09-28** · 业界 · 跨层 · Synopsys Autopilot 与 AgentEngineer（Synopsys 新闻稿）
  长程领域智能体平台；超过 50 个评估，计划 2026 年底提供。Fujitsu 称 RTL 生成生产力 10–30%。
  https://news.synopsys.com/2026-09-28-Synopsys-Powers-Autonomous-Engineering-with-a-Broad-Portfolio-of-Long-Horizon-Agents-and-Autopilot-Platform

- **2026-10-01** · 业界 · EDA · TSMC AI Design Kit（ADK）（SemiWiki 对 TSMC OIP 2026 的报道）
  报道称 TSMC 推出给智能体用的工艺知识层。3–5 倍 / 最高 6 倍为 TSMC 口径转述，本轮未找到官方 PDF。
  https://semiwiki.com/semiconductor-manufacturers/tsmc/374098-tsmcs-ai-design-kit-bringing-agentic-ai-into-chip-design/

- **2026-10-09** · 论文 · 跨层 · From Prompts to Silicon (AI-HDL)（arXiv:2610.18891）
  教学竞赛两届扩到 604 人；两支冠军设计在 Sky130 上完成零违例签核并提交流片。
  https://arxiv.org/abs/2610.18891

- **2026-10-14** · 创业公司 · 跨层 · ChipAgents × NVIDIA Build-A-Chip-Bench（ChipAgents 新闻稿）
  规格到 GDS 基准；初评约 40% 过功能验证，约 10% 达到物理 PPA。
  https://chipagents.ai/newsroom/chipagents-nvidia-build-a-chip-bench

- **2026-10-21** · 业界 · 后端 · Siemens 3D IC 电热智能体（Siemens 新闻稿）
  7 工具 9 步电热协同；官方基准称工程投入最多 -95%，探索覆盖最多约 50 倍。
  https://news.siemens.com/de-ch/siemens-nvidia-gtc-3d-ic-thermal-ai/

- **2026-10-30** · 论文 · 前端 · RTL-Interact（arXiv:2610.41225（在投 ICLR 2027））
  把 RTL 生成、EDA 检查构造和工具反馈修订训进同一个领域模型。
  https://arxiv.org/abs/2610.41225

- **2026-11-09** · 论文 · 跨层 · ChipOmni（arXiv:2611.13020）
  通用 EDA 智能体；长程案例把自然语言 RISC-V 规格推到通过物理签核的微型 SoC GDSII。
  https://arxiv.org/abs/2611.13020

- **2026-11-10** · 论文 · 前端 · Toward Autonomous RTL… (Dr. RTL)（ASP-DAC'27 邀请论文 / arXiv:2611.13713）
  工具接地的 RTL 优化智能体；20 个真实设计上相对商业综合，WNS/TNS 平均 +21%/+17%，面积 -6%。
  https://arxiv.org/abs/2611.13713

- **2026-11-13** · 论文 · 后端 · OpenLayout（ICCAD 2026 / arXiv:2611.18517）
  开源智能体物理设计平台与 OpenLayout-Bench；论文称相对基线约 20% 准确率提升。
  https://arxiv.org/abs/2611.18517

- **2026-11-18** · 业界 · 前端 · OpenAI Advancing chip design with our models（OpenAI 官方研究页）
  Jalapeño 向量/矩阵单元面积约 -13%/-10% 且已硅验证；AMD 旗舰 GPU IP 最多约 14%，部分已进入近期流片；将与 Synopsys 合作训练 GPT-Synopsys。
  https://openai.com/index/advancing-chip-design-with-our-models/

- **2026-11-19** · 论文 · 前端 · LOCA: Verified Netlist-to-RTL Recovery（arXiv:2611.25324）
  按逻辑锥拆分并用形式工具守门，从网表恢复 RTL；21 对中恢复 17，整网表提示仅 6。
  https://arxiv.org/abs/2611.25324

- **2026-11** · 创业公司 · 跨层 · Ripple-65B Preview（Ricursive 官网）
  从零训练的 65B 开源权重模型，内部变体用于其芯片设计平台；官方称完整研发成本 3200 万美元。
  https://www.ricursive.com/ripple

- **2021** · 论文 · 后端 · AlphaChip（背景）（Nature / DeepMind 博客（2024 补遗））
  背景：强化学习宏块放置，官方称用于多代 TPU。存在复现与方法争议，引用须看补遗与评论。
  https://deepmind.google/discover/blog/how-alphachip-transformed-computer-chip-design

- **2023-11** · 论文 · EDA · ChipNeMo（背景）（arXiv:2311.00176）
  背景：NVIDIA 领域适应 LLM 做芯片设计助手，是后续 CVDP / ACE-RTL 的前序。
  https://arxiv.org/abs/2311.00176
