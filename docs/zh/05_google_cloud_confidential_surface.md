# 模块 5: Google Cloud 机密计算产品面

模块 1 到 4 构建了机制。本模块讲的是 Google Cloud 实际卖的是什么、每个产品暴露了其中哪些机制，以及那个决定你架构走向的问题——**一个第三方模型服务负载，该基于 Google 那两个相当不同的机密产品中的哪一个来构建。** 它们不是彼此的变体。它们体现的是对"谁在信任边界之内"这个问题的**不同回答**，而选那个方便的会让你丢掉一个之后很难补回来的安全性质。

本模块涵盖 **Confidential VM 机型族**、**Confidential GKE Nodes 及其未覆盖之处**、**Confidential Space 及其三角色分离**、**GCP 的证明与密钥释放产品面**、**周边配套产品**，以及**面向推理的正面决策对比**。

!!! warning "产品面变化很快"
    本领域的机型可用性、地域覆盖与 GPU 支持以**月**为尺度变化。本模块中每一条产品论断在进入设计文档之前，都应当对照当前的 Google Cloud 文档重新核验。**架构**是稳定的；**可用性矩阵**不是。

---

## 第 1 部分: Confidential VM

### 1.1 产品矩阵

Confidential VM 是基础层——本模块其余一切都建立在它之上。

```mermaid
flowchart TD
    A["Confidential VM<br>--confidential-compute-type"] --> B["SEV<br>仅内存加密<br>无完整性"]
    A --> C["SEV_SNP<br>+ RMP 完整性<br>+ 内容丰富的证明报告"]
    A --> D["TDX<br>+ secure EPT 完整性<br>+ 原生 RTMR 运行时度量"]

    B --> B1["N2D、C2D、C3D、C4D 族<br>✅ 支持热迁移"]
    C --> C1["N2D 配 AMD Milan<br>❌ maintenance-policy=TERMINATE"]
    D --> D1["C3 族，以及 A3<br>机密 GPU 路径<br>❌ maintenance-policy=TERMINATE"]
```

| | **SEV** | **SEV-SNP** | **TDX** |
| :--- | :--- | :--- | :--- |
| 内存机密性 | 有 | 有 | 有 |
| 内存完整性 | **无** | 有 | 有 |
| 原生运行时度量 | 无 | 无——需要 vTPM | **有**（`RTMR0-3`） |
| 热迁移 | 支持 | 不支持 | 不支持 |
| 机型族（请核验时效） | N2D、C2D、C3D、C4D | N2D（AMD Milan） | C3，以及机密 GPU 的 A3 路径 |
| 机密 GPU 支持 | 无 | 见 §2.2 | **有**——文档化的 GPU 路径 |

**应当驱动你选择的那一行是内存完整性。** 纯 SEV 给的是没有完整性的机密性，而模块 1 §3.3 已确立：这对一个主动作恶的 hypervisor 是不够的——重放与重映射攻击可行。SEV 是可用性最广、最便宜的选项，而对一个威胁模型包含攻击者 A2 或 A3 的负载来说，它是**错的**那个。**如果一份设计文档只写"Confidential VM"而不说是哪种技术，它就还没做那个要紧的决定。**

### 1.2 实际约束

三条会塑造你容量规划的：

1. **SEV-SNP 与 TDX 需要 `--maintenance-policy=TERMINATE`。** 没有热迁移（模块 2 §4.2.1）。主机维护会停掉实例。对一个冷启动要好几分钟的推理节点，这是一个真实的可用性输入，不是脚注。
2. **地域可用性很窄。** 机密产品——尤其是 TDX 与机密 GPU——只在少数地域提供。这与数据驻留要求相互作用得很糟：一个需要数据留在特定地域的客户，可能会发现那里根本不提供机密选项，而由此产生的对话很尴尬，因为这**两个**要求都是安全要求。
3. **不只是可用性，还有容量。** A3 机密容量受限。**预留很重要。**

### 1.3 VM 周边的存储与密钥

Confidential VM 保护内存，它对磁盘只字未提，周边的存储故事必须刻意去搭：

| 关注点 | 机制 | 备注 |
| :--- | :--- | :--- |
| 启动盘与数据盘 | Google 托管加密、CMEK，或 Confidential Hyperdisk | CMEK 意味着**你**在 Cloud KMS 里控制 KEK——更好，但仍由 Google 运营 |
| 静态模型权重 | 应用层信封加密（模块 3 §5） | **不要靠磁盘加密来做这件事。** 解密必须由证明门控，而磁盘加密不是 |
| 秘密 | 基于证明的密钥释放，而非 Secret Manager IAM | Secret Manager 按**身份**释放；你需要按**度量值**释放 |

中间那一行最要紧，也最常做错。用 CMEK 把磁盘上的权重加密，保护的是"有人偷走磁盘"的场景。它**不**保护你不受云运营商侵害，因为运营商的平台能读那块盘、也握着通往密钥的路径。只有**基于证明门控的释放**才做得到。

---

## 第 2 部分: Confidential GKE Nodes

### 2.1 它是什么

Confidential GKE Nodes 本质上就是"把我这个节点池的 VM 跑成 Confidential VM"。kubelet、容器运行时和你的 Pod 全都跑在 TEE 内部。

集群级启用（Autopilot 或 Standard）：

```bash
gcloud container clusters create-auto CLUSTER_NAME \
  --location=LOCATION \
  --confidential-node-type=CONFIDENTIAL_COMPUTE_TECHNOLOGY   # sev | sev_snp | tdx
```

或节点池级启用（仅 Standard）：

```bash
gcloud container node-pools create NODE_POOL_NAME \
  --cluster=CLUSTER_NAME \
  --location=LOCATION \
  --machine-type=MACHINE_TYPE \
  --node-locations=ZONE1,ZONE2 \
  --confidential-node-type=CONFIDENTIAL_COMPUTE_TECHNOLOGY
```

**集群级启用不可逆。** 你无法在已有集群上把它关掉。节点池级启用是灵活的那条路，也是你在迭代期间想要的。

### 2.2 GPU 配置

模块 4 §4.1 里的机密 GPU 路径，表达成一个节点池：

```bash
gcloud container node-pools create cc-gpu-pool \
  --cluster=CLUSTER_NAME \
  --location=LOCATION \
  --node-locations=ZONE \
  --confidential-node-type=tdx \
  --machine-type=a3-highgpu-1g \
  --accelerator=type=nvidia-h100-80gb,count=1,gpu-driver-version=latest
```

这些约束值得重述，因为它们是"你能服务什么"的硬边界：

- 每节点一块 H100 80 GB，`a3-highgpu-1g`。
- Intel TDX。
- **无 GPU 共享**——没有 time-sharing，没有 multi-instance GPU。
- 有最低 GKE 版本要求，且随"手动还是自动安装驱动"、"是否使用 ComputeClasses 或 flex-start"而不同。

### 2.3 Confidential GKE Nodes **没有**覆盖什么

本节是这个模块存在的理由，也是从中最该带走的东西。

```mermaid
flowchart TD
    subgraph OUT ["❌ 在你的 TEE **之外**"]
        CP["Kubernetes 控制面<br>API server、scheduler、etcd<br>由 Google 运营"]
        CP2["Google 的文档说得很明确：<br>Confidential GKE Nodes 不改变<br>应用于集群控制面的安全措施。"]
    end

    subgraph IN ["🔒 在你的 TEE **之内** —— 全部"]
        K["kubelet"]
        CR["容器运行时"]
        P1["你的推理 Pod"]
        P2["⚠️ 控制面调度到这里的**任何其他** Pod"]
        DS["⚠️ 任何 DaemonSet，包括调试与日志 agent"]
    end

    CP -->|"想调度什么就调度什么"| K
    K --> P1
    K --> P2
    K --> DS
```

仔细跟着推论走：

1. 控制面在你的 TEE 之外，由 Google 运营。
2. kubelet 在你的 TEE **之内**，并服从控制面。
3. 因此，任何拥有足够控制面权限的人，都可以让任意代码运行在**你的信任边界之内**。

这包括 `kubectl exec` 进你的推理 Pod、在节点上调度一个特权调试 Pod，或者加一个读取进程内存的 DaemonSet。而内存加密在整个过程中都在**完美地**履行职责——它正在保护那个攻击者的代码不被 hypervisor 看见。

**诚实的表述**：Confidential GKE Nodes 把 hypervisor 和物理层移出了你的 TCB（攻击者 A2 与 A4，以及 A3 的很大一部分）。它**没有**移出 Kubernetes 控制面，而在 GKE 上控制面由 Google 运营。**如果你威胁模型里的头号攻击者是"Google"，那么单靠 Confidential GKE Nodes 并不能完整地应对它。**

这不是对产品的批评——这是对它用途的正确解读。但任何声称"防护云内部人员"的设计文档都必须把这一点写明，因为模型提供方的安全团队会找到它。

### 2.4 其他值得知道的限制

- 与 sole-tenant 节点不兼容。
- 不支持 Windows 节点池。
- Local SSD 仅支持临时存储用途。
- Node auto-provisioning 支持 SEV 与 SEV-SNP，但**不支持 TDX**——这很要紧，因为 TDX 正是机密 GPU 路径。
- 在无法热迁移的场景下，维护事件会造成中断。

---

## 第 3 部分: Confidential Space

### 3.1 一个回答不同问题的不同产品

Confidential Space 不是"带增强的 Confidential VM"。它是为**多方信任问题**量身打造的答案——也就是说，它正是为模块 1 §5.1 的场景而建的产品。

核心思想：一个加固的、由 Google 发布的、被度量的镜像，它的全部工作就是启动**一个**容器，并产出一份描述该容器的证明令牌。**没有 SSH。没有交互式访问。不能随意调度工作负载。** 该镜像基于 Container-Optimized OS，并为这一单一用途做了加固。

### 3.2 三个角色

这个分离才是这个产品真正的贡献，而且它直接映射到课程索引里的三方图。

```mermaid
flowchart TD
    subgraph WA ["✍️ 工作负载作者"]
        WA1["编写并发布<br>容器镜像"]
        WA2["❌ 无法访问数据<br>❌ 无法访问结果<br>❌ 无法控制谁能访问它们"]
    end

    subgraph WO ["🔧 工作负载运营方"]
        WO1["运行工作负载；<br>拥有完整的项目级<br>管理员权限"]
        WO2["❌ 无法访问数据<br>❌ 无法修改工作负载代码<br>或执行环境"]
    end

    subgraph DC ["🔐 数据协作方"]
        DC1["拥有受保护资源<br>并制定释放策略"]
        DC2["❌ 无法访问彼此的数据<br>❌ 无法修改工作负载代码"]
    end

    WA1 -->|"签名镜像 + 摘要"| CS["🔒 Confidential Space<br>加固镜像 + launcher<br>只跑**一个**容器<br>产出证明令牌"]
    WO1 -->|"预置与运营"| CS
    DC1 -->|"针对证明声明的<br>KMS 策略"| CS
    CS -->|"证明令牌"| KMS["Cloud KMS / EKM<br>仅在策略通过时<br>释放密钥"]
```

需要内化的那句话是：**工作负载运营方拥有完整的项目级管理员权限，却仍然读不到数据。** 这正是 Confidential GKE Nodes 给不了你的性质，也恰恰是一份 3P MaaS 设计所需要的性质。把它映射到真实各方：

| Confidential Space 角色 | 3P MaaS 中的一方 |
| :--- | :--- |
| 工作负载作者 | 构建推理镜像的一方（模型提供方，或一次联合审计的构建） |
| 工作负载运营方 | Google / 运行该服务的平台团队 |
| 数据协作方（权重） | 模型提供方，持有权重 KEK |
| 数据协作方（prompt） | 终端客户，如果 prompt 密钥也由证明门控 |

### 3.3 DEBUG 与生产镜像

Google 同时发布 Confidential Space 镜像的 debug 与生产两个变体。差别体现在证明令牌里：

| | 生产镜像 | Debug 镜像 |
| :--- | :--- | :--- |
| `dbgstat` 声明 | `disabled-since-boot` | `enabled` |
| 交互式访问 | 无 | 可用于排障 |
| 适合真实数据 | 是 | **否** |

这就是模块 3 里那条 `dbgstat` 检查的具体落地。一个不断言 `dbgstat == "disabled-since-boot"` 的依赖方，会**欣然**把生产密钥释放给一个运营商可以检视工作负载的 debug 镜像。这是一行策略遗漏，后果却是全局性的——也是整份释放策略里价值最高的单项检查。

镜像版本还携带 `support_attributes`（`STABLE`、`LATEST`、`USABLE`、`EXPERIMENTAL`）。生产释放策略应当钉死可接受取值，而不是照单全收——理由和你钉死 TCB 下限是同一个。

### 3.4 可观测性张力

Confidential Space 的强项——什么都进不去，除了工作负载**刻意**输出的东西什么都出不来——同时也是它的运维代价。你不能 SSH 进去。你不能挂调试器。如果容器在启动时崩了，你手上的线索非常少。

Google 提供了可选开启的机制来输出日志与内存监控，而证明令牌会反映它们是否被启用（Confidential Space 子模块下有一个 `monitoring_enabled` 结构）。**"这件事被反映在令牌里"才是那个重要的设计细节**：数据协作方可以写一条策略，拒绝向一个开启了内存监控的实例释放密钥。这才是正确的形状——**可观测性决策变成了一个可协商、可证明的性质，而不是运营商单方面拨动的开关。**

这个张力是真实的，而且没有干净的化解办法：你每增加一项诊断能力，就多开了一条通往信任边界之外的通道。模块 7 §3 讲的就是如何在这个约束下工作。

### 3.5 对推理而言的局限

Confidential Space 每个 VM 实例跑一个容器。它不是 Kubernetes。没有 HPA，没有服务网格，没有滚动发布原语，没有 Pod 级调度。

对一个推理**服务**——它需要自动扩缩容、负载均衡、滚动升级和基于健康度的替换——你实际上在重建 Kubernetes 给你的相当一部分东西。这是 §6 的核心权衡，也是一项**真实的工程成本**，而不是一句形式化的告诫。

---

## 第 4 部分: GCP 上的证明与密钥释放

### 4.1 令牌路径

```mermaid
flowchart TD
    A["🔒 Confidential Space 中的工作负载"] -->|"1 请求令牌"| L["容器 launcher<br>unix socket：<br>/run/container_launcher/teeserver.sock"]
    L -->|"2 硬件证据<br>SNP report / TD quote + GPU 证据"| GCA["Google Cloud Attestation<br>（验证方）"]
    GCA -->|"3 签名的 OIDC JWT<br>带 EAT 风格声明"| A
    A -->|"4 出示令牌"| STS["Security Token Service<br>Workload Identity Federation"]
    STS -->|"5 针对声明<br>求值属性条件"| STS
    STS -->|"6 联合凭据"| A
    A -->|"7 用该凭据解密"| KMS["Cloud KMS / Cloud HSM / Cloud EKM"]
```

这就是模块 3 §1.2 的 passport 模型。注意第 2 步：**Google Cloud Attestation 就是那个验证方。** 模块 3 §1.3 解释过为什么这是一个设计决策而非细节——对一个威胁模型包含 Google 的模型提供方来说，**由 Google 签名的证明结果，是那个被怀疑方作出的声明**。缓解手段见下面 §4.3。

### 4.2 把策略写成 Workload Identity Federation 属性条件

策略是一条针对令牌声明求值的 CEL 表达式：

```bash
gcloud iam workload-identity-pools providers create-oidc PROVIDER_ID \
  --location=global \
  --workload-identity-pool=POOL_ID \
  --issuer-uri="https://confidentialcomputing.googleapis.com" \
  --allowed-audiences="https://sts.googleapis.com" \
  --attribute-mapping="google.subject=assertion.sub" \
  --attribute-condition="
    assertion.swname == 'CONFIDENTIAL_SPACE'
    && assertion.dbgstat == 'disabled-since-boot'
    && assertion.hwmodel == 'GCP_INTEL_TDX'
    && 'sha256:APPROVED_DIGEST' in assertion.submods.container.image_digest
    && assertion.submods.nvidia_gpu.cc_mode == 'ON'
  "
```

这里每一个子句都是在前面某个模块里挣来的：

| 子句 | 在哪确立 |
| :--- | :--- |
| `swname == 'CONFIDENTIAL_SPACE'` | §3.1——这是 Confidential Space 镜像，不是裸 GCE VM |
| `dbgstat == 'disabled-since-boot'` | §3.3 与模块 3 §5.2——最重要的单项检查 |
| `hwmodel == 'GCP_INTEL_TDX'` | 模块 2 §4.1——你要的是完整性，而不只是 SEV 的机密性 |
| `image_digest` 白名单 | 模块 3 §2——链条抵达真正的工作负载 |
| `nvidia_gpu.cc_mode == 'ON'` | 模块 4 §2.1 与 §3.2——没有它权重就落进明文 HBM |

在可能的情况下，优先用 `submods.container.image_signatures[].key_id` 而不是摘要白名单。摘要白名单每次发版都得改；签名检查能跨版本存活，并且表达的是**意图**（"由这把密钥签名的镜像"），而不是一份实例枚举。

### 4.3 密钥存储的抉择

P3（互相可验证）这条性质就是在这里赢下或输掉的。

| 选项 | 密钥材料在哪 | 谁能改释放策略 | 保证 |
| :--- | :--- | :--- | :--- |
| **Cloud KMS** | Google 基础设施，Google 托管 HSM 支撑 | 项目内的 Google Cloud IAM 主体 | 策略级 |
| **Cloud HSM** | FIPS 认证 HSM，Google 运营 | 同上 | 策略级，密钥托管更好 |
| **Cloud EKM** | **你自己的**外部密钥管理器，在 Google 之外 | 你，在你自己的系统里 | **扎根于硬件与组织** |

对模型提供方的权重 KEK 来说，EKM 是那个**质上不同**的选项。用 Cloud KMS 时，"Google 要拿到明文密钥得做什么？"的答案里包含"改一条 IAM 策略"——一次内部操作。用 EKM 时，密钥从不存在于 Google 的基础设施中，释放决定由模型提供方自己运营的系统作出。

有两条需要诚实说明的注意事项：

- EKM 在冷启动路径上增加了对外部密钥管理器的延迟与可用性依赖。
- **即便用了 EKM，模型提供方也必须自己验证证明证据**才能拿到完整性质。如果外部密钥管理器只是单纯信任一个 Google 签名的令牌，那么验证方**仍然是** Google。要做对，意味着提供方的密钥管理器要拿底层硬件证据去对 AMD/Intel/NVIDIA 的根做校验。

---

## 第 5 部分: 配套阵容

### 5.1 镜像签名与 Binary Authorization

机密计算把安全问题从隔离转移到了供应链（模块 1 §4.3）。真正要紧的工具链：

- **Sigstore / cosign** 对容器镜像的签名，会以 `submods.container.image_signatures[]` 的形式出现在证明令牌里，带 `key_id` 与 `signature_algorithm`。这就是让释放策略能说"由模型提供方签名"而不是"摘要等于 X"的东西。
- **Binary Authorization** —— 一个 GKE 准入控制器，阻止未签名或未证明的镜像被部署。注意这个区别：Binary Authorization 阻止的是**部署**；证明策略阻止的是**密钥释放**。两者互补，而**只有后者由 Google 控制面之外的东西来执行**。

### 5.2 gVisor 不是"弱一点的 TEE"

这个混淆几乎出现在每一次设计评审里，所以值得说直白点。

| | **gVisor / 沙箱** | **机密计算** |
| :--- | :--- | :--- |
| 保护 | **宿主**不受**工作负载**侵害 | **工作负载**不受**宿主**侵害 |
| 攻击者 | 恶意或被攻陷的容器 | 恶意 hypervisor、云内部人员、物理攻击者 |
| 执行方 | 软件（用户态内核） | 硬件 |
| 对 3P MaaS 设计有用吗？ | 有——但理由不同 | 有——理由就是本书讲的这个 |

它们是**正交的**，不是替代关系。事实上两者对 3P MaaS 都相关：机密计算保护模型提供方的权重不受平台侵害，而沙箱保护平台不受模型提供方的容器侵害。一份完整的设计很可能两者都用——而**把这一点说出来**，恰恰表明威胁模型在**两个方向**上都被想过。

### 5.3 Shielded VM、Workload Identity 与其余

- **Shielded VM** 提供安全启动、vTPM 与完整性监控。它**不是**机密计算——内存没有被加密，hypervisor 读得到。注意 `GCP_SHIELDED_VM` 是 `hwmodel` 声明的一个可能取值，这正是为什么释放策略必须**断言它期望的 `hwmodel`**，而不能只检查令牌能否验签。
- **Workload Identity** 把一个 Kubernetes 服务账号绑到一个 Google 服务账号。它认证的是工作负载**是谁**，而不是它**在跑什么代码**。它不是证明的替代品，而一份用 Workload Identity 去门控权重访问的设计，是在需要**度量控制**的地方放了一个**身份控制**。
- **VPC Service Controls** 在网络边界约束数据外泄。它与模块 6 §8 相关——那里的问题是如何阻止一个 TEE 内的工作负载外泄 prompt，而这恰是少数几个 TEE **之外**的控制确实有用的场合，因为**威胁就是工作负载本身**。

---

## 第 6 部分: 推理场景下 Confidential Space vs Confidential GKE

### 6.1 正面对比

| 维度 | **Confidential GKE Nodes** | **Confidential Space** |
| :--- | :--- | :--- |
| 机密性单位 | 节点 | 运行一个容器的 VM 实例 |
| kubelet 在 TCB 内 | **是** | 不适用——没有 Kubernetes |
| 控制面能否向边界内注入代码 | **能** | **不能** |
| 运营方能否访问数据 | 能，只要集群权限足够 | **不能，按设计** |
| 证明身份 | 节点镜像；工作负载身份需额外工作 | 容器镜像摘要，原生在令牌里 |
| 开箱即得的证明令牌 | 无 | **有** |
| GPU 支持 | 有——`a3-highgpu-1g`，一块 H100，TDX | 有——以 `submods.nvidia_gpu` 声明呈现 |
| 自动扩缩容、滚动更新、服务网格 | **有——原生** | 无；你自己建 |
| 多容器 Pod、sidecar | 支持 | 一个容器 |
| 运维熟悉度 | 高 | 低 |
| 契合 3P MaaS 信任模型 | 部分 | **契合——它就是为此设计的** |

### 6.2 建议，并把代价说清楚

**对一个安全论断是"云运营商读不到模型权重或客户 prompt"的负载，Confidential Space 是架构上正确的原语，而单靠 Confidential GKE Nodes 不充分。**

理由就是 §2.3：在 Confidential GKE Nodes 上，由 Google 运营的控制面可以把代码调度进你的信任边界。仅这一条事实就削弱了那个头号论断，而且**任何 RBAC 配置都修不好它**——因为 RBAC 正是由那个你试图排除的控制面执行的。

这条建议的代价是真实的，不应被淡化。选择 Confidential Space 意味着对系统的机密部分放弃 HPA、滚动发布、服务网格、sidecar，以及整套 GKE 运维工具链。对一个需要随流量伸缩的推理服务来说，那是一笔可观的工程投入。

### 6.3 通常胜出的那个混合方案

实践中可行的架构是"两者皆非/两者皆是"——一个**分离平面**设计：

```mermaid
flowchart TD
    subgraph NORM ["普通 GKE —— 机密数据永不触及这里"]
        A["入口、路由、限流"]
        B["认证、配额、计费、计量"]
        C["机群控制面：<br>扩缩容决策、健康、发布"]
        D["指标与非内容日志"]
    end

    subgraph CONF ["🔒 Confidential Space 实例 —— 明文唯一存在的地方"]
        E["推理工作负载<br>已证明，单容器"]
        F["权重仅在基于证明的<br>密钥释放之后解密"]
        G["TLS 或载荷解密<br>在**内部**终结"]
    end

    A -->|"仅加密载荷——<br>绝不传明文 prompt"| E
    C -->|"生命周期指令，<br>**不是**数据访问"| E
    E -->|"加密响应、<br>不含内容的指标"| D
```

让它成立的规则是：**普通 GKE 平面可以编排机密平面，但绝不能看到明文。** 入口转发加密载荷；控制面启停实例；指标携带计数与延迟但不含内容。机密平面很小、可审计、只做一件事。

这就是模块 6 会完整展开的架构，包括最难的那部分——**prompt 如何从客户手里到达机密平面，而不在边界处被解密。**

---

## Lab: 部署到 Confidential Space 并打破策略

**目标**：把一个工作负载部署到 Confidential Space，向它释放一个由证明令牌门控的 Cloud KMS 秘密，然后改镜像里的一个字节，眼看着释放失败。这是模块 3 的实验在真实产品面上的具体化。

**成本**：一台小机密 VM 加几次 KMS 操作——远低于一美元。**状态**：命令遵循 Google Cloud 文档；`gcloud` 语法随 CLI 版本而变，请用 `--help` 与当前 Confidential Space 文档核对。

### 第 1 步 —— 构建一个会展示自身令牌的工作负载

```dockerfile
FROM python:3.12-slim
RUN pip install --no-cache-dir requests-unixsocket google-auth
COPY main.py /main.py
CMD ["python", "/main.py"]
```

`main.py` 应当从 launcher socket 取回令牌、打印解码后的声明、经 STS 换取凭据，并尝试一次 Cloud KMS 解密。推送到 Artifact Registry 并记下摘要。

### 第 2 步 —— 建立密钥与策略

```bash
# 一把持有测试秘密的 KMS 密钥
gcloud kms keyrings create cc-lab --location=global
gcloud kms keys create weights-kek --location=global --keyring=cc-lab --purpose=encryption

# 一个由证明声明门控的工作负载身份池
gcloud iam workload-identity-pools create cc-lab-pool --location=global

gcloud iam workload-identity-pools providers create-oidc cc-lab-provider \
  --location=global \
  --workload-identity-pool=cc-lab-pool \
  --issuer-uri="https://confidentialcomputing.googleapis.com" \
  --allowed-audiences="https://sts.googleapis.com" \
  --attribute-mapping="google.subject=assertion.sub" \
  --attribute-condition="assertion.swname == 'CONFIDENTIAL_SPACE' \
    && assertion.dbgstat == 'disabled-since-boot' \
    && 'sha256:YOUR_DIGEST' in assertion.submods.container.image_digest"
```

把由此产生的主体授予该密钥上的 `roles/cloudkms.cryptoKeyDecrypter`。

### 第 3 步 —— 在 Confidential Space 镜像上运行它

```bash
gcloud compute instances create cc-space-lab \
  --confidential-compute-type=SEV_SNP \
  --machine-type=n2d-standard-2 \
  --min-cpu-platform="AMD Milan" \
  --maintenance-policy=TERMINATE \
  --zone=us-central1-a \
  --image-project=confidential-space-images \
  --image-family=confidential-space \
  --metadata="^~^tee-image-reference=REGION-docker.pkg.dev/PROJECT/REPO/IMAGE@sha256:DIGEST" \
  --scopes=cloud-platform \
  --service-account=YOUR_SA@PROJECT.iam.gserviceaccount.com
```

在 Cloud Logging 里查看工作负载输出。解密应当成功。

### 第 4 步 —— 用三种方式打破它

每一种都先**预测**失败，再去跑。

1. **改镜像。** 在 `main.py` 里加一行注释、重建、推送，用新摘要重新部署但**不**更新属性条件。令牌换取会在摘要子句上失败。
2. **切换到 debug 镜像族。** 用 debug 版 Confidential Space 镜像重新部署。观察令牌里 `dbgstat` 变成 `enabled`，条件失败。**然后把 `dbgstat` 子句删掉再部署一次。** 解密现在**成功**了——在一个运营商拥有交互式访问权的镜像上。请在这个结果上多待一会儿；这是本模块最有教育意义的一次失败。
3. **在一台普通机密 VM 上试。** 把同一个容器跑在一台普通机密 VM 而非 Confidential Space 镜像上。注意 `swname` 现在是 `GCE` 而不是 `CONFIDENTIAL_SPACE`，条件失败。这就是那个子句并非冗余的原因。

### 第 5 步 —— 与 Confidential GKE Nodes 对照

把同一个容器部署到一个 Confidential GKE 节点池：

```bash
gcloud container node-pools create cc-lab-pool \
  --cluster=YOUR_CLUSTER --location=LOCATION \
  --confidential-node-type=sev_snp --machine-type=n2d-standard-4
```

现在试着从 Pod 内部取得一个等价的工作负载身份证明令牌。你会发现**没有** launcher socket 的开箱等价物——然后，用你**普通的**集群凭据，运行：

```bash
kubectl exec -it POD_NAME -- /bin/sh
```

你现在就在信任边界之内，站在一台内存被硬件加密的节点上，读取工作负载内存里的任何东西。**这就是 §2.3，用一条命令演示出来。** 没有任何东西坏掉；产品完全按设计工作。它只是不防你以为它防的那个攻击者。

### 第 6 步 —— 清理

```bash
gcloud compute instances delete cc-space-lab --zone=us-central1-a --quiet
gcloud container node-pools delete cc-lab-pool --cluster=YOUR_CLUSTER --location=LOCATION --quiet
```

---

## 总结: GCP 机密计算产品面

| 问题 | 答案 | 后果 |
| :--- | :--- | :--- |
| 选哪种 Confidential VM 技术？ | SEV-SNP 或 TDX，绝不用纯 SEV | 纯 SEV 没有内存完整性 |
| GPU 用哪种技术？ | TDX，`a3-highgpu-1g`，一块 H100 | 把可服务模型规模封在 80 GB |
| Confidential GKE 覆盖控制面吗？ | **不覆盖**——Google 文档明确这么说 | Google 运营的控制面能向你的 TEE 注入代码 |
| Confidential Space 运营方能读数据吗？ | **不能**——即便拥有完整项目管理员权限 | 这正是 3P MaaS 需要的性质 |
| 什么门控密钥释放？ | 针对令牌声明的 CEL 属性条件，随后是 IAM | 每个子句都对应模块 2–4 中的一个机制 |
| 哪些声明不可妥协？ | `dbgstat`、`hwmodel`、`image_digest`、`nvidia_gpu.cc_mode` | 仅遗漏 `dbgstat` 就足以作废整个保证 |
| 权重 KEK 该放哪？ | Cloud EKM，由提供方自己的验证方校验 | Cloud KMS 会把保证降级为一条 IAM 策略 |
| gVisor 是 TEE 的替代品吗？ | **不是**——方向相反，两者都可能有用 | 它保护宿主不受负载侵害，而非反过来 |
| Confidential Space 还是 Confidential GKE？ | 数据平面用 Confidential Space，外围用普通 GKE | 机密部分要放弃 HPA、滚动更新与 sidecar |

你现在拥有了全部组件：硬件 TEE、证明、机密 GPU，以及暴露它们的平台原语。剩下的是设计本身——权重怎么进来、TLS 在哪里终结、KV cache 对你的威胁模型做了什么，以及一个拿到 host root 的内部人员对成型系统**仍然**能做什么。那就是**模块 6: 在 GKE 上设计机密 LLM 服务 (`06_designing_confidential_llm_serving_on_gke.md`)**。
