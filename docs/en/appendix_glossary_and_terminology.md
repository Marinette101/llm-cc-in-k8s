# Appendix A: Master Glossary

Every acronym and term used in this book, grouped by domain. Each entry gives the expansion, a one-line definition, and where it is developed.

---

## 1. Core Concepts and Threat Modeling

### `TEE` (Trusted Execution Environment)
A hardware-enforced region whose contents are protected from more privileged software. Meaningful only when combined with attestation. → Module 1, §1.3

### `TCB` (Trusted Computing Base)
The set of components whose compromise breaks the security guarantee. VM TEEs trade a large TCB (the entire guest OS) for zero porting cost. → Module 1, §2.3

### `RoT` (Root of Trust)
An unextractable secret fused into silicon at manufacture, plus immutable boot code, endorsed by a vendor certificate chain. → Module 1, §3.1

### `CCC` (Confidential Computing Consortium)
The Linux Foundation body whose definition — protection of data in use, in a hardware-based, *attested* TEE — is the field's reference wording. → Module 1, §1.3

### Data in Use
Plaintext data in DRAM, registers, or GPU HBM during computation. The state that disk and transport encryption leave unprotected. → Module 1, §1.1

### Control versus Confidentiality
The defining asymmetry of VM TEEs: the hypervisor keeps full resource control over the guest while being locked out of its data. → Module 1, §2.2

### Sealing
Deriving an encryption key from the root of trust *and* the current measurement, so data sealed by one code version cannot be unsealed by different code. → Module 1, §3.5

### `TOCTOU` (Time of Check to Time of Use)
Attestation is a snapshot; execution continues afterward. Mitigated by immutability, not by re-checking. → Module 3, §2.3

### `DMA` (Direct Memory Access)
Device access to memory without CPU involvement. In a TEE, DMA targets must be shared (unencrypted) pages, which is why bounce buffers exist. → Module 2, §4.2.4

### `IOMMU` (Input-Output Memory Management Unit)
Hardware that translates and restricts device memory access. Programmed by the hypervisor, which is why it is not a confidentiality control in a TEE.

---

## 2. AMD SEV and SEV-SNP

### `SEV` (Secure Encrypted Virtualization)
AMD's first-generation memory encryption for VMs. Confidentiality only — no integrity, and register state is plaintext on `VMEXIT`. → Module 2, §1.1

### `SEV-ES` (SEV Encrypted State)
Adds `VMSA` encryption so register state survives world switches confidentially. Still no memory integrity. → Module 2, §1.2

### `SEV-SNP` (SEV Secure Nested Paging)
Adds memory *integrity* via the `RMP`, defeating replay, remap, and aliasing. The first generation that defends against an actively malicious hypervisor. → Module 2, §1.3

### `C-bit`
A physical-address bit in the guest's page tables marking a page as encrypted. Set = encrypted (the inverse of TDX's SHARED bit). → Module 2, §1.1

### `ASID` (Address Space Identifier)
Identifies a confidential VM; the memory controller holds one AES key per ASID. → Module 2, §1.1

### `RMP` (Reverse Map Table)
A system-wide table, one entry per 4 KB page, recording ownership and the guest physical address a page may back. Hypervisor cannot write it. The core of SNP integrity. → Module 2, §1.3

### `PVALIDATE`
The guest instruction that accepts a page as private memory. One-shot and hardware-tracked, which closes the replay attack. → Module 2, §1.3

### `VMPL` (Virtual Machine Privilege Level)
Four privilege levels *inside* an SNP guest (VMPL0 most privileged), enabling a paravisor or SVSM to provide services the guest kernel should not implement itself. → Module 2, §1.3

### `SVSM` (Secure VM Service Module)
Trusted software at VMPL0 providing in-guest services — most importantly a vTPM that the hypervisor cannot forge. → Module 2, §1.3

### `VMSA` (VM Save Area)
The encrypted, integrity-checked region holding guest register state across world switches. Its ciphertext is what CipherLeaks observes. → Module 2, §1.2

### `GHCB` (Guest-Hypervisor Communication Block)
A shared page through which the guest deliberately discloses what the hypervisor needs for I/O emulation. → Module 2, §1.2

### `#VC` (VMM Communication Exception)
The in-guest exception raised for operations requiring hypervisor involvement, making disclosure explicit and guest-controlled. → Module 2, §1.2

### `PSP` / `ASP` (Platform Security Processor / AMD Secure Processor)
The on-die coprocessor that is AMD's root of trust: holds keys, computes launch measurements, signs attestation reports. → Module 2, §1.1

### `VCEK` (Versioned Chip Endorsement Key)
The report-signing key, derived from chip-unique secrets **and the current TCB version** — which is why firmware updates invalidate cached certificates. → Module 2, §1.4

### `VLEK` (Versioned Loaded Endorsement Key)
An alternative to VCEK, keyed to a cloud service provider rather than an individual chip; reports no longer fingerprint a specific machine. → Module 3, §3.2

### `VMPCK` (VM Platform Communication Key)
Established at launch, known only to the guest and the AMD Secure Processor; secures the guest's attestation-request channel. → Module 2, §1.4

### `ARK` / `ASK` (AMD Root Key / AMD SEV Signing Key)
The certificate chain anchoring VCEK/VLEK signatures. The ARK is the trust anchor to pin. → Module 3, §3.2

### `KDS` (Key Distribution Service)
AMD's service for retrieving VCEK certificates and the CA chain. A runtime dependency on the verification path. → Module 3, §3.2

### `MA_REPORT_ID` (Migration Agent Report ID)
Links an attestation report to its migration agent's report, if migration is enabled. → Module 2, §1.4

---

## 3. Intel TDX and SGX

### `TDX` (Trust Domain Extensions)
Intel's VM-level TEE. Reaches the same guarantees as SEV-SNP through a new CPU mode and an Intel-signed software module. → Module 2, §2

### `TD` (Trust Domain)
A TDX confidential guest.

### `SEAM` (Secure Arbitration Mode)
A CPU mode more privileged than VMX root, in which only the TDX Module runs. → Module 2, §2.1

### `TDX Module`
Intel-signed software running in SEAM mode that arbitrates every transition into and out of a TD. Adds ~10⁵ lines of code to the TCB, in exchange for patchability. → Module 2, §2.1

### `SEAMCALL` / `TDCALL`
The interfaces by which the hypervisor and the guest respectively request services from the TDX Module. → Module 2, §2.1

### `SHARED bit`
The most significant guest-physical-address bit. Set = shared and plaintext; clear = private, encrypted, integrity-protected. The inverse convention to AMD's C-bit. → Module 2, §2.2

### `Secure EPT` (Secure Extended Page Table)
The second-stage page table for TD private memory, owned by the TDX Module rather than the hypervisor. TDX's equivalent of the RMP invariant. → Module 2, §2.2

### `MKTME` (Multi-Key Total Memory Encryption)
The memory encryption engine providing a per-TD key. → Module 2, §2.2

### `MRTD` (Measurement Register for Trust Domain)
The build-time launch measurement, finalized when the TD is constructed. Immutable thereafter. → Module 2, §2.3

### `RTMR0-3` (Runtime Measurement Registers)
Four extend-only registers behaving like TPM PCRs, covering kernel, initrd, rootfs, and workload. TDX's architectural advantage over SEV-SNP. → Module 2, §2.3

### `TDREPORT`
Local evidence MAC'd with a CPU-local key. **Not remotely verifiable** — it must be converted to a Quote. → Module 2, §2.4

### `TD Quote`
The remotely verifiable, asymmetrically signed attestation produced by the TD Quoting Enclave from a TDREPORT. → Module 2, §2.4

### `QE` (Quoting Enclave)
An SGX enclave that verifies a TDREPORT's MAC and re-signs it with an attestation key. SGX's surviving job. → Module 2, §2.4

### `DCAP` (Data Center Attestation Primitives)
Intel's library and collateral model for third-party attestation verification without a live Intel service on every check. → Module 3, §3.3

### `PCS` / `PCCS` (Provisioning Certification Service / Caching Service)
Intel's source for verification collateral, and the local cache you run in production. → Module 3, §3.3

### `PCK` (Provisioning Certification Key)
The certificate identifying a specific CPU package and TCB level in the Intel chain. → Module 3, §3.3

### `FMSPC` (Family-Model-Stepping-Platform-CustomSKU)
The platform identifier under which Intel publishes TCB info. → Module 3, §3.3

### `SGX` (Software Guard Extensions)
Intel's process-level enclave TEE. Disqualified for model serving by memory limits and lack of device access; survives as the TDX Quoting Enclave. → Module 1, §4.1

### `EPC` (Enclave Page Cache)
The limited encrypted memory region backing SGX enclaves — the constraint that ruled SGX out for LLM weights.

---

## 4. Other TEE Architectures

### `CCA` (Confidential Compute Architecture)
ARM's VM-level TEE, introducing Realms. → Module 2, §3

### `RMM` (Realm Management Monitor)
ARM CCA's trusted component managing Realms; the analogue of the TDX Module. → Module 2, §3

### `GPT` (Granule Protection Table)
ARM CCA's page-ownership table; the analogue of AMD's RMP. → Module 2, §3

### `CoVE` (Confidential VM Extension)
RISC-V's confidential computing standard, with a TEE Security Manager in the TDX Module role. Notable as an *open* TEE. → Module 2, §3

---

## 5. Attestation and Evidence

### `RATS` (Remote ATtestation procedureS)
The IETF architecture (RFC 9334) defining Attester, Verifier, Relying Party, Endorser, and Reference Value Provider. → Module 3, §1.1

### Attester / Verifier / Relying Party
The entity producing Evidence; the entity appraising it; the entity acting on the result. **Who plays Verifier is the most consequential decision in a confidential design.** → Module 3, §1.3

### Endorser
The party vouching that hardware is genuine — AMD, Intel, NVIDIA — via certificate chains. → Module 3, §1.1

### Reference Value Provider
The party publishing what measurements *should* be. Usually the weakest link. → Module 3, §7

### Passport Model
The attester obtains an attestation result once and presents it to relying parties. GCP's model. → Module 3, §1.2

### Background-Check Model
The relying party forwards evidence to a verifier itself. AWS Nitro's model; naturally fresher. → Module 3, §1.2

### Launch Measurement
A hash of the initial guest image and vCPU state, computed before execution. Immutable. `MEASUREMENT` on SNP, `MRTD` on TDX. → Module 1, §3.2

### Runtime Measurement
Extend-only registers covering what was loaded after launch. `RTMR` on TDX; a vTPM PCR on SEV-SNP. → Module 1, §3.2

### `PCR` (Platform Configuration Register)
A TPM register that can only be extended, never set: `PCR_new = H(PCR_old ‖ measurement)`. → Module 1, §3.2

### Event Log
The ordered record of what was measured, allowing a verifier to replay and interpret PCR values rather than compare against an opaque golden hash. → Module 3, §3.1

### `REPORT_DATA` / `REPORTDATA`
64 caller-supplied bytes in an attestation report. Carries the nonce and/or the TLS public key hash. The basis of RA-TLS. → Module 1, §3.4

### `HOST_DATA`
32 bytes supplied by the *hypervisor* at launch. Untrusted by construction — useful for correlation, never for security. → Module 2, §1.4

### `TCB_VERSION` / `SVN` (Security Version Number)
Firmware component security versions in the report. Pin a **floor**, never an exact value. → Module 3, §4.2

### `EAT` (Entity Attestation Token)
RFC 9711's standard claims format for attestation. The reason GCP tokens have claims named `eat_nonce`, `dbgstat`, `hwmodel`. → Module 3, §3.4

### `CoRIM` (Concise Reference Integrity Manifest)
A standard format for publishing reference values. → Module 3, §3.4

### `CoSWID` (Concise Software Identification)
Software identification tags binding reference values to identifiable software. → Module 3, §3.4

### `RA-TLS` (Remote Attestation TLS)
Binding an attestation report to a TLS session by embedding the public key hash in the report's caller-supplied data. The only clean end-to-end guarantee for a client. → Module 3, §6

### `dm-verity`
A Merkle tree over a read-only filesystem whose root hash is measured, with per-block verification on read. What makes a measured rootfs meaningful at runtime. → Module 3, §2.2

### Fail-Closed / Fail-Open
Whether an unreachable verifier or stale collateral blocks operation or is ignored. **Choosing by accident means choosing fail-open.** → Module 3, §3.3

---

## 6. Cryptography and Key Management

### `AES-XTS`
The length-preserving, address-tweaked, **deterministic** mode used for memory encryption. Its determinism is the ciphertext side channel. → Module 1, §3.3

### `AES-GCM`
Authenticated, non-deterministic mode used on the GPU PCIe path, where nonce and tag expansion is affordable. → Module 4, §2.2

### `KEK` / `DEK` (Key Encryption Key / Data Encryption Key)
Envelope encryption: bulk data under a DEK, the DEK wrapped under a KEK. Rotating the KEK is cheap; rotating the DEK means re-encrypting the model. → Module 6, §3.2

### `SKR` (Secure Key Release)
Releasing key material only to an environment whose attestation satisfies a policy. → Module 3, §5

### `HPKE` (Hybrid Public Key Encryption)
RFC 9180 public-key encryption, usable for client-side payload encryption to an attested public key. → Module 6, §5.2

### `SPDM` (Security Protocol and Data Model)
The DMTF standard for mutually authenticated key exchange with a device, used to establish the CPU-TEE-to-GPU session. → Module 4, §2.3

### `IDE` (Integrity and Data Encryption)
The PCIe link-layer standard for encrypting and integrity-protecting bus traffic. → Module 4, §2.3

### `TDISP` (TEE Device Interface Security Protocol)
The PCIe standard for assigning a device interface directly into a TEE. Eliminates bounce buffers when broadly implemented. → Module 4, §6.2

### Ciphertext Side Channel
Inferring plaintext changes by observing deterministic ciphertext. Architectural, not a bug. CipherLeaks and successors. → Module 2, §5.1

---

## 7. Confidential GPUs

### `CC mode` (Confidential Computing mode)
The NVIDIA GPU device state. `OFF`, `ON` (protections active, profiling disabled), or `DEVTOOLS` (**not a security mode**). → Module 4, §2.1

### Protected / Unprotected Memory Region
The HBM split: protected holds weights, KV cache, and activations; unprotected is the ciphertext bounce landing zone. → Module 4, §2.2

### Bounce Buffer
A shared, host-visible staging buffer through which encrypted data crosses the PCIe boundary. The dominant source of CC overhead. → Module 4, §2.2

### `GSP` (GPU System Processor)
The GPU's on-die microcontroller whose firmware is measured as part of GPU attestation. → Module 4, §2.2

### `RIM` (Reference Integrity Manifest)
NVIDIA's signed publication of expected measurements for a firmware and driver version. The GPU's answer to the reference-value problem. → Module 4, §3.1

### `NRAS` (NVIDIA Remote Attestation Service)
NVIDIA's hosted verifier. Convenient; adds a dependency and reveals your fleet. A local verifier is the alternative. → Module 4, §3.1

### `nvtrust`
NVIDIA's open-source tooling for GPU attestation and verification. → Module 4, lab

### Composite Attestation
A single policy decision over *bound* CPU TEE, GPU, and workload-image evidence. Verifying them separately proves little. → Module 4, §3.2

### `SPT` (Single GPU Passthrough)
The Hopper-generation confidential GPU mode: one GPU per confidential VM, no protected interconnect. Appears as `cc_feature` in GCP tokens. → Module 4, §4.1

### Protected NVLink
Blackwell-generation hardware-encrypted GPU-to-GPU interconnect, enabling 1/2/4/8-GPU confidential groups and therefore tensor parallelism inside a TEE. → Module 4, §4.2

### `SEV-TIO` (SEV Trusted I/O)
AMD's trusted device assignment, pairing Instinct accelerators with SEV-SNP. → Module 4, §6.1

### `nvidia-persistenced`
The daemon keeping the driver loaded so that CC session renegotiation and re-attestation do not occur on every process exit. Required in CC mode. → Module 4, §2.4

### GPU Ready State
The hardware gate that refuses compute until attestation has succeeded (`nvidia-smi conf-compute -srs 1`). Puts attestation on the cold-start critical path by design. → Module 4, §2.4

---

## 8. Google Cloud

### GKE Hypercluster
Google Cloud's supercomputing-scale Kubernetes architecture: a single conformant control plane managing accelerator capacity as **linked runners** — instances not registered as `Node` objects and carrying no Kubernetes agents — across regions. Eligibility-gated. **Scale is not a confidentiality property**; see Sealed configuration. → Module 5, §2 & §2.5, Module 6, §2

### Sealed configuration (Hypercluster)
The runner mode that makes Hypercluster confidential: minimal OS image with SSH and container shell disabled, an attestation agent reporting firmware and workload measurements, instance-enforced signed-image-digest policy, and an explicit statement that administrators and Google personnel cannot access the host. **The default configuration retains administrator and SRE SSH access and is not a confidential deployment.** → Module 5, §2.5

### Titanium Intelligence Enclave (TIE)
The TEE backing sealed Hypercluster runners on TPUs, part of Google's Private AI Compute platform. GPU-based sealed runners use NVIDIA Confidential Computing instead. → Module 5, §2.5

### AI Hypercomputer
Google Cloud's holistic AI supercomputing architecture combining performance-optimized hardware, open software frameworks (LeaderWorkerSet, Kueue, JobSet), and storage systems. → Module 5, §2

### Confidential VM
The base product. `--confidential-compute-type=SEV | SEV_SNP | TDX`. **Plain SEV lacks memory integrity.** → Module 5, §1

### Confidential GKE Nodes
Node pools whose VMs are Confidential VMs. `--confidential-node-type=sev|sev_snp|tdx`. **Does not cover the Google-operated control plane.** → Module 5, §2

### Confidential Containers (CoCo)
An upstream Kubernetes architecture running pods in isolated hardware microVM TEEs (e.g. Kata Containers on TDX/SEV-SNP), keeping the node Kubelet and host OS outside the pod's TCB. → Module 5, §6 & Module 6, §2.1

### Confidential Space
A hardened, measured image running exactly one container and emitting an attestation token. Separates workload author, workload operator, and data collaborators — the operator has full project admin and still cannot read the data. → Module 5, §3

### Container Launcher
The Confidential Space component that starts the workload and serves attestation tokens over `/run/container_launcher/teeserver.sock`. → Module 5, §4.1

### Google Cloud Attestation
GCP's verifier, issuing OIDC JWT attestation tokens. Note that for a threat model including Google, a Google-signed result is a statement by the distrusted party. → Module 5, §4.1

### `dbgstat`
The token claim indicating debug state: `disabled-since-boot` or `enabled`. **The single most important claim to assert.** → Module 5, §3.3

### `hwmodel`
The token claim naming the hardware: `GCP_AMD_SEV`, `GCP_AMD_SEV_ES`, `GCP_INTEL_TDX`, or `GCP_SHIELDED_VM`. Assert the one you expect — Shielded VM is not a TEE. → Module 5, §4.2

### `swname` / `swversion`
`CONFIDENTIAL_SPACE` or `GCE`, and the image version. Distinguishes a Confidential Space image from a bare confidential VM. → Module 5, §4.2

### `submods`
The token's nested claim structure: `submods.container` (image digest, signatures, args, env), `submods.gce` (project, zone, instance), `submods.confidential_space` (support attributes, monitoring), `submods.nvidia_gpu` (`cc_mode`, per-GPU model, driver, VBIOS). → Module 3, §5.2

### `eat_nonce`
Caller-supplied freshness values in the token. Use them; relying on `exp` alone accepts a replay window. → Module 3, §4.1

### `WIF` (Workload Identity Federation)
The mechanism exchanging an attestation token for a Google credential, with a CEL **attribute condition** over the token claims acting as the release policy. → Module 5, §4.2

### `CMEK` (Customer-Managed Encryption Keys)
Encryption keys you control in Cloud KMS. Better than Google-managed, and still not attestation-gated. → Module 5, §1.3

### `EKM` (External Key Manager)
Cloud KMS backed by a key manager **outside** Google. The qualitatively different option for a model provider's weight KEK. → Module 5, §4.3

### Shielded VM
Secure boot, vTPM, and integrity monitoring. **Not confidential computing** — memory is not encrypted. → Module 5, §5.3

### Binary Authorization
GKE admission control blocking unsigned or unattested images. Prevents *deployment*; the attestation policy prevents *key release*. → Module 5, §5.1

### `VPC-SC` (VPC Service Controls)
Perimeter controls on data egress — one of the few places a control *outside* the TEE helps, because the threat is the workload itself. → Module 5, §5.3

---

## 9. LLM Serving & AI Orchestration Intersection

### `LeaderWorkerSet` (`LWS`)
An open-source Kubernetes API controller for distributed multi-node AI workloads. Coordinates 1 Leader Pod and N Worker Pods as a unified group, handling mutual attestation and tensor parallel weight distribution. → Module 6, §2 & §3.2

### `Dynamic Workload Scheduler` (`DWS`) / `flex-start`
Google Cloud scheduling mechanism for accelerator infrastructure. `flex-start` allows gang-scheduling multi-node GPU capacity with deterministic execution windows, mitigating cold-start provisioning jitter. → Module 5, §2 & Module 6, §4.2

### `Kueue`
A Kubernetes-native queue manager that orchestrates batch and inference capacity, fair-sharing, and priority preemption across multi-tenant GPU pools. → Module 6, §4.2 & Module 7, §4.3

### `KV cache` (Key-Value cache)
Cached attention keys and values. **A high-fidelity encoding of everything the model has been told** — treat it with the sensitivity of the prompt itself. → Module 6, §6.1

### Prefix Caching
Reusing computed KV blocks for a shared prompt prefix. **Across tenants this is a timing oracle over other tenants' prompt content**, and memory encryption does not help. → Module 6, §6.2

### Continuous Batching
Iteration-level scheduling that mixes multiple tenants' requests into one forward pass. Isolation between them is software, not hardware. → Module 6, §7.2

### Disaggregated Prefill/Decode
Splitting the two phases across machines, shipping KV cache between them. Requires mutually attested, encrypted transport in a confidential design. → Module 6, §6.3

### `TTFT` / `TPOT` (Time To First Token / Time Per Output Token)
Prefill-dominated and decode-dominated latency metrics respectively. TPOT is least affected by confidential computing. → Module 7, §1.3

### Arithmetic Intensity
FLOPs per byte transferred. CC taxes the denominator only, so relative overhead scales as $1/I$ — which is why it shrinks with model size, batch size, and sequence length. → Module 4, §5.2

### Attested Ingress
Terminating TLS, or decrypting the payload, **inside** the TEE. A managed L7 load balancer holds plaintext prompts and voids the guarantee. → Module 6, §5

### `MaaS` / `3P MaaS` (Model-as-a-Service / Third-Party MaaS)
Serving another organization's model on your infrastructure. The three-way trust conflict that motivates this entire book. → index

### `PCC` (Private Cloud Compute)
Apple's confidential inference architecture: stateless computation, enforceable guarantees, no privileged runtime access, non-targetability, verifiable transparency. The bar to measure against. → Module 7, §6.2

---

Return to the [curriculum index](index.md), or continue to [Appendix B: Primary Source Reading List](appendix_reading_list.md).
