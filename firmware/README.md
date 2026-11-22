# firmware

这个目录收集 GPU / AI 芯片固件的实践综述：它在软硬件栈里处在哪一层，各厂商怎么查询和升级，以及升级、兼容和排错时真正常见的约束。

## 怎么读

1. 先打开 [GPU与AI芯片固件实践综述.html](GPU与AI芯片固件实践综述.html)。这是正文，浏览器直接打开即可。页面风格与仓库里已有的「互联链路概念入门」综述一致。
2. 需要追加或检索原始链接时，看 [entries.md](entries.md)。每条带日期（能查到就写）、类型、厂商或主题、一两句摘要和 URL。访问异常会在对应条目里注明。

本目录不收录受版权保护的 PDF 或视频文件，只链到正式公开页。

## 范围

覆盖 NVIDIA（GSP、开源内核模块、VBIOS / InfoROM、DGX/HGX 整机固件、Fabric Manager、保密计算）、AMD Instinct（linux-firmware、IFWI / VBIOS、ROCm 矩阵）、Intel Gaudi、华为昇腾、AWS Neuron、Google TPU、Tenstorrent，以及集群升级和安全研究的公开材料。公开资料少的厂商，综述里会直接说明，不编造操作步骤。
