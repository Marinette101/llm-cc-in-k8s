# 附录 B: 一手资料阅读清单

本书有保质期。架构是稳定的；可用性矩阵、GPU 世代与攻击文献不是。本附录存在的意义，是让这份笔记里的任何一条论断都能回到源头核验。

每一条都标注了该资料真正好用在哪。标注 **(已核验)** 的链接是写这份笔记时抓取或确认过的；没有链接的条目按标题与出版方引用，因为我没有核验到稳定 URL——**请按标题搜索，不要相信猜出来的链接。**

---

## 1. 标准与架构

### `RFC 9334` —— 远程证明流程 (RATS) 架构
**(已核验)** <https://www.rfc-editor.org/info/rfc9334/> · <https://datatracker.ietf.org/doc/html/rfc9334>

整个领域所用的词汇：Attester、Verifier、Relying Party、Endorser、Reference Value Provider，以及 passport 与 background-check 模型。**短、好读，值得完整读一遍。** → 模块 3 §1

### `RFC 9711` —— 实体证明令牌 (EAT)
**(已核验)** <https://datatracker.ietf.org/doc/rfc9711/>

GCP 证明令牌在精神上所遵循的标准声明格式——这正是令牌里有 `eat_nonce`、`dbgstat`、`hwmodel`、`swname` 而不是厂商私有字段名的原因。→ 模块 3 §3.4

### `RFC 9180` —— 混合公钥加密 (HPKE)
客户端把载荷加密给一把已证明公钥所用的原语。→ 模块 6 §5.2

### IETF RATS 工作组 —— CoRIM 与 CoSWID 草案
发布参考值与软件身份的标准格式。**关注它们**；它们是参考值问题的最终答案。通过 RATS 工作组的 datatracker 页面跟踪。→ 模块 3 §3.4

### DMTF —— 安全协议与数据模型 (SPDM) 规范
机密 GPU 会话建立所依赖的设备认证与密钥交换协议。由 DMTF 发布。→ 模块 4 §2.3

### PCI-SIG —— TDISP 与 IDE 规范
TEE 设备接口安全协议与完整性数据加密。最终会消除 bounce buffer 的标准化终点。需会员资格；下面的 DMTF 与厂商白皮书是可获取的摘要。→ 模块 4 §6.2

### 机密计算联盟 (Confidential Computing Consortium)
<https://confidentialcomputing.io/>

其定义锚定了模块 1 §1.3 的那个 Linux 基金会组织。当你需要一份厂商中立的材料交给干系人时，它发布的白皮书是合理的选择。

---

## 2. AMD SEV-SNP

### AMD —— *SEV Secure Nested Paging Firmware ABI Specification*
证明报告结构、guest policy 位、`TCB_VERSION` 与固件接口的权威参考。**当你需要确切知道某个报告字段是什么意思时，就查这份文档。** 发布于 AMD 开发者站点的 SEV 文档区。

### AMD —— *SEV-SNP Platform Attestation Using VirTEE/SEV*（出版号 58217）
**(已核验)** <https://www.amd.com/content/dam/amd/en/documents/developer/58217-epyc-9004-ug-platform-attestation-using-virtee-snp.pdf>

用开源工具取得并验证证明报告的实操走查。最接近厂商出的动手指南。→ 模块 2 实验

### AMD —— *SEV-SNP Attestation: Establishing Trust in Guests*（Jeremy Powell）
**(已核验)** <https://www.amd.com/content/dam/amd/en/documents/developer/lss-snp-attestation.pdf>

对证明流程、VCEK 派生与证书链的清晰讲解。在啃 ABI 规范之前用来建立直觉很合适。

### `virtee/snpguest`
**(已核验)** <https://github.com/virtee/snpguest>

模块 2 实验所用的 CLI：请求报告、从 AMD KDS 取回 CA 与 VCEK、验证证书链。**读它的源码是理解"验证到底包含什么"的高效途径。**

### `google/go-sev-guest`
**(已核验)** <https://github.com/google/go-sev-guest>

封装 `/dev/sev-guest` 并提供证明验证的 Go 库。如果你要自建验证方——而模块 3 §1.3 主张你应该——这就是该读的参考实现。

---

## 3. Intel TDX

### Intel —— *Trust Domain Extensions (TDX) Module Architecture Specification* 及 TDX 白皮书系列
SEAM 模式、TDX Module、secure EPT、`MRTD`/`RTMR` 以及 TDREPORT 到 Quote 路径的权威描述。发布于 Intel 的 TDX 文档门户。

### Intel —— *Device Attestation Model in Confidential Computing Environment*（白皮书）
**(已核验)** <https://cdrdv2-public.intel.com/783079/whitepaper-device-attestation-model-in-confidential-computing-environment-v0.6.4.pdf>

设备（加速器）证明如何与 CPU TEE 证明组合。与模块 4 §3.2 的复合证明论证直接相关。

### Intel Trust Authority —— 证明模式与 EAT Profile
**(已核验)** <https://docs.trustauthority.intel.com/main/articles/articles/ita/concept-patterns.html> · <https://portal.trustauthority.intel.com/eat_profile.html>

对 passport 与 background-check 在一个生产服务中如何体现的清晰阐述，以及一份具体的 EAT profile。**即使你从不使用该产品，也值得一读。**

### Intel —— DCAP 文档与 `SGXDataCenterAttestationPrimitives`
collateral 模型、PCCS 缓存与验证库。**在决定你的 fail-closed / fail-open 姿态之前先读它。** → 模块 3 §3.3

---

## 4. 机密 GPU

### NVIDIA —— *Confidential Computing on NVIDIA H100 GPUs for Secure and Trustworthy AI*
**(已核验)** <https://developer.nvidia.com/blog/confidential-computing-on-h100-gpus-for-secure-and-trustworthy-ai/>

CC 模式、受保护内存、bounce buffer 与证明流程的经典入门。模块 4 从这里开始。

### NVIDIA —— *Hardware-Rooted AI Security That Won't Slow You Down*
**(已核验)** <https://developer.nvidia.com/blog/hardware-rooted-ai-security-that-wont-slow-you-down>

讲 Blackwell 世代的多卡机密计算，包括 NVLink 加密与 1/2/4/8 卡分配——**那个决定"哪些模型根本能不能服务"的变化。** → 模块 4 §4.2

### *Creating the First Confidential GPUs* —— Communications of the ACM
**(已核验)** <https://cacm.acm.org/practice/creating-the-first-confidential-gpus/>

由造出它的工程师写的设计回顾。**是关于"这套架构为什么长这样"而非"它做什么"的最佳解释。**

### Rob Nertney —— *Remote Attestation for NVIDIA Hopper and Blackwell GPUs, CPUs, and Beyond*（OC3 2025）
**(已核验)** <https://cdn.prod.website-files.com/63c54a346e01f30e726f97cf/67f00a27564271b2f87c4988_Rob%20Nertney%20OC3%202025%20-%20Nvidia%20Blackwell.pdf>

从 NVIDIA 视角讲 CPU + GPU 复合证明，包括 RIM 与"NRAS vs 本地验证方"的抉择。→ 模块 4 §3

### `NVIDIA/nvtrust`
**(已核验)** <https://github.com/NVIDIA/nvtrust>

模块 4 实验所用的证明 SDK 与本地验证方。**它的 issue 列表对真实世界的失败模式信息量异常大**——度量值不匹配、ready-state 问题、CPU/GPU 配对约束。

---

## 5. Google Cloud

### Confidential VM —— 概览、受支持配置与发布说明
**(已核验)** <https://docs.cloud.google.com/confidential-computing/confidential-vm/docs/confidential-vm-overview>

受支持配置那一页是该加书签的：机型、CC 技术与 zone，**变化频繁到模块 5 里任何内容都不该盖过它。**

### Confidential GKE Nodes
**(已核验)** <https://docs.cloud.google.com/kubernetes-engine/docs/how-to/confidential-gke-nodes>

启用 flag、限制，以及那句明确表述——Confidential GKE Nodes 不改变应用于集群控制面的安全措施。**模块 5 §2.3 就建立在这句话上。**

### 带 GPU 的 Confidential GKE Nodes
**(已核验)** <https://docs.cloud.google.com/kubernetes-engine/docs/how-to/gpus-confidential-nodes>

GPU 约束：机型、一块 H100、TDX、无 GPU 共享，以及 GKE 版本要求。→ 模块 4 §4.1

### Confidential Space —— 概览与证明令牌声明
**(已核验)** <https://docs.cloud.google.com/confidential-computing/confidential-space/docs/confidential-space-overview> · <https://docs.cloud.google.com/confidential-computing/confidential-space/docs/reference/token-claims>

**令牌声明参考页是本书中最有用的一个 GCP 页面。** 模块 3 §5.2 策略清单里的每一条声明都来自它，包括 `submods.nvidia_gpu` 结构。

### `salrashid123/confidential_space`
**(已核验)** <https://github.com/salrashid123/confidential_space>

大量 Confidential Space 配合基于证明的密钥释放的实做示例。**比官方快速上手更详细，也对边界情况更诚实。**

### Google Cloud —— *How Confidential Computing Lays the Foundation for Trusted AI*
**(已核验)** <https://cloud.google.com/blog/products/identity-security/how-confidential-computing-lays-the-foundation-for-trusted-ai/>

Google 自己对机密 AI 故事的框定。对理解 Google 内部听众会期待的词汇很有用，**建议与模块 7 §5.2（框定在哪里跑到机制前面）对照阅读。**

---

## 6. 对比架构

### Apple —— *Private Cloud Compute: A New Frontier for AI Privacy in the Cloud*
**(已核验)** <https://security.apple.com/blog/private-cloud-compute/>

**本清单中最有价值的一份文档。** 那五条要求——无状态计算、可强制执行的保证、无特权运行时访问、不可定向、可验证透明性——是关于"机密推理应当意味着什么"迄今最严谨的公开陈述。**在设计任何东西之前先读它。** → 模块 7 §6.2

### Apple —— *Expanding Private Cloud Compute*
**(已核验)** <https://security.apple.com/blog/expanding-pcc/>

PCC 扩展到 Google Cloud，使用 Intel TDX、NVIDIA Confidential Computing 与 Google 的 Titan 芯片。**在组件层面上就是模块 6 所描述的架构**——而且明确演示了模型提供方可以在别人的基础设施上运行、同时保有软件控制权与独立验证。→ 模块 7 §6.3

### Apple —— *Stateless Computation and Enforceable Guarantees*
**(已核验)** <https://security.apple.com/documentation/private-cloud-compute/statelessandenforcable>

前两条要求背后的详细技术文档。

### AWS —— *Attest an Amazon EC2 instance with AMD SEV-SNP*
**(已核验)** <https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/snp-attestation.html>

作为 GCP 模型的对照很有用，而且 **AWS 用 KMS 条件键做基于证明的密钥释放，是业内最干净的做法**——即便你从不部署在 AWS，也值得理解。→ 模块 7 §6.2

### Azure Confidential Containers 与 CNCF Confidential Containers 项目
模块 1 §4.4 与模块 5 §6 论证的、对 Kubernetes 而言架构上更好的机密性单位——Pod 沙箱模型。通过 CNCF Confidential Containers 项目与微软的 AKS 机密容器文档跟踪。

---

## 7. 攻击文献

**至少读第一篇。** 模型提供方的安全团队知道这批工作，而能准确讨论它，正是一个可信的安全论断与一句营销话术之间的分野。

### CIPHERLEAKS —— *Breaking Constant-time Cryptography on AMD SEV via the Ciphertext Side Channel*（USENIX Security 2021）
**(已核验)** <https://www.usenix.org/conference/usenixsecurity21/presentation/li-mengyuan>

密文侧信道的奠基性结果。确立了确定性内存加密会泄露"值变化"信息，以及**常量时间代码在这里不是防御**。→ 模块 2 §5.1

### CipherH —— *Automated Detection of Ciphertext Side-channel Vulnerabilities*（USENIX Security 2023）
**(已核验)** <https://www.usenix.org/system/files/usenixsecurity23-deng-sen.pdf>

对该漏洞类别的自动化发现。如果你需要评估自己的代码，这篇相关。

### Cipherfix、CipherGuard、Zebrafix —— 缓解方案
**(已核验)** <https://arxiv.org/pdf/2210.13124> · <https://arxiv.org/html/2502.13401> · <https://arxiv.org/pdf/2502.09139>

打破明文到密文确定性的软件与编译器缓解。**读一篇就能理解为什么缓解代价高昂，以及为什么"干脆别把密钥长期放在 guest DRAM 里"往往是更好的答案。**

### TDXdown —— *Single-Stepping and Instruction Counting Attacks against Intel TDX*（CCS 2024）
**(已核验)** <https://dl.acm.org/doi/10.1145/3658644.3690230>

击穿了 TDX 内建的单步执行缓解。模块 2 §5.3 的参考，也是"TDX Module 的可打补丁性既是优势也是 TCB 变动来源"的例证。

### Heracles —— *Chosen Plaintext Attack on AMD SEV-SNP*（CCS 2025）
**(已核验)** <https://heracles-attack.github.io/Heracles-CCS2025.pdf>

针对 SEV-SNP 的当前技术水平。**仔细读它的威胁模型章节**——那些假设决定了它是否适用于你的部署。

### Heckler、WeSee、CacheWarp
针对 SEV-SNP 的中断注入与缓存操纵攻击。按名字搜索，各有项目页。**结构性教训——凡是 hypervisor 保有控制权的东西都是攻击面——比单个结果更重要。** → 模块 2 §5.2

---

## 8. 性能研究

### *Confidential Computing on NVIDIA Hopper GPUs: A Performance Benchmark Study*
**(已核验)** <https://arxiv.org/pdf/2409.03992>

被引最多的 H100 CC 基准。确立了开销由 CPU–GPU PCIe 传输主导，并随模型规模、批大小与序列长度增大而缩小。**模块 4 §5.2 的经验基础。**

### *Performance of Confidential Computing GPUs*
**(已核验)** <https://arxiv.org/pdf/2505.16501>

跨多种负载类型的更宽泛处理。

### *Confidential LLM Inference: Performance and Cost Across CPU and GPU TEEs*
**(已核验)** <https://www.arxiv.org/pdf/2509.18886>

把 CPU TEE 代价与 GPU TEE 代价分离开——**正是模块 7 §1.1 坚持的那种分解**——并包含一个成本模型。

### *Benchmarking Confidential GPU Inference on NVIDIA H100 under Intel TDX*
**(已核验)** <https://arxiv.org/html/2607.19353v1>

最接近模块 6 参考架构的那个配置，也是 15–25% 容量预留启发式的来源。

---

## 怎么用这份清单

- **在做设计？** 先 Apple PCC，再 RFC 9334，再 Confidential Space 令牌声明参考。
- **在调试证明？** 字段语义查 AMD ABI 规范或 Intel TDX 规范，然后读 `snpguest` / `go-sev-guest` / `nvtrust` 源码。
- **在为一个安全论断辩护？** 第 7 节的攻击文献。**主动说出限定条件，才是让论断可信的东西。**
- **在做容量规划？** 第 8 节的性能研究，然后一旦有了自己的测量数据就立刻替换掉它们。
- **在核对这份笔记里的东西是否还成立？** 第 5 节的 Google Cloud 文档，它以**月**为尺度变化。

---

返回[课程索引](index.md)或[附录 A: 术语总表](appendix_glossary_and_terminology.md)。
