# 附录 A: 术语总表

本书用到的全部缩写与术语，按领域分组。每条给出全称、一句话定义，以及它在哪里展开。

---

## 1. 核心概念与威胁建模

### `TEE` (Trusted Execution Environment，可信执行环境)
一块由硬件强制保护、其内容不被更高特权软件读取的区域。**只有与证明结合才有意义。** → 模块 1 §1.3

### `TCB` (Trusted Computing Base，可信计算基)
一旦被攻破即致命的组件集合。VM TEE 用一个大 TCB（整个 guest OS）换来了零移植成本。→ 模块 1 §2.3

### `RoT` (Root of Trust，信任根)
制造时熔进硅片的不可导出秘密，加上不可变启动代码，并由厂商证书链背书。→ 模块 1 §3.1

### `CCC` (Confidential Computing Consortium，机密计算联盟)
Linux 基金会下的组织，其定义——在基于硬件的、**已证明的** TEE 中保护使用中的数据——是本领域的参考表述。→ 模块 1 §1.3

### Data in Use（使用中的数据）
计算期间位于 DRAM、寄存器或 GPU HBM 中的明文数据。磁盘与传输加密留下不保护的那个状态。→ 模块 1 §1.1

### 控制权 vs 机密性
VM TEE 的定义性不对称：hypervisor 保有对 guest 的完整资源控制权，同时被挡在其数据之外。→ 模块 1 §2.2

### Sealing（封存）
从信任根**并且**当前度量值派生加密密钥，使得被某个代码版本封存的数据无法被不同的代码解封。→ 模块 1 §3.5

### `TOCTOU` (Time of Check to Time of Use，检查时刻与使用时刻之差)
证明是一次快照，执行在此之后继续。靠不可变性缓解，而不是靠反复检查。→ 模块 3 §2.3

### `DMA` (Direct Memory Access，直接内存访问)
设备不经 CPU 访问内存。在 TEE 中，DMA 目标必须是共享（未加密）页，这正是 bounce buffer 存在的原因。→ 模块 2 §4.2.4

### `IOMMU` (输入输出内存管理单元)
翻译并限制设备内存访问的硬件。由 hypervisor 编程，因此它在 TEE 中**不是**一项机密性控制。

---

## 2. AMD SEV 与 SEV-SNP

### `SEV` (Secure Encrypted Virtualization)
AMD 第一代 VM 内存加密。**仅机密性**——无完整性，且 `VMEXIT` 时寄存器状态是明文。→ 模块 2 §1.1

### `SEV-ES` (SEV Encrypted State)
增加 `VMSA` 加密，使寄存器状态在世界切换中保持机密。**仍无内存完整性。** → 模块 2 §1.2

### `SEV-SNP` (SEV Secure Nested Paging)
经由 `RMP` 增加内存**完整性**，挫败重放、重映射与别名。**第一代能防住主动作恶 hypervisor 的实现。** → 模块 2 §1.3

### `C-bit`
guest 页表中标记页面已加密的一个物理地址位。置位 = 加密（与 TDX 的 SHARED 位相反）。→ 模块 2 §1.1

### `ASID` (Address Space Identifier，地址空间标识符)
标识一台机密 VM；内存控制器为每个 ASID 持有一把 AES 密钥。→ 模块 2 §1.1

### `RMP` (Reverse Map Table，反向映射表)
全系统唯一的表，每 4 KB 页一条表项，记录归属以及该页被允许承载的 guest 物理地址。**hypervisor 不可写。** SNP 完整性的核心。→ 模块 2 §1.3

### `PVALIDATE`
guest 用来接受某页作为私有内存的指令。**一次性且由硬件追踪**，从而封死重放攻击。→ 模块 2 §1.3

### `VMPL` (Virtual Machine Privilege Level，虚拟机特权级)
SNP guest **内部**的四个特权级（VMPL0 最高），使 paravisor 或 SVSM 得以提供 guest kernel 不应自行实现的服务。→ 模块 2 §1.3

### `SVSM` (Secure VM Service Module)
运行在 VMPL0 的可信软件，提供 guest 内服务——最重要的是一个 hypervisor 无法伪造的 vTPM。→ 模块 2 §1.3

### `VMSA` (VM Save Area)
跨世界切换保存 guest 寄存器状态的加密且带完整性校验的区域。**它的密文正是 CipherLeaks 所观测的东西。** → 模块 2 §1.2

### `GHCB` (Guest-Hypervisor Communication Block)
一个共享页，guest 通过它**刻意**披露 hypervisor 做 I/O 模拟所需的信息。→ 模块 2 §1.2

### `#VC` (VMM Communication Exception)
需要 hypervisor 参与的操作在 guest 内抛出的异常，使披露变成显式且由 guest 控制。→ 模块 2 §1.2

### `PSP` / `ASP` (Platform Security Processor / AMD Secure Processor)
片上协处理器，AMD 的信任根：持有密钥、计算 launch measurement、签名证明报告。→ 模块 2 §1.1

### `VCEK` (Versioned Chip Endorsement Key)
报告签名密钥，由芯片唯一秘密**与当前 TCB 版本**派生——**这正是固件更新会让缓存证书失效的原因。** → 模块 2 §1.4

### `VLEK` (Versioned Loaded Endorsement Key)
VCEK 的替代方案，绑定到云服务商而非单颗芯片；报告不再能给特定机器打指纹。→ 模块 3 §3.2

### `VMPCK` (VM Platform Communication Key)
启动时建立，仅 guest 与 AMD 安全处理器知晓；保护 guest 的证明请求通道。→ 模块 2 §1.4

### `ARK` / `ASK` (AMD Root Key / AMD SEV Signing Key)
锚定 VCEK/VLEK 签名的证书链。**ARK 是该钉死的信任锚。** → 模块 3 §3.2

### `KDS` (Key Distribution Service，密钥分发服务)
AMD 用于取回 VCEK 证书与 CA 链的服务。验证路径上的一个运行时依赖。→ 模块 3 §3.2

### `MA_REPORT_ID` (Migration Agent Report ID)
在启用迁移时，把证明报告与其迁移代理的报告关联起来。→ 模块 2 §1.4

---

## 3. Intel TDX 与 SGX

### `TDX` (Trust Domain Extensions)
Intel 的虚拟机级 TEE。通过一个新 CPU 模式与一个 Intel 签名的软件模块，达成与 SEV-SNP 相同的保证。→ 模块 2 §2

### `TD` (Trust Domain)
一个 TDX 机密 guest。

### `SEAM` (Secure Arbitration Mode，安全仲裁模式)
比 VMX root 更高特权的 CPU 模式，其中只运行 TDX Module。→ 模块 2 §2.1

### `TDX Module`
运行在 SEAM 模式下的 Intel 签名软件，仲裁进出 TD 的每一次转换。给 TCB 增加约 10⁵ 行代码，换来可打补丁性。→ 模块 2 §2.1

### `SEAMCALL` / `TDCALL`
hypervisor 与 guest 分别向 TDX Module 请求服务的接口。→ 模块 2 §2.1

### `SHARED bit`
guest 物理地址的最高位。置位 = 共享且明文；清零 = 私有、加密、带完整性保护。**与 AMD C-bit 约定相反。** → 模块 2 §2.2

### `Secure EPT` (Secure Extended Page Table)
TD 私有内存的二级页表，由 **TDX Module** 而非 hypervisor 拥有。TDX 版的 RMP 不变式。→ 模块 2 §2.2

### `MKTME` (Multi-Key Total Memory Encryption)
提供每 TD 一把密钥的内存加密引擎。→ 模块 2 §2.2

### `MRTD` (Measurement Register for Trust Domain)
构建期 launch measurement，在 TD 构造完成时 finalize，此后不可变。→ 模块 2 §2.3

### `RTMR0-3` (Runtime Measurement Registers，运行时度量寄存器)
四个只可扩展的寄存器，行为类似 TPM PCR，覆盖内核、initrd、rootfs 与工作负载。**TDX 相对 SEV-SNP 的架构优势。** → 模块 2 §2.3

### `TDREPORT`
用 CPU 本地密钥做 MAC 的本地证据。**不可远程验证**——必须转换成 Quote。→ 模块 2 §2.4

### `TD Quote`
由 TD Quoting Enclave 从 TDREPORT 产出的、可远程验证的非对称签名证明。→ 模块 2 §2.4

### `QE` (Quoting Enclave)
一个 SGX enclave，校验 TDREPORT 的 MAC 并用证明密钥重新签名。**SGX 存活下来的那份工作。** → 模块 2 §2.4

### `DCAP` (Data Center Attestation Primitives)
Intel 的库与 collateral 模型，使第三方验证无需每次都联系 Intel 在线服务。→ 模块 3 §3.3

### `PCS` / `PCCS` (配置证明服务 / 缓存服务)
Intel 验证 collateral 的来源，以及你在生产中运行的本地缓存。→ 模块 3 §3.3

### `PCK` (Provisioning Certification Key)
Intel 链中标识特定 CPU 封装与 TCB 层级的证书。→ 模块 3 §3.3

### `FMSPC` (Family-Model-Stepping-Platform-CustomSKU)
Intel 发布 TCB info 时所依据的平台标识符。→ 模块 3 §3.3

### `SGX` (Software Guard Extensions)
Intel 的进程级 enclave TEE。因内存限制与无法访问设备而在模型服务上出局；以 TDX Quoting Enclave 的形式存活。→ 模块 1 §4.1

### `EPC` (Enclave Page Cache)
支撑 SGX enclave 的受限加密内存区——正是这个约束把 SGX 挡在 LLM 权重之外。

---

## 4. 其他 TEE 架构

### `CCA` (Confidential Compute Architecture)
ARM 的虚拟机级 TEE，引入 Realm。→ 模块 2 §3

### `RMM` (Realm Management Monitor)
ARM CCA 中管理 Realm 的可信组件；TDX Module 的对应物。→ 模块 2 §3

### `GPT` (Granule Protection Table)
ARM CCA 的页归属表；AMD RMP 的对应物。→ 模块 2 §3

### `CoVE` (Confidential VM Extension)
RISC-V 的机密计算标准，其 TEE Security Manager 扮演 TDX Module 的角色。**作为一个开放 TEE 而值得关注。** → 模块 2 §3

---

## 5. 证明与证据

### `RATS` (Remote ATtestation procedureS，远程证明流程)
IETF 架构（RFC 9334），定义 Attester、Verifier、Relying Party、Endorser 与 Reference Value Provider。→ 模块 3 §1.1

### Attester / Verifier / Relying Party（证明方 / 验证方 / 依赖方）
产出证据的实体；评估证据的实体；据结果行动的实体。**谁来扮演验证方，是一份机密设计中后果最重大的决定。** → 模块 3 §1.3

### Endorser（背书方）
经由证书链担保硬件为真品的一方——AMD、Intel、NVIDIA。→ 模块 3 §1.1

### Reference Value Provider（参考值提供方）
发布度量值**应该**是什么的一方。通常是最薄弱的一环。→ 模块 3 §7

### Passport 模型
证明方取得一次证明结果，然后出示给各依赖方。GCP 采用的模型。→ 模块 3 §1.2

### Background-Check 模型
依赖方自己把证据转发给验证方。AWS Nitro 的模型；天然更新鲜。→ 模块 3 §1.2

### Launch Measurement（启动度量）
在执行之前计算的初始 guest 镜像与 vCPU 状态哈希，不可变。SNP 上是 `MEASUREMENT`，TDX 上是 `MRTD`。→ 模块 1 §3.2

### Runtime Measurement（运行时度量）
覆盖启动之后所加载内容的只可扩展寄存器。TDX 上是 `RTMR`；SEV-SNP 上是 vTPM PCR。→ 模块 1 §3.2

### `PCR` (Platform Configuration Register，平台配置寄存器)
只可扩展、不可设置的 TPM 寄存器：`PCR_new = H(PCR_old ‖ measurement)`。→ 模块 1 §3.2

### Event Log（事件日志）
被度量内容的有序记录，使验证方能够重放并**解释** PCR 值，而不是只能与一个不透明的黄金哈希比对。→ 模块 3 §3.1

### `REPORT_DATA` / `REPORTDATA`
证明报告中由调用方提供的 64 字节。承载 nonce 和/或 TLS 公钥哈希。**RA-TLS 的基础。** → 模块 1 §3.4

### `HOST_DATA`
由 **hypervisor** 在启动时提供的 32 字节。按构造即不可信——可用于关联，**绝不可用于安全判定**。→ 模块 2 §1.4

### `TCB_VERSION` / `SVN` (Security Version Number，安全版本号)
报告中的固件组件安全版本。**钉一个下限，绝不钉死具体值。** → 模块 3 §4.2

### `EAT` (Entity Attestation Token，实体证明令牌)
RFC 9711 的标准证明声明格式。这正是 GCP 令牌里有 `eat_nonce`、`dbgstat`、`hwmodel` 这些声明名的原因。→ 模块 3 §3.4

### `CoRIM` (Concise Reference Integrity Manifest)
发布参考值的标准格式。→ 模块 3 §3.4

### `CoSWID` (Concise Software Identification)
软件标识标签，把参考值绑定到可识别的软件。→ 模块 3 §3.4

### `RA-TLS` (Remote Attestation TLS)
把公钥哈希嵌入报告的调用方数据，从而将证明报告绑定到一条 TLS 会话上。**给客户端端到端保证的唯一干净做法。** → 模块 3 §6

### `dm-verity`
在只读文件系统上建立的 Merkle 树，其根哈希被度量，读取时逐块校验。**它让"被度量的 rootfs"在运行期仍然有意义。** → 模块 3 §2.2

### Fail-Closed / Fail-Open
验证方不可达或 collateral 陈旧时，是阻断运行还是忽略。**靠意外来选就等于选了 fail-open。** → 模块 3 §3.3

---

## 6. 密码学与密钥管理

### `AES-XTS`
用于内存加密的保长、以地址为 tweak 的**确定性**模式。**它的确定性就是密文侧信道。** → 模块 1 §3.3

### `AES-GCM`
带认证、非确定性的模式，用于 GPU PCIe 路径——在那里 nonce 与 tag 的膨胀负担得起。→ 模块 4 §2.2

### `KEK` / `DEK` (密钥加密密钥 / 数据加密密钥)
信封加密：大块数据用 DEK 加密，DEK 再用 KEK 封装。**轮转 KEK 很便宜；轮转 DEK 意味着重新加密模型。** → 模块 6 §3.2

### `SKR` (Secure Key Release，安全密钥释放)
仅向证明满足策略的环境释放密钥材料。→ 模块 3 §5

### `HPKE` (Hybrid Public Key Encryption，混合公钥加密)
RFC 9180 的公钥加密，可用于客户端把载荷加密给一把已证明的公钥。→ 模块 6 §5.2

### `SPDM` (Security Protocol and Data Model)
DMTF 标准，用于与设备做双向认证的密钥交换，建立 CPU TEE 到 GPU 的会话。→ 模块 4 §2.3

### `IDE` (Integrity and Data Encryption)
给 PCIe 总线流量做加密与完整性保护的链路层标准。→ 模块 4 §2.3

### `TDISP` (TEE Device Interface Security Protocol)
把设备接口直接指派进 TEE 的 PCIe 标准。广泛实现后将消除 bounce buffer。→ 模块 4 §6.2

### Ciphertext Side Channel（密文侧信道）
通过观测确定性密文来推断明文变化。**架构性的，不是 bug。** CipherLeaks 及其后继工作。→ 模块 2 §5.1

---

## 7. 机密 GPU

### `CC mode`（机密计算模式）
NVIDIA GPU 的设备状态：`OFF`、`ON`（保护生效，profiling 禁用）、`DEVTOOLS`（**不是安全模式**）。→ 模块 4 §2.1

### 受保护区 / 非保护区
HBM 的划分：受保护区存放权重、KV cache 与激活值；非保护区是密文 bounce 的着陆区。→ 模块 4 §2.2

### Bounce Buffer
一块共享、宿主可见的暂存缓冲区，加密数据经由它跨越 PCIe 边界。**CC 开销的主要来源。** → 模块 4 §2.2

### `GSP` (GPU System Processor)
GPU 的片上微控制器，其固件作为 GPU 证明的一部分被度量。→ 模块 4 §2.2

### `RIM` (Reference Integrity Manifest，参考完整性清单)
NVIDIA 针对某个固件与驱动版本签名发布的期望度量值。**GPU 侧对参考值问题的回答。** → 模块 4 §3.1

### `NRAS` (NVIDIA Remote Attestation Service)
NVIDIA 托管的验证方。方便；但增加依赖并暴露你的机群情况。本地验证方是替代方案。→ 模块 4 §3.1

### `nvtrust`
NVIDIA 的开源 GPU 证明与验证工具链。→ 模块 4 实验

### Composite Attestation（复合证明）
针对**被绑定的** CPU TEE、GPU 与工作负载镜像证据的**单一**策略决策。分开验证证明不了多少。→ 模块 4 §3.2

### `SPT` (Single GPU Passthrough，单卡直通)
Hopper 世代的机密 GPU 模式：每台机密 VM 一块 GPU，无受保护互联。在 GCP 令牌中以 `cc_feature` 出现。→ 模块 4 §4.1

### Protected NVLink（受保护 NVLink）
Blackwell 世代硬件加密的 GPU 间互联，支持 1/2/4/8 卡机密分组，从而使 TEE 内张量并行成为可能。→ 模块 4 §4.2

### `SEV-TIO` (SEV Trusted I/O)
AMD 的可信设备指派，把 Instinct 加速器与 SEV-SNP 配对。→ 模块 4 §6.1

### `nvidia-persistenced`
保持驱动加载的守护进程，避免每次进程退出都触发 CC 会话重协商与重新证明。**CC 模式下必需。** → 模块 4 §2.4

### GPU Ready State（GPU 就绪状态）
在证明成功前拒绝计算的硬件门（`nvidia-smi conf-compute -srs 1`）。**按设计把证明放在冷启动关键路径上。** → 模块 4 §2.4

---

## 8. Google Cloud

### GKE Hypercluster
Google Cloud 专为大规模 AI 训练与机密 LLM 推理构建的超算级 Kubernetes 架构。深度整合机密 GPU 加速节点池、高吞吐存储（GCS FUSE / Hyperdisk ML）与 AI 编排引擎。→ 模块 5 §2 与 模块 6 §2

### AI Hypercomputer
Google Cloud 的一体化 AI 超算架构，结合了针对性能优化的硬件体系、开源软件框架（LeaderWorkerSet、Kueue、JobSet）与高吞吐存储系统。→ 模块 5 §2

### Confidential VM
基础产品。`--confidential-compute-type=SEV | SEV_SNP | TDX`。**纯 SEV 没有内存完整性。** → 模块 5 §1

### Confidential GKE Nodes
其 VM 为 Confidential VM 的节点池。`--confidential-node-type=sev|sev_snp|tdx`。**不覆盖 Google 运营的控制面。** → 模块 5 §2

### Confidential Containers（CoCo，机密容器）
上游 Kubernetes 架构，将 Pod 运行在独立的硬件 MicroVM TEE 中（如基于 TDX/SEV-SNP 的 Kata Containers），将节点 Kubelet 与宿主机操作系统排除在 Pod 的 TCB 之外。→ 模块 5 §6 与 模块 6 §2.1

### Confidential Space
一个加固的、被度量的镜像，只跑一个容器并产出证明令牌。分离工作负载作者、工作负载运营方与数据协作方——**运营方拥有完整项目管理员权限，却仍然读不到数据。** → 模块 5 §3

### Container Launcher（容器启动器）
Confidential Space 中启动工作负载并经 `/run/container_launcher/teeserver.sock` 提供证明令牌的组件。→ 模块 5 §4.1

### Google Cloud Attestation
GCP 的验证方，签发 OIDC JWT 证明令牌。注意：对一个包含 Google 的威胁模型，**由 Google 签名的结果是被怀疑方作出的声明**。→ 模块 5 §4.1

### `dbgstat`
表示调试状态的令牌声明：`disabled-since-boot` 或 `enabled`。**最重要的那一条必须断言的声明。** → 模块 5 §3.3

### `hwmodel`
标识硬件的令牌声明：`GCP_AMD_SEV`、`GCP_AMD_SEV_ES`、`GCP_INTEL_TDX` 或 `GCP_SHIELDED_VM`。**断言你期望的那个——Shielded VM 不是 TEE。** → 模块 5 §4.2

### `swname` / `swversion`
`CONFIDENTIAL_SPACE` 或 `GCE`，以及镜像版本。用于区分 Confidential Space 镜像与裸机密 VM。→ 模块 5 §4.2

### `submods`
令牌的嵌套声明结构：`submods.container`（镜像摘要、签名、args、env）、`submods.gce`（project、zone、instance）、`submods.confidential_space`（支持属性、监控）、`submods.nvidia_gpu`（`cc_mode`、逐 GPU 型号、驱动、VBIOS）。→ 模块 3 §5.2

### `eat_nonce`
令牌中由调用方提供的新鲜性值。**用它；只靠 `exp` 等于接受一个重放窗口。** → 模块 3 §4.1

### `WIF` (Workload Identity Federation，工作负载身份联合)
把证明令牌换成 Google 凭据的机制，其针对令牌声明的 CEL **属性条件**充当释放策略。→ 模块 5 §4.2

### `CMEK` (Customer-Managed Encryption Keys，客户管理的加密密钥)
你在 Cloud KMS 中控制的加密密钥。比 Google 托管更好，但**仍然不是由证明门控的**。→ 模块 5 §1.3

### `EKM` (External Key Manager，外部密钥管理器)
由 Google **之外**的密钥管理器支撑的 Cloud KMS。对模型提供方的权重 KEK 来说，这是那个**质上不同**的选项。→ 模块 5 §4.3

### Shielded VM
安全启动、vTPM 与完整性监控。**不是机密计算**——内存没有被加密。→ 模块 5 §5.3

### Binary Authorization
阻止未签名或未证明镜像被部署的 GKE 准入控制。它阻止**部署**；证明策略阻止**密钥释放**。→ 模块 5 §5.1

### `VPC-SC` (VPC Service Controls)
针对数据外泄的边界控制——**少数几个 TEE 之外的控制确实有用的场合**，因为威胁就是工作负载本身。→ 模块 5 §5.3

---

## 9. 与 LLM 服务及 AI 编排的交叉

### `LeaderWorkerSet` (`LWS`)
用于分布式多节点 AI 工作负载的开源 Kubernetes API 控制器。将 1 个 Leader Pod 与 N 个 Worker Pod 协同为一个统一拓扑工作组，支持双向证明与张量并行权重分发。→ 模块 6 §2 与 §3.2

### `Dynamic Workload Scheduler` (`DWS`) / `flex-start`
Google Cloud 加速器算力调度机制。`flex-start` 模式支持以确定性的执行时间窗口对多节点 GPU 算力进行群调度与容量锁定，消除冷启动排队抖动。→ 模块 5 §2 与 模块 6 §4.2

### `Kueue`
云原生 Kubernetes 作业队列管理器，管理多租户 GPU 节点池上的批量与推理作业流、公平共享与优先级抢占。→ 模块 6 §4.2 与 模块 7 §4.3

### `KV cache`（键值缓存）
缓存的 attention key 与 value。**是模型被告知的一切的高保真编码**——请以对待 prompt 本身的敏感度对待它。→ 模块 6 §6.1

### Prefix Caching（前缀缓存）
跨请求复用共享 prompt 前缀已算好的 KV 块。**跨租户时这是一台针对其他租户 prompt 内容的时序预言机**，而内存加密帮不上忙。→ 模块 6 §6.2

### Continuous Batching（连续批处理）
迭代级调度，把多个租户的请求混进同一次前向。**它们之间的隔离是软件的，不是硬件的。** → 模块 6 §7.2

### Disaggregated Prefill/Decode（分离式预填充/解码）
把两个阶段拆到不同机器上，并在其间搬运 KV cache。在机密设计中需要双向证明的加密传输。→ 模块 6 §6.3

### `TTFT` / `TPOT` (首 token 时延 / 每输出 token 时延)
分别由 prefill 和 decode 主导的延迟指标。**TPOT 受机密计算影响最小。** → 模块 7 §1.3

### Arithmetic Intensity（算术强度）
每传输字节对应的 FLOPs。CC 只对分母征税，因此相对开销按 $1/I$ 缩放——**这正是它随模型规模、批大小与序列长度增大而缩小的原因。** → 模块 4 §5.2

### Attested Ingress（已证明入口）
在 TEE **内部**终结 TLS 或解密载荷。**托管 L7 负载均衡器持有明文 prompt，会作废整个保证。** → 模块 6 §5

### `MaaS` / `3P MaaS` (Model-as-a-Service / 第三方 MaaS)
在你的基础设施上托管别人的模型。**驱动整本书的那个三方信任冲突。** → 索引页

### `PCC` (Private Cloud Compute)
Apple 的机密推理架构：无状态计算、可强制执行的保证、无特权运行时访问、不可定向、可验证透明性。**该对标的标尺。** → 模块 7 §6.2

---

返回[课程索引](index.md)，或继续阅读[附录 B: 一手资料阅读清单](appendix_reading_list.md)。
