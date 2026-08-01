# Appendix B: Primary Source Reading List

This book has a shelf life. The architecture is stable; the availability matrices, GPU generations, and attack literature are not. This appendix exists so that any claim in these notes can be re-verified at its source.

Entries are annotated with what each source is actually good for. Links marked **(verified)** were fetched or confirmed while writing these notes; entries without a link are cited by title and publisher because I did not verify a stable URL — search by title rather than trusting a guessed one.

---

## 1. Standards and Architecture

### `RFC 9334` — Remote ATtestation procedureS (RATS) Architecture
**(verified)** <https://www.rfc-editor.org/info/rfc9334/> · <https://datatracker.ietf.org/doc/html/rfc9334>

The vocabulary the whole field uses: Attester, Verifier, Relying Party, Endorser, Reference Value Provider, and the passport versus background-check models. Short, readable, and worth reading in full once. → Module 3, §1

### `RFC 9711` — The Entity Attestation Token (EAT)
**(verified)** <https://datatracker.ietf.org/doc/rfc9711/>

The standard claims format that GCP's attestation token follows in spirit — this is why the token has `eat_nonce`, `dbgstat`, `hwmodel`, and `swname` rather than vendor-specific field names. → Module 3, §3.4

### `RFC 9180` — Hybrid Public Key Encryption (HPKE)
The primitive for client-side payload encryption to an attested public key. → Module 6, §5.2

### IETF RATS Working Group — CoRIM and CoSWID drafts
Standard formats for publishing reference values and software identity. Watch these; they are the eventual answer to the reference-value problem. Track via the RATS working group's datatracker page. → Module 3, §3.4

### DMTF — Security Protocol and Data Model (SPDM) Specification
The device authentication and key-exchange protocol underlying confidential GPU session establishment. Published by DMTF. → Module 4, §2.3

### PCI-SIG — TDISP and IDE specifications
TEE Device Interface Security Protocol and Integrity and Data Encryption. The standardization endpoint that eventually removes bounce buffers. Membership-gated; the DMTF and vendor whitepapers below are the accessible summaries. → Module 4, §6.2

### Confidential Computing Consortium
<https://confidentialcomputing.io/>

The Linux Foundation body whose definition anchors Module 1, §1.3. Its published whitepapers are a reasonable neutral overview when you need something vendor-independent to hand to a stakeholder.

---

## 2. AMD SEV-SNP

### AMD — *SEV Secure Nested Paging Firmware ABI Specification*
The authoritative reference for the attestation report structure, guest policy bits, `TCB_VERSION`, and the firmware interface. **This is the document to consult when you need to know exactly what a report field means.** Published on AMD's developer site under the SEV documentation section.

### AMD — *SEV-SNP Platform Attestation Using VirTEE/SEV* (Publication 58217)
**(verified)** <https://www.amd.com/content/dam/amd/en/documents/developer/58217-epyc-9004-ug-platform-attestation-using-virtee-snp.pdf>

A practical walkthrough of obtaining and verifying an attestation report with the open-source tooling. The closest thing to a hands-on guide from the vendor. → Module 2, lab

### AMD — *SEV-SNP Attestation: Establishing Trust in Guests* (Jeremy Powell)
**(verified)** <https://www.amd.com/content/dam/amd/en/documents/developer/lss-snp-attestation.pdf>

A clear presentation of the attestation flow, the VCEK derivation, and the certificate chain. Good for building intuition before reading the ABI spec.

### `virtee/snpguest`
**(verified)** <https://github.com/virtee/snpguest>

The CLI used in the Module 2 lab: request a report, fetch the CA and VCEK from AMD's KDS, verify the chain. Reading its source is an efficient way to understand what verification actually entails.

### `google/go-sev-guest`
**(verified)** <https://github.com/google/go-sev-guest>

A Go library wrapping `/dev/sev-guest` plus attestation verification. The reference implementation to read if you are building your own verifier — which Module 3, §1.3 argues you should be.

---

## 3. Intel TDX

### Intel — *Trust Domain Extensions (TDX) Module Architecture Specification* and the TDX whitepaper series
The authoritative description of SEAM mode, the TDX Module, secure EPT, `MRTD`/`RTMR`, and the TDREPORT-to-Quote path. Published on Intel's TDX documentation portal.

### Intel — *Device Attestation Model in Confidential Computing Environment* (whitepaper)
**(verified)** <https://cdrdv2-public.intel.com/783079/whitepaper-device-attestation-model-in-confidential-computing-environment-v0.6.4.pdf>

How device (accelerator) attestation composes with CPU TEE attestation. Directly relevant to the composite-attestation argument in Module 4, §3.2.

### Intel Trust Authority — Attestation Patterns and EAT Profile
**(verified)** <https://docs.trustauthority.intel.com/main/articles/articles/ita/concept-patterns.html> · <https://portal.trustauthority.intel.com/eat_profile.html>

A clear articulation of passport versus background-check in a production service, and a concrete EAT profile. Useful even if you never use the product.

### Intel — DCAP (Data Center Attestation Primitives) documentation and `SGXDataCenterAttestationPrimitives`
The collateral model, PCCS caching, and the verification library. Read this before deciding your fail-closed / fail-open posture. → Module 3, §3.3

---

## 4. Confidential GPUs

### NVIDIA — *Confidential Computing on NVIDIA H100 GPUs for Secure and Trustworthy AI*
**(verified)** <https://developer.nvidia.com/blog/confidential-computing-on-h100-gpus-for-secure-and-trustworthy-ai/>

The canonical introduction to CC mode, protected memory, bounce buffers, and the attestation flow. Start here for Module 4.

### NVIDIA — *Hardware-Rooted AI Security That Won't Slow You Down*
**(verified)** <https://developer.nvidia.com/blog/hardware-rooted-ai-security-that-wont-slow-you-down>

Covers the Blackwell-generation multi-GPU confidential computing story, including NVLink encryption and 1/2/4/8-GPU assignment — the change that determines which models are servable at all. → Module 4, §4.2

### *Creating the First Confidential GPUs* — Communications of the ACM
**(verified)** <https://cacm.acm.org/practice/creating-the-first-confidential-gpus/>

The design retrospective, written by the engineers who built it. The best available explanation of *why* the architecture looks the way it does, as opposed to what it does.

### Rob Nertney — *Remote Attestation for NVIDIA Hopper and Blackwell GPUs, CPUs, and Beyond* (OC3 2025)
**(verified)** <https://cdn.prod.website-files.com/63c54a346e01f30e726f97cf/67f00a27564271b2f87c4988_Rob%20Nertney%20OC3%202025%20-%20Nvidia%20Blackwell.pdf>

Composite CPU + GPU attestation from NVIDIA's perspective, including RIMs and the NRAS-versus-local-verifier decision. → Module 4, §3

### `NVIDIA/nvtrust`
**(verified)** <https://github.com/NVIDIA/nvtrust>

The attestation SDK and local verifier used in the Module 4 lab. The issue tracker is unusually informative about real-world failure modes — measurement mismatches, ready-state problems, CPU/GPU pairing constraints.

---

## 5. Google Cloud

### Confidential VM — overview, supported configurations, and release notes
**(verified)** <https://docs.cloud.google.com/confidential-computing/confidential-vm/docs/confidential-vm-overview>

The supported-configurations page is the one to bookmark: machine types, CC technologies, and zones, which change often enough that nothing in Module 5 should be trusted over it.

### Confidential GKE Nodes
**(verified)** <https://docs.cloud.google.com/kubernetes-engine/docs/how-to/confidential-gke-nodes>

Enablement flags, limitations, and the explicit statement that Confidential GKE Nodes does not change the security measures applied to cluster control planes — the sentence Module 5, §2.3 is built on.

### Confidential GKE Nodes with GPUs
**(verified)** <https://docs.cloud.google.com/kubernetes-engine/docs/how-to/gpus-confidential-nodes>

The GPU constraints: machine type, one H100, TDX, no GPU sharing, and the GKE version requirements. → Module 4, §4.1

### Confidential Space — overview and attestation token claims
**(verified)** <https://docs.cloud.google.com/confidential-computing/confidential-space/docs/confidential-space-overview> · <https://docs.cloud.google.com/confidential-computing/confidential-space/docs/reference/token-claims>

The token-claims reference is the single most useful GCP page in this book. Every claim in the Module 3, §5.2 policy checklist comes from it, including the `submods.nvidia_gpu` structure.

### `salrashid123/confidential_space`
**(verified)** <https://github.com/salrashid123/confidential_space>

Extensive worked examples of Confidential Space with attested key release. More detailed and more honest about edge cases than the official quickstarts.

### Google Cloud — *How Confidential Computing Lays the Foundation for Trusted AI*
**(verified)** <https://cloud.google.com/blog/products/identity-security/how-confidential-computing-lays-the-foundation-for-trusted-ai/>

Google's own framing of the confidential AI story. Useful for understanding the vocabulary a Google-internal audience will expect, and worth reading alongside Module 7, §5.2 on where framing outruns mechanism.

---

## 6. Comparative Architectures

### Apple — *Private Cloud Compute: A New Frontier for AI Privacy in the Cloud*
**(verified)** <https://security.apple.com/blog/private-cloud-compute/>

**The single most valuable document in this list.** The five requirements — stateless computation, enforceable guarantees, no privileged runtime access, non-targetability, verifiable transparency — are the most rigorous published statement of what confidential inference should mean. Read it before designing anything. → Module 7, §6.2

### Apple — *Expanding Private Cloud Compute*
**(verified)** <https://security.apple.com/blog/expanding-pcc/>

PCC extended onto Google Cloud with Intel TDX, NVIDIA Confidential Computing, and Google's Titan chip. Component for component, the architecture Module 6 describes — and an explicit demonstration that the model provider can retain software control and independent verification while running on another party's infrastructure. → Module 7, §6.3

### Apple — *Stateless Computation and Enforceable Guarantees*
**(verified)** <https://security.apple.com/documentation/private-cloud-compute/statelessandenforcable>

The detailed technical documentation behind the first two requirements.

### AWS — *Attest an Amazon EC2 instance with AMD SEV-SNP*
**(verified)** <https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/snp-attestation.html>

Useful as a contrast to GCP's model, and AWS's KMS condition-key approach to attested key release is the cleanest in the industry — worth understanding even if you never deploy on AWS. → Module 7, §6.2

### Azure Confidential Containers and the CNCF Confidential Containers project
The pod-sandbox model that Module 1, §4.4 and Module 5, §6 argue is the architecturally better unit of confidentiality for Kubernetes. Track via the CNCF Confidential Containers project and Microsoft's AKS confidential containers documentation.

---

## 7. Attack Literature

Read at least the first of these. A model provider's security team will know this work, and being able to discuss it accurately is what separates a credible security claim from a marketing one.

### CIPHERLEAKS — *Breaking Constant-time Cryptography on AMD SEV via the Ciphertext Side Channel* (USENIX Security 2021)
**(verified)** <https://www.usenix.org/conference/usenixsecurity21/presentation/li-mengyuan>

The foundational ciphertext side-channel result. Establishes that deterministic memory encryption leaks value-change information, and that constant-time code is not a defense. → Module 2, §5.1

### CipherH — *Automated Detection of Ciphertext Side-channel Vulnerabilities* (USENIX Security 2023)
**(verified)** <https://www.usenix.org/system/files/usenixsecurity23-deng-sen.pdf>

Automated discovery of the vulnerability class. Relevant if you need to assess your own code.

### Cipherfix, CipherGuard, Zebrafix — mitigation approaches
**(verified)** <https://arxiv.org/pdf/2210.13124> · <https://arxiv.org/html/2502.13401> · <https://arxiv.org/pdf/2502.09139>

Software and compiler mitigations that break plaintext-to-ciphertext determinism. Read one to understand why the mitigation is expensive and why "just keep keys out of guest DRAM" is often the better answer.

### TDXdown — *Single-Stepping and Instruction Counting Attacks against Intel TDX* (CCS 2024)
**(verified)** <https://dl.acm.org/doi/10.1145/3658644.3690230>

Defeats TDX's built-in single-stepping mitigation. The reference for Module 2, §5.3 and for why the TDX Module's patchability is both an advantage and a source of TCB churn.

### Heracles — *Chosen Plaintext Attack on AMD SEV-SNP* (CCS 2025)
**(verified)** <https://heracles-attack.github.io/Heracles-CCS2025.pdf>

The current state of the art against SEV-SNP. Read the threat model section carefully — the assumptions matter for whether it applies to your deployment.

### Heckler, WeSee, CacheWarp
Interrupt-injection and cache-manipulation attacks against SEV-SNP. Search by name; each has a project page. The structural lesson — anything the hypervisor retains control of is an attack surface — matters more than the individual results. → Module 2, §5.2

---

## 8. Performance Studies

### *Confidential Computing on NVIDIA Hopper GPUs: A Performance Benchmark Study*
**(verified)** <https://arxiv.org/pdf/2409.03992>

The most-cited H100 CC benchmark. Establishes that overhead is dominated by CPU–GPU PCIe transfer and shrinks with model size, batch size, and sequence length. The empirical basis for Module 4, §5.2.

### *Performance of Confidential Computing GPUs*
**(verified)** <https://arxiv.org/pdf/2505.16501>

A broader treatment across workload types.

### *Confidential LLM Inference: Performance and Cost Across CPU and GPU TEEs*
**(verified)** <https://www.arxiv.org/pdf/2509.18886>

Separates CPU TEE cost from GPU TEE cost — the decomposition Module 7, §1.1 insists on — and includes a cost model.

### *Benchmarking Confidential GPU Inference on NVIDIA H100 under Intel TDX*
**(verified)** <https://arxiv.org/html/2607.19353v1>

The configuration closest to the Module 6 reference architecture, and the source of the 15–25% capacity-reservation heuristic.

---

## How to Use This List

- **Designing?** Apple PCC first, then RFC 9334, then the Confidential Space token-claims reference.
- **Debugging attestation?** The AMD ABI spec or Intel TDX spec for field semantics, then `snpguest` / `go-sev-guest` / `nvtrust` source.
- **Justifying a security claim?** The attack literature in §7. Volunteering the qualifications is what makes the claim credible.
- **Capacity planning?** The performance studies in §8, then replace them with your own measurements as soon as you have them.
- **Checking whether something in these notes is still true?** The Google Cloud documentation in §5, which changes on a scale of months.

---

Return to the [curriculum index](index.md) or [Appendix A: Master Glossary](appendix_glossary_and_terminology.md).
