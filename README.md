# Strata

**A unified, open-source compute fabric and agentic harness for the intelligence era**

> Vision Paper · v0.3 · June 2026
> OSS-first · CNCF track · Healthcare vertical launch
> *Pre-publication draft — confidential*

---

## Table of contents

1. [The problem](#1-the-problem)
2. [Our answer: Strata](#2-our-answer-strata)
3. [Guiding principles](#3-guiding-principles)
4. [Architecture: the compute fabric](#4-architecture-the-compute-fabric)
5. [Multi-tenancy — isolation at every layer](#5-multi-tenancy--isolation-at-every-layer)
6. [Confidential compute and containers](#6-confidential-compute-and-containers)
7. [Agentic management and operations](#7-agentic-management-and-operations)
8. [Hyperconverged control plane — no dedicated machines](#8-hyperconverged-control-plane--no-dedicated-machines)
9. [Storage and networking](#9-storage-and-networking)
10. [Accelerator subsystem](#10-accelerator-subsystem)
11. [Why healthcare first](#11-why-healthcare-first)
12. [Strata Harness — the trust fabric](#12-strata-harness--the-trust-fabric)
13. [Deploying agentic systems — the hospital use case](#13-deploying-agentic-systems--the-hospital-use-case)
14. [Guardrail engine — layered enforcement](#14-guardrail-engine--layered-enforcement)
15. [Human escalation queue](#15-human-escalation-queue)
16. [Federated harness policy](#16-federated-harness-policy)
17. [Compliance as a first-class output](#17-compliance-as-a-first-class-output)
18. [Provider ecosystem](#18-provider-ecosystem)
19. [Community and governance](#19-community-and-governance)
20. [Roadmap](#20-roadmap)
21. [Why now](#21-why-now)

---

## 1. The problem

Modern infrastructure is deeply fragmented. Compute, storage, and networking are provisioned through separate control planes, separate APIs, and separate operational teams. Running a GPU cluster for inference beside a high-throughput NVMe database beside a latency-sensitive clinical service means operating three fundamentally different systems — often across three different vendors, with three different operational models.

The cost of this fragmentation is highest where it matters most: environments where reliable, low-latency, AI-augmented computing can save lives. A hospital cannot afford a three-day provisioning cycle. A medical researcher cannot afford vendor lock-in that prevents them from moving a model to an edge appliance in a clinic with no cloud connectivity. A practitioner using an AI-assisted diagnostic tool cannot afford a platform that requires a dedicated infrastructure team to keep running.

Agentic AI deepens the problem further. A diagnostic agent that calls a drug interaction service, queries an EHR, invokes a radiology classifier, and summarizes a clinical note touches four different systems — each with its own API, its own access control model, its own audit log. There is no unified plane that says: this agent is approved to use these capabilities, within these guardrails, and every decision it makes is recorded in a form regulators can inspect.

> **The infrastructure layer should be invisible. Teams should think about what they are running, not where or how — and regulators should be able to verify what happened, without teams spending weeks assembling audit trails by hand.**

---

## 2. Our answer: Strata

Strata is a unified, open-source infrastructure control plane and agentic harness that manages compute, storage, networking, AI agent lifecycle and governance as a single coherent system. It treats processes, containers, and virtual machines as three expressions of the same abstraction. It runs on a Raspberry Pi edge node and on a 10,000-core GPU cluster. It runs on bare metal Linux and inside Kubernetes. It operates with no dedicated control plane hardware — the control plane is embedded in the agents that run on every node.

On top of the compute fabric, Strata Harness provides the trust and governance layer that makes it safe to deploy AI agents into clinical environments: a skill and tool registry, a layered guardrail engine, a human escalation queue, a federated policy system, and an immutable audit ledger — all running locally, with no patient data leaving the facility.

---

## 3. Guiding principles

| Principle | Description |
|---|---|
| **OSS first** | Everything we build is contributed to CNCF or the Linux Agent Foundation. No proprietary lock-in. Community shapes v1, as it did with Kubernetes. |
| **Vendor agnostic** | Cloud, on-premise, edge, hybrid. AWS, GCP, bare metal, a rack in a hospital basement — one control plane, one API. |
| **Any form factor** | Single-node edge appliance or thousand-node GPU cluster. Same binary, same API, same operational model. |
| **No control plane tax** | The control plane is embedded in every agent node. No dedicated machines, no circular upgrade dependencies. |
| **Resilient by default** | Workloads survive agent restarts, upgrades, and quorum loss. Stateful services are never disrupted by infrastructure operations. |
| **AI-native** | First-class GPU, NPU, and CPU inference acceleration. MIG partitioning, NUMA affinity, RoCEv2 fabric — built in from the start. |

---

## 4. Architecture: the compute fabric

Strata runs as a single binary — `strata-agent` — on every node. A subset of nodes (three to five) participate in a Raft consensus cluster that replicates desired state. All other nodes are learners: they receive replicated state without voting. The entire system is written in Rust.

Every resource — a compute workload, a storage volume, a network attachment, a tenant namespace, an agent deployment — is modeled as a declarative spec with a typed reconciler that drives actual state toward desired state.

```
┌──────────────────────────────────────────────────────────────┐
│  Strata Harness                                              │
│  Agent registry · skill registry · guardrail engine          │
│  audit ledger · escalation queue · federated policy          │
├──────────────────────────────────────────────────────────────┤
│  Agentic operations                                          │
│  Capacity planner · upgrade orchestrator · incident responder│
├──────────────────────────────────────────────────────────────┤
│  Control plane                                               │
│  Embedded Raft · declarative reconciler · scheduler          │
├──────────────────────────────────────────────────────────────┤
│  Multi-tenancy & confidential compute                        │
│  Namespace isolation · TEE (SEV-SNP / TDX) · attestation    │
├──────────────────────────────────────────────────────────────┤
│  Compute                                                     │
│  Process · OCI container · confidential container · VM       │
├──────────────────────────────────────────────────────────────┤
│  Storage                                                     │
│  NVMe (SPDK) · distributed block (Linstor) · JuiceFS        │
├──────────────────────────────────────────────────────────────┤
│  Networking                                                  │
│  eBPF/XDP (Cilium) · SR-IOV · RoCEv2 RDMA                  │
├──────────────────────────────────────────────────────────────┤
│  Accelerators                                                │
│  Nvidia MIG · AMD ROCm · Intel AMX · ARM SVE2               │
└──────────────────────────────────────────────────────────────┘
```

### Compute backends

- **Process** — cgroups v2, Linux namespaces for lightweight isolation without OCI overhead.
- **Container** — containerd over ttrpc. EROFS snapshotter for read-only inference model layers.
- **Confidential container** — OCI container inside a hardware TEE via Kata Containers (kata-cc). Image encrypted at rest; key released only after hardware attestation.
- **VM** — cloud-hypervisor (~100ms boot, VFIO GPU passthrough) or Firecracker for microVM isolation.

### Open source component map

| Layer | Component | Role |
|---|---|---|
| State store | TiKV | Distributed KV, horizontal scale, Rust client |
| Containers | containerd | Industry-standard runtime, ttrpc interface |
| Confidential containers | Kata Containers (kata-cc) | OCI containers inside hardware TEE |
| VMs | cloud-hypervisor | Rust VMM, fast boot, VFIO GPU passthrough |
| NVMe I/O | SPDK | Kernel-bypass NVMe via vhost-user |
| Distributed block | Linstor + DRBD | Synchronous replication, kernel-space path |
| Model weights FS | JuiceFS | S3-backed, aggressive local SSD cache |
| Networking | Cilium / eBPF | XDP fast path, per-tenant network policy |
| GPU management | NVML / ROCm SMI | MIG partitioning, topology queries |
| Identity | SPIFFE / SPIRE | Zero-trust workload identity, mTLS |
| Attestation | AMD KDS / Intel PCS | Hardware TEE measurement verification |
| Key management | HashiCorp Vault + TEE plugin | Sealed secret release gated on attestation |
| Observability | OpenTelemetry | Traces, metrics, logs — vendor-neutral |

---

## 5. Multi-tenancy — isolation at every layer

A hospital system is not a single tenant. A cardiology department, a radiology group, a clinical research team, and a third-party diagnostic AI vendor may all run workloads on the same Strata cluster simultaneously. Each must be isolated: their network traffic, their storage I/O, their GPU time, and their data must not leak across boundaries, and their operational failures must not cascade.

Strata models multi-tenancy as a first-class resource — the **Namespace** — that every other resource lives inside, and enforces isolation at each layer of the stack independently.

> **Tenant isolation is not a feature added on top of the platform. It is a structural property enforced at the hardware, kernel, and network layers simultaneously. No single isolation boundary is relied upon alone.**

### Isolation model by layer

| Layer | Isolation mechanism | What it prevents | Enforcement |
|---|---|---|---|
| **Network** | Per-tenant eBPF network policy · VXLAN tenant segments with distinct VNIs · deny-by-default between namespaces · explicit allow-list for cross-tenant service calls | Tenant A sniffing or injecting traffic on Tenant B's segment. Cross-tenant data exfiltration via network. | Cilium eBPF · XDP |
| **Compute** | cgroups v2 hierarchy — tenant root cgroup → workload cgroups · CPU quota and pinning per namespace · memory limits with hard eviction · PID namespace isolation | One tenant consuming all CPU and starving others. Process visibility across namespace boundaries. | cgroups v2 · Linux namespaces |
| **Storage** | Namespace-scoped volumes — cross-namespace mounts blocked at the reconciler · per-tenant I/O QoS (IOPS and bandwidth limits via blk-mq cgroups) · separate encryption keys per tenant volume | Tenant A consuming all NVMe bandwidth. Volume mounts crossing namespace boundaries. | blk-mq · LUKS |
| **GPU** | MIG partitions assigned per namespace — hardware-isolated memory between instances · time-slicing with strict quota enforcement via NVML for non-MIG deployments | Tenant A reading GPU memory residue from Tenant B's completed inference. GPU exhaustion by one tenant. | NVML MIG · VFIO |
| **Harness** | Skill registry is namespace-scoped · audit ledger partitioned by namespace · escalation queue events namespace-scoped · SPIFFE trust domain per tenant namespace | Cross-tenant agent invocation. Audit log data leaking between tenants. Skill enumeration across namespaces. | SPIFFE · Harness RBAC |
| **Identity** | Each namespace has its own SPIFFE trust domain. Workload certificates cannot authenticate to resources in another namespace without explicit federation policy. | A compromised workload in Tenant A authenticating to Tenant B's services using a stolen certificate. | SPIRE · mTLS |

### Hardware virtualization for tenant isolation

For tenants requiring stronger isolation than Linux namespaces and cgroups provide — regulated workloads, third-party vendor code, or environments where kernel-level isolation is insufficient — Strata supports hardware-backed isolation:

**SR-IOV NIC Virtual Functions** — each tenant namespace is assigned dedicated NIC VFs. Traffic passes through its VF, enforced in hardware by the physical NIC. No shared kernel network stack between tenants. The agent assigns VFs at namespace creation time via sysfs and binds them to `vfio-pci`.

**SR-IOV GPU Virtual Functions** — for Nvidia A-series and H-series GPUs that support SR-IOV, each tenant can be assigned a dedicated GPU VF rather than a MIG instance. VF isolation is enforced by the GPU hardware — no shared memory, no shared compute context, no cross-VF cache side channels.

**VM-per-tenant** — for the highest isolation tier, each tenant namespace runs inside a dedicated VM (cloud-hypervisor or Firecracker). The hypervisor provides the isolation boundary. A kernel vulnerability in one tenant's VM does not expose another. From the tenant's perspective the API is identical to the container or process tier — the isolation tier is an operator-configured property of the namespace spec.

Tenants select their isolation tier in their namespace spec. The scheduler enforces co-location constraints accordingly — VM-tier tenants are never placed on the same physical host as a lower isolation tier tenant unless the cluster operator explicitly allows it.

---

## 6. Confidential compute and containers

Tenant namespace isolation prevents workloads from seeing each other. Confidential compute prevents the infrastructure operator from seeing the workloads. These are different threat models, and both matter in healthcare.

A hospital running a diagnostic AI model trained on proprietary data does not want the cloud provider, the managed service operator, or even their own infrastructure team to be able to read the model weights or the patient data during inference. Hardware trusted execution environments (TEEs) make this possible: the CPU encrypts the memory of the workload, and only code with a verifiable measurement that matches a known-good value can decrypt it.

> **Confidential compute does not require trusting the infrastructure operator. The patient data and the model weights are encrypted by the CPU hardware itself. The operator sees only ciphertext — even with root access to the physical machine.**

### Supported TEE technologies

| Technology | Scope | Key property |
|---|---|---|
| **AMD SEV-SNP** | Full VM memory encryption | Per-VM key held in AMD PSP. Hardware attestation report proves VM initial state. Primary target for VM-tier confidential workloads. |
| **Intel TDX** | Trust Domain memory encryption | Keys managed by Intel TDX module in firmware. TDREPORT for remote attestation. Supported on 4th-gen Xeon Scalable and newer. |
| **Intel SGX** | Application-level enclaves | Smaller TCB than full-VM TEEs. Used for key management sidecars, attestation brokers, and cryptographic operations isolated from the guest OS. |
| **Kata Containers (kata-cc)** | OCI containers inside TEE | Unmodified container images, encrypted at rest, decrypted only inside a verified TEE. Container runtime, guest kernel, and OCI layers all measured and included in the attestation report. |

### Trust ladder — what each tier protects

| Tier | Isolation boundary | Residual threats | Suitable for |
|---|---|---|---|
| 1 — Standard container | cgroups + namespaces | Host OS, infrastructure operator, kernel exploit all have memory access | General workloads, non-sensitive data |
| 2 — VM-per-tenant | Hypervisor | Hypervisor and infrastructure operator still have VM memory access | Regulated workloads, stronger tenant isolation |
| 3 — Confidential VM (SEV-SNP / TDX) | CPU memory encryption | Physical memory bus attacks (mitigated by memory encryption) | PHI processing, model weight protection, operator-untrusted environments |
| 4 — Confidential container (kata-cc + SEV-SNP / TDX) | CPU memory encryption + measured container image | None within the defined threat model | Full operator-blind PHI inference, third-party model IP protection |

### Remote attestation and sealed secrets

Hardware attestation is only useful if a key release policy enforces it. Strata integrates with the remote attestation flow to gate secret delivery on a verified TEE measurement:

1. **TEE measurement** — the hardware generates an attestation report containing a cryptographic measurement of the workload: the code, initial memory contents, and firmware. This measurement is unique to this exact version of this exact workload on this exact TEE platform.

2. **Attestation verification** — the attestation report is sent to a verifier: either Strata's own attestation service (self-hosted) or a third-party service (AMD KDS, Intel PCS). The verifier checks the hardware signature and confirms the measurement matches the expected value registered when the workload was approved. **In the highest-trust deployments, the attestation verifier is customer-controlled** — Strata provides the software, the customer operates it. This ensures that a compromised Strata employee cannot register a backdoored measurement as valid.

3. **Sealed key release** — only after successful attestation does the key management system (HashiCorp Vault with TEE plugin, or Azure mHSM) release the decryption key for the workload's data. If the measurement does not match — because the workload code has been tampered with or the TEE firmware is outdated — the key is withheld and the workload cannot start.

4. **Operator-blind inference** — the model is encrypted at rest with a key sealed to a specific TEE measurement. The model owner publishes the expected measurement. The infrastructure operator runs the cluster and can observe that inference is happening and how long it takes — but cannot read the weights or the patient inputs. This protects both patient PHI and third-party model IP simultaneously.

### Confidential compute in the Strata Harness

The harness is attestation-aware. When an agent is deployed into a confidential container, the harness records the expected TEE measurement in the agent registry alongside the model version and skill set. The audit ledger entry for every inference includes the attestation status of the executing container — whether it ran in a verified TEE, which TEE platform, and whether the measurement matched the registered expected value. Attestation status is part of the compliance record, not a separate out-of-band check.

**PHI in telemetry:** the operator-blind boundary depends on the telemetry pipeline being PHI-free. If a workload name or a trace attribute leaks a patient identifier into OpenTelemetry, the boundary collapses. The telemetry schema enforces PHI exclusion, and this is validated by the same L1 input classifier used in the clinical harness — applied to outbound telemetry attributes before they leave the confidential boundary.

---

## 7. Agentic management and operations

The harness that governs AI agents running clinical workloads is the same harness that governs the agents managing the infrastructure itself. Operations agents — capacity planners, upgrade orchestrators, anomaly detectors, incident responders — run inside the Strata Harness with the same skill registry, the same guardrail engine, the same audit ledger, and the same human escalation queue as any other agent.

The infrastructure team is the clinical team. The cluster is the patient.

> **The infrastructure operator does not trust the operations agent any more than the clinician trusts the diagnostic agent. Both operate within a defined scope, under enforced guardrails, with every decision recorded and every destructive action requiring human approval.**

### Operations agent roles

| Agent | Responsibilities | Autonomous scope |
|---|---|---|
| **Capacity planner** | Watches resource utilization trends. Predicts demand using historical patterns and scheduled workload metadata. Proposes node additions, MIG repartitioning, or volume expansions before headroom is exhausted. | Generate proposals, open tickets. Never purchase hardware or modify production node config. |
| **Anomaly detector** | Correlates OpenTelemetry metrics and traces across nodes and workloads. Detects unusual patterns — latency spikes, unexpected restarts, guardrail violation rate increases, Raft election anomalies — with causal context. | Create alerts, label root cause candidates. Never take cluster-level action. |
| **Upgrade orchestrator** | Plans and executes rolling upgrades. Selects order based on Raft voter placement and stateful workload co-location. Monitors health gates between nodes. Automatically rolls back if post-upgrade error rate exceeds threshold. | Plan, drain, restart agent binary, verify re-attach. Escalates: any rollback, any voter that fails to rejoin within SLA. |
| **Incident responder** | First responder to infrastructure alerts. Runs defined runbooks autonomously — collect logs, check audit ledger, test connectivity, identify affected workloads. Produces a structured incident report within minutes. | Read-only investigation, log collection, runbook execution up to first decision point. Escalates: any remediation that modifies running workloads. |
| **Cost optimizer** | Identifies underutilized resources — idle MIG instances, over-provisioned memory limits, low-I/O volumes. Proposes right-sizing with projected impact. Tracks whether accepted proposals delivered projected savings. | Generate proposals with projected savings. Escalates: any change to running workload resource limits. |
| **Security auditor** | Continuously audits cluster state against policy. Detects drift — weakened namespace isolation, skill artifacts with invalid signatures, expiring SPIFFE certificates, TEE measurements that differ from registered expected values. | Detect, report, open compliance ticket. Escalates: any active policy violation immediately, at Critical SLA. |

### Operations guardrail boundaries

Every operations agent has a hard boundary between what it can do autonomously and what it must escalate. The boundary is defined in the harness skill registry and enforced by the guardrail engine — the same L1–L5 enforcement chain as clinical agents:

| Autonomy level | Permitted actions |
|---|---|
| **Always autonomous** | Read cluster state · query audit ledger · collect logs · run read-only diagnostic skills · generate proposals · send notifications · open escalation events |
| **Autonomous with rate limit** | Restart a crashed process (re-attach semantics only) · rotate expiring certificates · rebalance load between healthy nodes · garbage-collect completed job artifacts |
| **Requires escalation approval** | Drain a node · modify a running workload's resource limits · execute a rollback · change network policy rules · repartition MIG instances · modify tenant namespace isolation config |
| **Never autonomous** | Delete a namespace · remove a Raft voter permanently · modify the audit ledger · change the harness guardrail policy · access workload data or PHI for any operational purpose |

### Delivery model — self-hosted or as a service

**Self-hosted operations agents** — operations agents deploy as first-class workloads inside the Strata cluster they manage. They are defined in the agent registry like any other agent, with approved skills and guardrail profiles. They authenticate to the cluster API using SPIFFE workload identity scoped to the operations namespace. Who chooses this: hospitals and research institutions that require all management operations to remain within their network boundary, air-gapped environments, organizations with regulatory requirements that prohibit external management plane access.

**Strata Ops (managed service)** — operations agents run in Strata's cloud, connected to the customer cluster via an outbound-only secure channel (no inbound ports opened on the cluster). The channel is mTLS-authenticated with SPIFFE, and the operations agent's access is scoped to read-only cluster telemetry and the harness escalation queue API — it cannot directly call compute or storage backends. Who chooses this: organizations that want operational intelligence without maintaining the operations agents themselves. The service handles upgrades of the operations agents, tuning of anomaly detection models, and 24/7 escalation coverage.

**Data boundary for Strata Ops:** the managed service sees telemetry (metrics, traces, node health) and escalation event metadata. It never sees PHI, workload data, or audit ledger contents — those remain inside the customer's cluster boundary, enforced by the same scoped SPIFFE credentials that govern all other cross-boundary access.

### Operations as a compliance asset

In regulated environments, demonstrating that infrastructure changes are governed and auditable is as important as demonstrating that clinical AI outputs are governed. Strata Ops produces the same artifacts for infrastructure operations as the harness produces for clinical agent invocations: every agent action — every log collection, every proposal, every escalation, every approved remediation — is written to the audit ledger with the same append-only, hash-chained guarantees. A regulator asking "who changed the network isolation policy and when" receives the same answer as "who approved the dosage recommendation and when": a tamper-evident record with reviewer identity, timestamp, and a cryptographic hash of the action.

---

## 8. Hyperconverged control plane — no dedicated machines

Traditional control planes require dedicated machines separate from the workload fleet. This creates a circular upgrade problem: upgrading the control plane risks disrupting the workloads it manages, and upgrading workload nodes requires the control plane to be healthy. For stateful services such as a clinical database, this is unacceptable.

Strata breaks this dependency through two invariants:

1. The control plane is embedded in `strata-agent` — the same binary that runs on every node. Three to five nodes participate in Raft consensus as voters; the rest are learners.
2. The executor that runs workloads reads from a **local RocksDB WAL**, not from the network. A workload, once running, continues running through any agent restart, upgrade, or loss of Raft quorum.

> **The executor reads from local disk, not the network. That single property is what breaks the liveness coupling between the control plane and your database.**

### Safe rolling upgrade sequence

```
Step 1  Mark node Draining — scheduler stops new placements
Step 2  Transfer Raft leadership to another voter (if applicable) · <500ms
Step 3  Demote to learner via joint consensus — quorum maintained throughout
Step 4  systemctl restart strata-agent (new binary) · agent restarts in ~2s
Step 5  Executor reads RocksDB WAL · scans running workloads
        observed == desired → no action taken on database
Step 6  Rejoin as learner · catch up from last applied index · re-promote
Step 7  Node marked Active · scheduler resumes placement
        Database: uninterrupted throughout all 7 steps
```

### Autonomous mode — surviving quorum loss

If Raft quorum is lost, the executor switches to **autonomous mode**. It continues running existing workloads and healing crashed processes, but freezes desired-state reconciliation entirely — it will not act on deletions or changes until quorum is restored. This prevents a split-brain scenario from terminating a running database.

---

## 9. Storage and networking

### Storage

- **Local NVMe** — exposed via SPDK vhost-user target, controlled through SPDK's JSON-RPC socket. For VMs: virtio-blk or NVMe-over-vhost-user. For containers and processes: direct bind-mount with io_uring. Per-tenant I/O QoS enforced via blk-mq cgroups — each tenant namespace has its own IOPS and bandwidth ceiling.
- **Distributed block** — Linstor with DRBD for synchronous replication. Replication path stays in kernel space. The agent drives Linstor via its REST API.
- **Model weights** — JuiceFS mounted read-only, backed by S3-compatible object storage, with aggressive local SSD caching. For confidential inference workloads, model weight objects are encrypted with keys sealed to the TEE measurement.

### Networking

The default data plane is eBPF/XDP via Cilium's dataplane library — fast path stays in kernel, no Kubernetes required. Per-tenant network policy is enforced in the XDP layer: tenant namespaces are assigned distinct VXLAN VNIs, and cross-namespace traffic is denied by default with explicit allow-list rules for approved inter-tenant service calls.

For GPU clusters requiring low-latency communication, the agent configures RoCEv2 with rdma-core for control and ibverbs for the data path, including GID table setup and QP management. SR-IOV is supported for workloads requiring dedicated NIC VFs — the agent enumerates VFs via sysfs, binds them to `vfio-pci`, and passes PCI addresses to cloud-hypervisor's device assignment API.

---

## 10. Accelerator subsystem

Strata treats GPUs, NPUs, and CPU acceleration features as first-class resources with inventory, scheduling, and lifecycle management on par with memory and CPU cores. The agent discovers hardware at startup — NVML for Nvidia, ROCm SMI for AMD, `cpuid` for CPU features — and publishes the inventory to the control plane.

- **Nvidia MIG** — profiles configured via NVML and exposed as independent schedulable resources. A single A100 can be partitioned into up to seven MIG instances, each hardware-isolated from the others and assignable to a distinct tenant namespace.
- **GPU SR-IOV** — for GPUs that support SR-IOV, each tenant can be assigned a dedicated GPU VF with hardware-enforced memory isolation. No shared compute context, no cross-VF cache side channels.
- **GPU topology** — NVLink graph and PCIe bandwidth from NVML's topology API drive co-location decisions for peer-to-peer workloads.
- **CPU inference** — Intel AMX and AVX-512 VNNI detected and surfaced as node labels. ARM SVE2 detected on ARM nodes. The scheduler targets CPU-based inference to the right hardware without operator intervention.

---

## 11. Why healthcare first

Healthcare is simultaneously the highest-stakes and most infrastructure-constrained domain in the world. Hospitals operate at the intersection of three forces no existing platform handles well together: regulatory compliance (HIPAA, GDPR, FDA 21 CFR Part 11, EU AI Act), the physical reality of intermittent or absent cloud connectivity in clinical settings, and rapidly growing demand for real-time AI inference at the point of care.

| Metric | Figure |
|---|---|
| Global healthcare IT spend | $390B projected by 2028 |
| Top AI adoption barrier | 78% of hospitals cite infrastructure complexity |
| FDA-cleared AI/ML devices | 6,000+ as of 2025, growing 40% year-on-year |

### Primary use cases

| Use case | How Strata enables it |
|---|---|
| **Radiology AI** | Inference on GPU-backed VMs or confidential containers with model weights sealed to a TEE measurement. Sub-second latency, fully air-gapped, model IP protected from the operator. |
| **Genomics pipelines** | I/O-intensive bioinformatics on SPDK NVMe with NUMA-pinned CPU processes, per-namespace I/O QoS preventing research pipelines from starving clinical workloads. |
| **Clinical edge** | Single-node Strata on a clinic workstation. Runs confidential containers and a local LLM for clinical summarization with operator-blind PHI handling. No internet required. |
| **Research clusters** | Multi-node GPU clusters at academic medical centers. Federated learning across institutions over RoCEv2 fabric. Peer-federated harness policy for cross-institution agent governance. |
| **Multi-tenant hospital platform** | Cardiology, radiology, pharmacy, and third-party vendors all on the same cluster with hardware-enforced namespace isolation. A vendor's agent cannot enumerate another vendor's approved skills. |

---

## 12. Strata Harness — the trust fabric

When a hospital deploys an agentic system, the infrastructure question is only half the problem. The other half is trust: who approved this skill, what can this agent decide autonomously, what must it escalate, and how do we prove to a regulator that every inference was within bounds?

Strata Harness answers those questions. It sits above the compute fabric and below the clinical application. It owns the agent lifecycle, the skill and tool registry, the guardrail enforcement pipeline, the inference routing policy, the human escalation queue, and the compliance audit trail — all running locally, with no patient data leaving the facility.

> **The harness is not a deployment tool. It is a trust fabric. It is what makes it safe to give an AI agent access to a hospital's systems — not by trusting the model, but by enforcing the envelope around it.**

| Component | Role |
|---|---|
| **Agent registry** | Versioned, signed agent definitions — model, skill set, tool bindings, guardrail profile, data access scope. Every agent is an immutable artifact deployed from registry; never run ad-hoc. |
| **Skill registry** | Discrete approved capabilities — drug interaction lookup, FHIR query, imaging classifier, note summarizer. Each skill is independently versioned, approved, and scoped. Namespace-scoped — tenants cannot enumerate each other's skills. |
| **Tool bindings** | System integrations — EHR API, PACS, lab systems. Bound at deploy time with least-privilege SPIFFE credentials. Revocable independently of the agent. |
| **Guardrail engine** | Five-layer enforcement: input classification, scope check, clinical boundary rules, output validation, escalation triggers. Enforced in-process — no network round-trip, no model override possible. |
| **Inference router** | Selects model and hardware target per request based on data classification and attestation requirements. PHI-tagged requests never routed off-cluster. Requests requiring confidential compute routed only to verified TEE nodes. |
| **Audit ledger** | Append-only, tamper-evident log of every agent invocation including TEE attestation status. Namespace-partitioned. Queryable for regulatory reporting without manual assembly. |

---

## 13. Deploying agentic systems — the hospital use case

A hospital deploys a diagnostic support agent into a confidential container. The agent can query a patient's medication list, run a drug interaction check, invoke a radiology classifier, and produce a clinical summary — but it must not issue a diagnosis autonomously, must not recommend a drug dosage without a clinician in the loop, and must never route patient data outside the facility. The confidential container ensures that even the infrastructure team cannot read the PHI during inference.

### Skill and tool approval pipeline

| Stage | Actor | Action |
|---|---|---|
| Submit | Provider | Skill artifact + regulatory attestation submitted to harness registry |
| Safety scan | Automated | Output analysis, hallucination rate, known failure mode check |
| Clinical review | Hospital informatics team | Scope review — not safety re-evaluation if third-party attested |
| Sign & publish | Harness | Versioned, signed, scoped artifact committed to registry |
| Active | All agents | Skill available to agents with appropriate scope binding |

Third-party providers with existing FDA clearance, CE marking, or hospital vendor approval submit a pre-attested skill bundle. The harness carries the attestation record alongside the artifact, reducing the hospital's review burden to a scope check rather than a safety re-evaluation.

### Active skill registry — example

| Skill | Provider | Approval | Data scope |
|---|---|---|---|
| Drug interaction check v2.1.0 | Strata | Hospital approved | Med list only |
| Chest X-ray classifier v1.4.2 | Rad-AI Inc. | FDA 510(k) | DICOM images — confidential container required |
| Clinical note summarizer v3.0.1 | MedScribe Co. | Hospital approved | PHI — local only, TEE attestation required |
| FHIR patient query v1.0.0 | Strata | Hospital approved | PHI — audit required |
| Lab result interpreter v0.9.0 | Internal | Pending review | Lab data only |

### Inference routing

| Routing target | Applicable requests | Conditions |
|---|---|---|
| **On-cluster standard** | Non-PHI queries, administrative summarization | Data class = non-PHI |
| **On-cluster TEE** | PHI-tagged requests, clinical decision support, radiology inference with third-party models | Data class = PHI or model requires IP protection. Routes only to nodes with verified TEE. Attestation status logged. |
| **External (allowlisted only)** | De-identified research queries | Data class = non-PHI; BAA in place; explicit policy rule required |

---

## 14. Guardrail engine — layered enforcement

Guardrails are not prompts. They are enforced in the harness runtime, in-process on the node running the agent. The model cannot override them. A guardrail violation does not produce a model output — it produces a structured exception that goes to the audit ledger and, if configured, to the human escalation queue.

| Layer | Name | What it checks | When it runs |
|---|---|---|---|
| L1 | Input classification | Data sensitivity (PHI, PII, de-identified) and clinical context. Gates which models, skills, and TEE tier are eligible. | Before any model or skill is invoked |
| L2 | Scope enforcement | Agent's declared skill set and tool bindings vs. the request. Structural check — independent of model behavior. | Before each tool call and skill invocation |
| L3 | Clinical boundary rules | Domain-specific hard rules: no autonomous diagnosis, no autonomous dosage recommendation, consent checks. Stored as signed policy objects. | Against both inputs and candidate outputs |
| L4 | Output validation | Fast local classifier (sub-100ms): hallucinated drug names, contradictions with structured context, confidence below threshold. Classifier is itself a versioned, approved skill registry artifact. | On every model completion before output reaches caller |
| L5 | Escalation & circuit breaker | Routes borderline cases to human escalation queue. Hard-stops policy violations. Auto-suspends agents exceeding violation rate threshold. | On policy violations — hard stops are instantaneous |

### A normal invocation — trace

```
00:00.000  AUDIT  session=a3f9 user=dr.chen agent=diagnostic-support-v2.1
00:00.001  GUARD  L1 input classification → PHI=true context=diagnostic tee_required=true
00:00.002  ROUTE  inference_target=on-cluster-tee model=clinical-llm-v4.2 node=gpu-node-03
00:00.002  ATTEST attestation=verified tee=amd-sev-snp measurement=e3f9a1... ✓
00:00.003  GUARD  L2 scope check → skill=drug-interaction-check ✓
00:00.041  SKILL  drug-interaction-check(metoprolol, lisinopril, heparin) → 0 critical interactions
00:00.042  AUDIT  skill_call logged result_hash=9f3a2c
00:00.043  GUARD  L3 boundary check → no diagnosis assertion in output candidate ✓
00:00.387  INFER  model inference complete 343ms · 1,204 tokens · clinical-llm-v4.2
00:00.388  GUARD  L4 output validation → confidence=0.91 no hallucinated drug names ✓
00:00.390  AUDIT  response delivered output_hash=7b1e4d latency=390ms tee=verified guardrails=5
```

### A guardrail stop — trace

```
00:00.387  INFER  model inference complete · clinical-llm-v4.2
00:00.388  GUARD  L4 output validation → drug dosage recommendation detected in output
00:00.388  BLOCK  L3 boundary violation → autonomous dosage recommendation outside approved scope
00:00.389  AUDIT  VIOLATION logged rule=no-autonomous-dosage severity=HIGH session=a3f9
00:00.389  ESCAL  escalation opened → queue=clinical-review SLA=15min notify=informatics-oncall
00:00.390  RESP   caller receives structured_error code=CLINICAL_BOUNDARY_EXCEEDED
```

---

## 15. Human escalation queue

When a guardrail fires, the model's output is stopped — but the clinical situation that prompted the query has not stopped. The escalation queue bridges that gap. It routes the event to the right human, with complete context, within a time-bound SLA, and records what the human decided and why.

It is not an alerting system. It is a **structured decision workflow**: every escalation event is an atomic unit of clinical governance with an owner, a deadline, a set of permissible responses, and an immutable resolution record.

### Content provided to every reviewer

Every field is populated automatically by the harness at the moment of guardrail trigger — the reviewer never queries another system:

- Agent identity, version, and TEE attestation status
- Requesting clinician identity and department
- Patient reference (anonymised above department level)
- Guardrail layers triggered, with rule names and severity
- The original query text
- The blocked model output (not shown to the clinician)
- Structured context provided to the agent (vitals, medication list, etc.)
- SLA countdown with secondary escalation path on miss

### Reviewer response options

| Response | Output state | Audit record | Policy effect |
|---|---|---|---|
| **Approve & release** | Delivered unchanged | Reviewer identity, timestamp, notes, output hash | None. One-time session override. |
| **Modify & release** | Reviewer edits; modified version delivered | Original hash + modified hash + full diff | None. Diff permanently archived. |
| **Reject** | Not delivered; clinician receives structured error | Rejection reason, reviewer identity, timestamp | None. |
| **Flag for policy review** | Rejected for now | Policy review ticket opened, linked to this event | Opens a formal policy amendment workflow. Guardrail rule updated if approved. |

### SLA configuration

| Severity | Primary SLA | Primary assignee | On SLA miss |
|---|---|---|---|
| Critical — patient safety | 5 minutes | On-call attending | Dept. chief + compliance incident |
| High — clinical boundary | 15 minutes | Clinical informatics on-call | Pharmacy / specialty on-call |
| Medium — confidence low | 2 hours | AI governance team | CMIO daily digest |
| Low — policy advisory | 24 hours | Clinical informatics team | Weekly governance report |

---

## 16. Federated harness policy

A health system is not a hospital. Large systems operate dozens of hospitals across multiple states, each with its own credentialing, formularies, and policy requirements. Federated policy defines an inheritance hierarchy, a controlled propagation protocol, and a clear separation between what system leadership mandates and what individual sites configure.

> **Policy is inherited downward and cannot be weakened. It can only be strengthened or specialized at lower levels. This invariant is enforced by the runtime — it is not convention.**

### Three-tier hierarchy

| Tier | Actor | Authority | What they configure |
|---|---|---|---|
| Tier 1 — Health system | e.g. UC Health | Mandatory floor — cannot be overridden | System-wide guardrail rules, TEE requirements, approved skill baseline, cross-site compliance reporting |
| Tier 2 — Hospital | e.g. UCSF Medical Center | Can restrict, not relax | Site-specific guardrail additions, skill scope opt-in, escalation SLA and reviewer config, namespace isolation tier minimums |
| Tier 3 — Department | e.g. Cardiology | Can restrict within site scope only | Specialty guardrail rules, agent config, local escalation routing |

### Policy propagation — observe, warn, enforce

| Phase | Behaviour | Promotion trigger |
|---|---|---|
| **Observe (shadow)** | Violations logged but not blocked. Teams see impact before enforcement. | System governance reviews shadow report and signs promotion |
| **Warn** | Violations allowed through with warning appended. Departments can adapt. | System CMIO signs enforcement promotion — not automatic |
| **Enforce** | Full enforcement. Violations block output and trigger escalation queue. | Rollback takes effect within one Raft heartbeat (~2 seconds) |

### Cross-system federation — research consortia

Some deployments extend beyond a single health system. Strata supports a **peer federation model**: institutions negotiate a shared policy namespace governed by unanimous consent. No member can push a rule into another's harness unilaterally. Shared reporting shows aggregate compliance metrics only — individual patient events never cross the institutional boundary.

---

## 17. Compliance as a first-class output

| Requirement | Standard | How harness satisfies it | Artifact |
|---|---|---|---|
| PHI never leaves facility | HIPAA | Inference router enforces data classification at runtime; TEE routing enforces confidential compute requirement | Route log + attestation record |
| Audit trail of AI outputs | HIPAA · 21 CFR 11 | Append-only ledger, output hash, user identity, model version, TEE attestation status on every call | Audit ledger |
| Human oversight of clinical AI | FDA AI/ML · EU AI Act | Clinical boundary rules + mandatory escalation for diagnosis assertions | Escalation log |
| Software version traceability | 21 CFR 11 | Every invocation logs model, skill, agent, harness, and TEE firmware version | Version manifest |
| Validated software changes | 21 CFR 11 | Approval pipeline with signed attestations, TEE measurement registration, canary rollout, auto-rollback | Approval record |
| Access control | HIPAA | SPIFFE workload identity, per-session consent check, namespace-scoped RBAC | Access log |
| Operator-blind PHI processing | HIPAA minimum-necessary | Confidential containers with sealed keys; operator sees only ciphertext | TEE attestation record |
| Incident response evidence | EU AI Act · HIPAA | Full session replay; tamper-evidence via hash chain; TEE measurement at time of incident | Session replay |
| Infrastructure change governance | Internal · audit | Operations agent actions in same audit ledger as clinical actions — same hash chain, same tamper-evidence | Ops audit ledger |

---

## 18. Provider ecosystem

Strata Harness defines an open **provider SDK** — a specification for packaging a skill or model as a signed, self-attesting artifact that carries its own safety documentation, test results, regulatory status, and — where applicable — expected TEE measurement for confidential deployment.

The hospital's compliance team reviews **scope and integration** — not the underlying model safety, which travels with the attestation. A provider whose model runs inside a confidential container can include the expected TEE measurement in the attestation bundle. The hospital's harness verifies the measurement at deployment time and at every invocation. This compresses the time from regulatory clearance to hospital deployment from months to days, while giving the hospital a stronger guarantee than a vendor attestation letter alone.

The attestation schema is the key network effect. If Strata defines the open standard for how a clinical AI provider packages regulatory documentation as a machine-readable artifact, then every hospital using Strata benefits from every provider approval.

### Provider SDK components

- **Open spec** — signed artifacts with embedded attestations. Rust and Python libraries. Versioned, reproducible builds.
- **Attestation schema** — regulatory status, validation dataset, known failure modes, contraindicated contexts, TEE measurement (optional). Machine-readable. Travels into every audit ledger.
- **Marketplace** — curated registry of approved providers. One-click pull into any Strata Harness instance.
- **Canary rollout** — new skill versions roll to 5% of traffic automatically. Automated rollback on guardrail violation rate increase. Hospital approval required to reach 100%.

---

## 19. Community and governance

Strata's growth strategy is modeled on Kubernetes: seed the community with a working, opinionated v0, release it early, and let practitioners shape v1. We donate the compute fabric core to CNCF under a Sandbox proposal and the agentic infrastructure primitives — the harness protocol, the provider SDK, the attestation schema — to the Linux Agent Foundation. We retain the hosted Strata Cloud harness and Strata Ops as commercial products.

We seed the community by open-sourcing the core before our Series A, partnering with three to five academic medical centers as design partners, and presenting at KubeCon and at AMIA and HIMSS in year one.

> **We are not building a walled garden. We are building the commons — and building a business on top of the commons by being the best operators of it.**

---

## 20. Roadmap

| When | Milestone | Scope |
|---|---|---|
| **Now** | v0 — core fabric | Embedded Raft control plane. Process and container backends. Local NVMe via SPDK. eBPF networking with per-tenant network policy. Zero-downtime rolling upgrades. SPIFFE identity. Namespace isolation (cgroups v2 + Linux namespaces). CLI and gRPC API. Harness: skill registry, guardrail engine L1–L4, audit ledger, inference router. |
| **Q4 2026** | v1 — community release | VM backend. Distributed block storage (Linstor). Confidential containers (kata-cc + AMD SEV-SNP). GPU support — Nvidia MIG, AMD ROCm. Hardware SR-IOV isolation. Remote attestation service. Kubernetes mode. CNCF Sandbox submission. Harness: human escalation queue, provider SDK v1, three medical design partners in production. |
| **H1 2027** | v1.5 — harness and ops GA | Strata Cloud multi-tenant harness: federated policy, fleet upgrades, compliance reporting, HIPAA audit trail export, workload marketplace. Strata Ops managed service: all six operations agents, self-hosted and SaaS modes. Intel TDX support. Enterprise support tier. First paid customers. |
| **H2 2027** | v2 — intelligence fabric | Federated learning primitives. Inference-optimized scheduling (disaggregated prefill/decode). Confidential inference with operator-blind model weights at scale. ARM SVE2 and Intel AMX first-class. Edge form factors — NVIDIA Jetson, Apple Silicon, Qualcomm. Multi-institution peer-federated research clusters. Provider SDK donated to Linux Agent Foundation. |

---

## 21. Why now

Three forces are converging:

1. **The inference era** is making GPU and accelerator management, multi-tenant isolation, and confidential compute first-class infrastructure problems. Every organization running clinical AI needs to solve placement, isolation, utilization, and attestation simultaneously — and no open platform does this today.

2. **Regulatory pressure** — EU AI Act, FDA AI/ML guidance, state-level AI-in-medicine legislation — is pushing healthcare organizations toward auditable, on-premise, operator-blind infrastructure that cloud-only solutions cannot provide. Confidential compute is moving from a niche capability to a compliance requirement.

3. **The Rust systems ecosystem** has matured to the point where a production-grade stack — Raft, NVMe userspace I/O, eBPF, VMM, TEE integration, agentic runtime — can be built and maintained by a small team with high confidence in correctness, memory safety, and performance.

The window to define the open standard for AI-era infrastructure is open. Kubernetes defined the container scheduling standard in 2016. Strata intends to define the compute fabric, multi-tenant isolation, confidential compute, and agentic trust standard for the inference era — one owned by the community that runs it, one that extends from a clinic edge node to a thousand-GPU research cluster, and one where the compliance audit trail, the attestation record, and the operations governance log are built in from day one.

---

*Strata · Vision Paper v0.3 · June 2026*
*OSS core → CNCF · Agentic primitives → Linux Agent Foundation · Harness + Ops → commercial products*
*Pre-publication draft — confidential*
