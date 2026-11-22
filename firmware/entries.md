# GPU / AI 芯片固件 条目清单

截至 2026-11-22（Asia/Shanghai）。用于追加，不必先改综述页。类型按文章 / 视频 / 官方文档 / 实践经验 / 论文 / 讲座标注。访问异常（403、登录墙等）在摘要末尾注明；404 的条目不收录。

## NVIDIA 驱动、GSP 与开源内核模块

- **2024-07-17** · 文章 · NVIDIA · NVIDIA Transitions Fully Towards Open-Source GPU Kernel Modules（NVIDIA Technical Blog）
  R560 起默认改为开源内核模块；Turing 及之后建议改用开源模块，Maxwell / Pascal / Volta 继续用专有模块；Grace Hopper 与 Blackwell 必须用开源模块。中文版同步发布。
  https://developer.nvidia.com/blog/nvidia-transitions-fully-towards-open-source-gpu-kernel-modules/
  https://developer.nvidia.cn/blog/nvidia-transitions-fully-towards-open-source-gpu-kernel-modules/

- **2025** · 官方文档 · NVIDIA · Chapter 44. GSP Firmware（Linux 驱动 README 580.95.05）
  定义 GSP：用驱动分发的 `gsp_*.bin` 卸载初始化和管理；支持的 GPU 上默认启用。用 `nvidia-smi -q` 或 `/proc/driver/nvidia/gpus/.../information` 查询，禁用显示 N/A。专有模块可用 `NVreg_EnableGpuFirmware=0` 关闭。
  https://download.nvidia.com/XFree86/Linux-x86_64/580.95.05/README/gsp.html

- **2025** · 官方文档 · NVIDIA · Chapter 45. Open Linux Kernel Modules（Linux 驱动 README 580.95.05）
  开源模块依赖 Turing 引入的 GSP，因此不能支持更早的 GPU；Blackwell 及之后只支持开源模块。两种模块互斥。保密计算、GDS、HMM 等只在开源模块上工作。初始化变慢可用 nvidia-persistenced 缓解。
  https://download.nvidia.com/XFree86/Linux-x86_64/580.95.05/README/kernel_open.html

- **2026** · 官方文档 · NVIDIA · Kernel Modules（Data Center Driver Installation Guide, latest）
  从 515 起提供专有与开源两种内核模块；560 起开源模块为默认。给出各发行版安装 `nvidia-open` 的包装命令。
  https://docs.nvidia.com/datacenter/tesla/driver-installation-guide/latest/kernel-modules.html

- **2026** · 官方文档 · NVIDIA · open-gpu-kernel-modules README（GitHub）
  编译出的开源模块必须与同一驱动版本的 GSP 固件和用户态组件一起使用；可用 `.run --no-kernel-modules` 只装用户态。也说明 Nouveau 会从源码中抽取固件镜像。
  https://github.com/NVIDIA/open-gpu-kernel-modules

- **2026** · 官方文档 · NVIDIA · GSP firmware cannot be disabled when open GPU kernel modules are installed（vGPU 已知问题 6353226）
  开源内核模块下即使设置 `NVreg_EnableGpuFirmware=0` 并重启，GSP 仍保持启用。官方标注 “Not a bug”。绕过需改回专有模块。
  https://docs.nvidia.com/vgpu/latest/known-issues/bug-6353226-gsp-firmware-cannot-be-disabled-with-open-kernel-modules.html

- **2026-11-11** · 官方文档 · NVIDIA · Driver Persistence / Persistence Daemon
  主推用户态 `nvidia-persistenced`，遗留 Persistence Mode 仍暂时支持但不再作为未来修复重点。守护进程需由发行版或管理员纳入 init 系统。
  https://docs.nvidia.com/deploy/driver-persistence/
  https://docs.nvidia.com/deploy/driver-persistence/latest/persistence-daemon.html

- **2026** · 官方文档 · NVIDIA · nvidia-smi
  官方手册：VBIOS、InfoROM（镜像版本与对象版本、checksum、退出码 14 表示 InfoROM 损坏）、GSP Firmware Version、行重映射查询项。
  https://docs.nvidia.com/deploy/nvidia-smi/index.html

## NVIDIA 内存 RAS、Xid 与整机固件

- **2026-11-20** · 官方文档 · NVIDIA · Xid Errors / Analyzing Xid Errors with the Xid Catalog
  Xid 是驱动打到内核日志的错误报告。Ampere 及更新 GPU 应下载官方 Catalog 表格，按上下文、组合规则和 Tenant 动作处理，而不是只背代号。
  https://docs.nvidia.com/deploy/xid-errors/
  https://docs.nvidia.com/deploy/xid-errors/latest/analyzing-xid-catalog.html

- **2026** · 官方文档 · NVIDIA · Row Remapping / User Visible Statistics / RMA Policy（GPU Memory Error Management）
  Ampere 起用硬件行重映射替代页退役，最多 512 次，需 GPU reset 才生效；Xid 63/64 对应 InfoROM 记录成功或失败；界面计数是 InfoROM 条目数。并给出 failure 标志与 RMA 门槛。
  https://docs.nvidia.com/deploy/a100-gpu-mem-error-mgmt/latest/row-remapping.html
  https://docs.nvidia.com/deploy/a100-gpu-mem-error-mgmt/latest/user-visible-statistics.html
  https://docs.nvidia.com/deploy/a100-gpu-mem-error-mgmt/latest/rma-policy-thresholds-for-row-remapping.html

- **2026** · 官方文档 · NVIDIA · About the nvfwupd Tool（DGX H100/H200 Firmware Update Guide）
  `nvfwupd` 的语法、目标类型（DGX、HGX、GB200 等）和通过 BMC Redfish 更新的示例。
  https://docs.nvidia.com/dgx/dgxh100-fw-update-guide/nvfwupd-reference.html

- **2026** · 官方文档 · NVIDIA · Viewing the Installed Firmware and Package Versions（DGX H100/H200）
  如何查看已安装的固件部件和包版本。
  https://docs.nvidia.com/dgx/dgxh100-fw-update-guide/view-fw-components-packages.html

- **2026** · 官方文档 · NVIDIA · Firmware Update Steps（DGX B200 Firmware Update Guide）
  升级前须停止包括 `nvidia-smi` 在内的 GPU 活动；部分 VBIOS 版本需要过渡包，不能跨数据结构直接跳版本；GPU 托盘部件并行更新，并需确认 BackgroundCopyStatus。
  https://docs.nvidia.com/dgx/dgxb200-fw-update-guide/sequence.html

- **2026** · 官方文档 · NVIDIA · DGX B300 Firmware Update Guide
  主板托盘与 GPU 托盘的可升级部件；BMC 冷复位、chassis power cycle、AC 下电三种激活方式分别用于不同部件。
  https://docs.nvidia.com/dgx/dgxb300-fw-update-guide/

- **2026** · 官方文档 · NVIDIA · NVIDIA Firmware Update Guide（multi-node NVLink systems）
  `nvfwupd` 覆盖 Grace Hopper / Blackwell / Vera Rubin、HGX/MGX、NVOS 交换机和电源架；输入为 NVIDIA 签名的 PLDM 包。
  https://docs.nvidia.com/multi-node-nvlink-systems/nvfupd-guide/introduction.html
  https://docs.nvidia.com/multi-node-nvlink-systems/nvfupd-guide/examples/nvos-switch-updates.html

- **2026** · 官方文档 · NVIDIA · DGX SuperPOD / BasePOD Update Guide：DGX Firmware Packages
  集群侧建议按发行说明、经 BCM 分发，工具仍是 `nvfwupd`。
  https://docs.nvidia.com/dgx-superpod/update-guide/latest/firmware.html

- **2026-09-21** · 官方文档 · NVIDIA · NVIDIA Fabric Manager User Guide（文档史 2.6）
  NVSwitch 系统的织物管理、SXid 与恢复流程；B200/B300 改由 DCGM + NVSDM 取交换错误；从驱动发行 620 起，第四代 NVSwitch 上的 Fabric Manager 开始独立版本号。
  https://docs.nvidia.com/datacenter/tesla/fabric-manager-user-guide/

## NVIDIA 保密计算、Operator 与安全公告

- **2026** · 官方文档 · NVIDIA · Secure AI Compatibility Matrix
  列出 GPU、VBIOS、CUDA 驱动和保密计算模式的已验证组合。说明 HGX Hopper FW 1.6.0 在 Protected PCIe 下可能掉总线，应使用 1.7.0；1.8.0 起不能回退到 1.7.1 或更早。
  https://www.nvidia.com/en-us/data-center/solutions/confidential-computing/secure-ai-compatibility-matrix/

- **2026** · 官方文档 · NVIDIA · Attesting the GPU / Attestation / Threat Model
  跑负载前须 attestation；成功后还要 `nvidia-smi conf-compute -srs 1` 才放行。部署前确认 persistenced 在跑、CC 状态为 ON。威胁模型指出保护取决于架构、模式、固件和驱动版本。
  https://docs.nvidia.com/confidential-computing/virtual-machines/deployment-guide/attest
  https://docs.nvidia.com/attestation/index.html
  https://docs.nvidia.com/attestation/attestation-client-tools-sdk/latest/gpu_and_switch_attestation.html
  https://docs.nvidia.com/confidential-computing/virtual-machines/threat-model

- **2026** · 官方文档 · NVIDIA · GPU Driver Upgrades（GPU Operator, latest）
  Operator 只管理容器化驱动，不管宿主机预装驱动，也不覆盖板级或整机固件。升级需排空 GPU 工作负载。
  https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-driver-upgrades.html

- **2026** · 官方文档 · NVIDIA · Supported Drivers and CUDA Toolkit Versions
  数据中心驱动与 CUDA Toolkit 的配套说明，规划升级时的版本锁依据之一。
  https://docs.nvidia.com/datacenter/tesla/drivers/supported-drivers-and-cuda-toolkit-versions.html

- **2026** · 官方文档 · NVIDIA · Product Security
  NVIDIA 驱动和软件安全公告的官方入口。
  https://www.nvidia.com/en-us/product-security/

- **2026** · 官方文档 · Google Cloud · Troubleshoot GPU VMs
  云上 GPU 虚拟机如何阅读 NVIDIA Xid，指向同一套官方概念。
  https://cloud.google.com/compute/docs/troubleshooting/troubleshooting-gpus

## AMD

- **2026** · 官方文档 · AMD · Firmware Updates（Instinct Customer Acceptance Guide）
  把固件分成 `/lib/firmware/amdgpu` 运行时微码和 OEM 发布的 VBIOS / PLDM；前者用发行版 `linux-firmware` 更新后重启，后者只用该机型合格的 OEM 包。给出升级后最小验证命令和集群一致性建议。
  https://instinct.docs.amd.com/projects/system-acceptance/en/latest/common/firmware-updates.html

- **2026** · 官方文档 · AMD · ROCm compatibility matrix
  按 Instinct 型号列出受支持的驱动版本和 PLDM bundle（固件）版本。规划升级时应对最新表，不要沿用旧集群的版本组合。
  https://rocm.docs.amd.com/en/latest/compatibility/compatibility-matrix.html

- **2026-07** · 官方文档 · AMD · AMD Instinct GPU Accelerator Firmware Tool User Guide v3.0（amdfwflash）
  用 amdfwflash 更新 IFWI / RMFW 的步骤和 FAQ。硬约束：同一 XGMI hive 不能混用不同固件版本。撰写时该 PDF 在 drivers.amd.com 返回 HTTP 200。
  https://drivers.amd.com/relnotes/amd_instinct_gpu_accelerator_firmware_tool_user_guide_v3_0.pdf

- **2026** · 官方文档 · AMD · Requirements to partition MI300X GPUs
  分区功能会钉死经验证的 VBIOS 版本；用 `amd-smi static --vbios` 查询。
  https://instinct.docs.amd.com/projects/amdgpu-docs/en/latest/gpu-partitioning/mi300x/requirements.html

- **2026** · 官方文档 · Linux 内核 · dGPU firmware flashing / AMDGPU Glossary
  Navi3x 及更新 dGPU 可通过 PSP 的 sysfs（`psp_vbflash`）刷 IFWI。术语表给出 PSP、SMU/SMC、MES、SDMA、IMU 等缩写的官方释义。
  https://www.kernel.org/dochtml/latest/gpu/amdgpu/flashing.html
  https://kernel.org/doc/Documentation/gpu/amdgpu/flashing.rst
  https://www.kernel.org/dochtml/latest/gpu/amdgpu/amdgpu-glossary.html

- **2026** · 官方文档 · linux-firmware
  发行版 GPU 运行时微码的上游仓库（含 nvidia/ 与 amdgpu/ 目录）。
  https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/about/
  https://gitlab.com/kernel-firmware/linux-firmware
  https://gitlab.com/kernel-firmware/linux-firmware/-/tree/main/nvidia

- **2026-11-04** · 实践经验 · AMD / Ubuntu · Launchpad #2173166（社区缺陷，非 AMD 官方规范）
  某次 `apt upgrade` 后，RX 9070 XT 的 DMUB（DCN 4.0.1）固件被 PSP 拒绝，内核报 DMCUB 错误。说明发行版 `linux-firmware` 与驱动/PSP 不匹配是真实故障模式。
  https://bugs.launchpad.net/ubuntu/+source/linux-firmware/+bug/2173166

## Intel Gaudi、昇腾、Neuron、TPU、Tenstorrent

- **2026** · 官方文档 · Intel Gaudi · Firmware Upgrade（文档 1.24.2，重定向后的稳定 URL）
  卸驱动、用 `hl-fw-loader` 更新 SPI、再装驱动并对 Support Matrix；升级中避免掉电。另含 eROM 与固件更新锁。
  https://docs.habana.ai/en/latest/Installation_Guide/Firmware_and_Platform_Components/Firmware_Upgrade.html

- **2026** · 官方文档 · Intel Gaudi · Firmware Update Tool
  `hl-fw-loader` 选项，包括指定设备、状态查询和恢复模式（安全设备上不可用）。
  https://docs.habana.ai/en/latest/Management_and_Monitoring/Embedded_System_Tools_Guide/Firmware_Update_Tool.html

- **2026** · 官方文档 · Intel Gaudi · Intel Gaudi Secure Firmware Flow
  查询安全固件状态、启用安全模式、更新 SVN；官方写明 SVN 更新后不能再降到旧固件。
  https://docs.habana.ai/en/latest/Management_and_Monitoring/Embedded_System_Tools_Guide/Intel_Gaudi_Secure_Firmware.html

- **2024–2026** · 官方文档 · 华为 · Atlas / 昇腾 NPU 驱动和固件升级指导、npu-smi 命令参考
  公开页面写明：首次安装「驱动 → 固件」，覆盖安装或升级「固件 → 驱动」。查询用 `npu-smi info -t board`，升级用 `npu-smi upgrade`。部分推理卡还需维护 MCU。本环境访问 support.huawei.com 返回 403 / 可能有 Cookie 墙，收录时请以能打开的官方页面为准。
  https://support.huawei.com/enterprise/zh/doc/EDOC1100441839
  https://support.huawei.com/enterprise/zh/doc/EDOC1100540364/a41d9380
  https://support.huawei.com/enterprise/zh/doc/EDOC1100493495/b5474dcf
  https://support.huawei.com/enterprise/en/doc/EDOC1100449166/4c56c4f6/before-the-upgrade

- **2025–2026** · 官方文档 · AWS · Neuron 文档中心 / 停止支持 neuron-device-version
  公开材料以 SDK、驱动和运行时为主，几乎没有面向用户的芯片固件刷写手册。Neuron 2.19 起 `neuron-monitor` 不再提供 `neuron-device-version`，改用 `instance_type` 与 `neuroncore_version`。
  https://awsdocs-neuron.readthedocs-hosted.com/
  https://awsdocs-neuron.readthedocs-hosted.com/en/latest/about-neuron/announcements/neuron2.x/announce-eos-neurondevice-version.html

- **2026** · 官方文档 · Google Cloud · TPU software versions
  用户选择的是 TPU software / runtime version，不是可自助刷写的固件包。固件由设施侧管理。
  https://docs.cloud.google.com/tpu/docs/runtimes

- **2026** · 官方文档 · Tenstorrent · Manual Installation / tt-flash / tt-system-firmware
  先装 TT-KMD，再用 `tt-flash` 烧公开的 firmware bundle，然后重启；Blackhole 要求固件 ≥ 18.3.0。用 `tt-smi` 验证。各 SDK 有自己的兼容矩阵。
  https://docs.tenstorrent.com/getting-started/manual-software-install.html
  https://docs.tenstorrent.com/index.html
  https://github.com/tenstorrent/tt-flash
  https://github.com/tenstorrent/tt-system-firmware
  https://github.com/tenstorrent/tt-kmd

## 会议、报道与安全研究

- **2023-12-01** · 讲座 / 文章 · LPC 2023 / Nouveau · A Nouveau graphics driver update（LWN，Jonathan Corbet 记述 Dave Airlie 报告）
  NVIDIA 把大量驱动逻辑搬进 GSP 固件并允许 Linux 发行版分发；ABI 不稳定、文档不足、镜像体积达数十 MB。Nouveau 在 6.7-rc1 左右首次并入 GSP 支持。
  https://lwn.net/Articles/953144/

- **2025-12** · 讲座 · LPC 2025 · Nova GPU Workshop（Alexandre Courbot、Danilo Krummrich、Dave Airlie、Joel Fernandes、John Hubbard 等）
  议题包括 GSP 固件 API 的设计与演化。会议在东京，页面时区 Asia/Tokyo（2025-12-11 至 13）。抓取时贡献页仍显示尚无讲稿或录像。
  https://lpc.events/event/19/contributions/2297/

- **2025** · 文章 · LWN · gpu: nova-core: process and prepare more firmwares to boot GSP
  上游 Nova（Rust）驱动为启动 GSP 准备更多固件的补丁讨论。
  https://lwn.net/Articles/1034788/

- **2024-01-16** · 论文 · GPU 安全 · Whispering Pixels: Exploiting Uninitialized Register Accesses in Modern GPUs（arXiv:2401.08881）
  讨论现代 GPU 上未初始化寄存器造成的数据残留与泄露。对象是 GPU 编程接口一侧，不是固件刷写流程。
  https://arxiv.org/abs/2401.08881

- **2026-06** · 论文 · GPU 安全协处理器 · A Canary in the Crypto Mine: Defeating Stack Protection in a GPU Secure Coprocessor（Jon Pry，Zenodo）
  分析数据中心加速器安全协处理器上的栈保护；文中以与 A100 同晶圆的 CMP 170HX 为例。只作为攻击面存在的公开研究入口，不转述利用步骤。
  https://zenodo.org/records/20916112
