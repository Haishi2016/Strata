# Strata

**A unified, open-source compute fabric and agentic harness for the intelligence era**

> Vision Paper · v0.2 · June 2026  
> OSS-first · CNCF track · Healthcare vertical launch  
> *Pre-publication draft — confidential*

---

## Table of contents

1. [The problem](#1-the-problem)
2. [Our answer: Strata](#2-our-answer-strata)
3. [Guiding principles](#3-guiding-principles)
4. [Architecture: the compute fabric](#4-architecture-the-compute-fabric)
5. [Hyperconverged control plane — no dedicated machines](#5-hyperconverged-control-plane--no-dedicated-machines)
6. [Storage and networking](#6-storage-and-networking)
7. [Accelerator subsystem](#7-accelerator-subsystem)
8. [Why healthcare first](#8-why-healthcare-first)
9. [Strata Harness — the trust fabric](#9-strata-harness--the-trust-fabric)
10. [Deploying agentic systems — the hospital use case](#10-deploying-agentic-systems--the-hospital-use-case)
11. [Guardrail engine — layered enforcement](#11-guardrail-engine--layered-enforcement)
12. [Human escalation queue](#12-human-escalation-queue)
13. [Federated harness policy](#13-federated-harness-policy)
14. [Compliance as a first-class output](#14-compliance-as-a-first-class-output)
15. [Provider ecosystem](#15-provider-ecosystem)
16. [Community and governance](#16-community-and-governance)
17. [Roadmap](#17-roadmap)
18. [Why now](#18-why-now)

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

Strata runs as a single binary — `strata-agent` — on every node. A subset of nodes (three to five) participate in a Raft consensus cluster that replicates desired state. All other nodes are learners: they receive replicated state without voting. The entire system is written in Rust, which provides the memory safety guarantees and systems-level performance required for infrastructure that runs clinical workloads.

Every resource — a compute workload, a storage volume, a network attachment, an agent deployment — is modeled as a declarative spec with a typed reconciler that drives actual state toward desired state.

```
┌─────────────────────────────────────────────────────────┐
│  Strata Harness (SaaS)                                  │
│  Multi-tenant orchestration, policy, compliance, audit  │
├─────────────────────────────────────────────────────────┤
│  Control plane                                          │
│  Embedded Raft · declarative reconciler · scheduler     │
├─────────────────────────────────────────────────────────┤
│  Compute                                                │
│  Process · OCI container · VM — one spec, three backends│
├─────────────────────────────────────────────────────────┤
│  Storage                                                │
│  NVMe (SPDK) · distributed block (Linstor) · JuiceFS   │
├─────────────────────────────────────────────────────────┤
│  Networking                                             │
│  eBPF/XDP (Cilium) · SR-IOV · RoCEv2 RDMA              │
├─────────────────────────────────────────────────────────┤
│  Accelerators                                           │
│  Nvidia MIG · AMD ROCm · Intel AMX · ARM SVE2           │
└─────────────────────────────────────────────────────────┘
```

### Compute backends

A single `ComputeSpec` selects the backend at deploy time:

- **Process** — cgroups v2, Linux namespaces for lightweight isolation. Suitable for trusted internal services where startup latency matters.
- **Container** — containerd over ttrpc API. EROFS snapshotter for read-only inference model layers (fast startup, no copy-on-write overhead).
- **VM** — cloud-hypervisor for general workloads (~100ms boot, VFIO GPU passthrough), Firecracker for microVM isolation in multi-tenant contexts.

### Open source component map

| Layer | Component | Role |
|---|---|---|
| State store | TiKV | Distributed KV, horizontal scale, Rust client — no etcd memory floor |
| Containers | containerd | Industry-standard runtime, ttrpc interface, pluggable snapshotters |
| VMs | cloud-hypervisor | Rust VMM, fast boot, VFIO GPU passthrough, REST API |
| NVMe I/O | SPDK | Kernel-bypass NVMe via vhost-user, sub-microsecond latency |
| Distributed block | Linstor + DRBD | Synchronous replication, kernel-space path, simple ops model |
| Model weights FS | JuiceFS | S3-backed, aggressive local SSD cache, POSIX interface |
| Networking | Cilium / eBPF | XDP fast path, no Kubernetes dependency required |
| GPU management | NVML / ROCm SMI | MIG partitioning, topology queries, utilization telemetry |
| Identity | SPIFFE / SPIRE | Zero-trust workload identity, mTLS everywhere |
| Observability | OpenTelemetry | Traces, metrics, logs — vendor-neutral, runs on-cluster |

---

## 5. Hyperconverged control plane — no dedicated machines

Traditional control planes require dedicated machines separate from the workload fleet. This creates a circular upgrade problem: upgrading the control plane risks disrupting the workloads it manages, and upgrading workload nodes requires the control plane to be healthy. For stateful services such as a clinical database, this is unacceptable.

Strata breaks this dependency through two invariants:

1. The control plane is embedded in `strata-agent` — the same binary that runs on every node. Three to five nodes participate in Raft consensus as voters; the rest are learners.
2. The executor that runs workloads reads from a **local RocksDB WAL**, not from the network. A workload, once running, continues running through any agent restart, upgrade, or loss of Raft quorum.

> **The executor reads from local disk, not the network. That single property is what breaks the liveness coupling between the control plane and your database.**

### Safe rolling upgrade sequence

Upgrading a node with a running database follows this deterministic sequence. The database never pauses:

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

## 6. Storage and networking

### Storage

- **Local NVMe** — exposed via SPDK vhost-user target, controlled through SPDK's JSON-RPC socket. For VMs: virtio-blk or NVMe-over-vhost-user. For containers and processes: direct bind-mount with io_uring. blk-mq scheduler is configurable per volume (`mq-deadline` for mixed workloads, `none` for NVMe SSDs).
- **Distributed block** — Linstor with DRBD for synchronous replication. Replication path stays in kernel space. The agent drives Linstor via its REST API.
- **Model weights** — JuiceFS mounted read-only, backed by S3-compatible object storage, with aggressive local SSD caching. Model loaders use standard POSIX filesystem calls; no modification required.

### Networking

The default data plane is eBPF/XDP via Cilium's dataplane library — fast path stays in kernel, no Kubernetes required. For GPU clusters, the agent configures RoCEv2 with rdma-core for control and ibverbs for the data path, including GID table setup and QP management. SR-IOV is supported for workloads requiring dedicated NIC VFs — the agent enumerates VFs via sysfs, binds them to `vfio-pci`, and passes PCI addresses to cloud-hypervisor's device assignment API.

---

## 7. Accelerator subsystem

Strata treats GPUs, NPUs, and CPU acceleration features as first-class resources with inventory, scheduling, and lifecycle management on par with memory and CPU cores. The agent discovers hardware at startup — NVML for Nvidia, ROCm SMI for AMD, `cpuid` for CPU features — and publishes the inventory to the control plane.

- **Nvidia MIG** — profiles configured via NVML and exposed as independent schedulable resources. A single A100 can be partitioned into up to seven MIG instances.
- **GPU topology** — NVLink graph and PCIe bandwidth from NVML's topology API drive co-location decisions for peer-to-peer workloads.
- **CPU inference** — Intel AMX and AVX-512 VNNI detected and surfaced as node labels, allowing the scheduler to target CPU-based inference to the right hardware without operator intervention.

---

## 8. Why healthcare first

Healthcare is simultaneously the highest-stakes and most infrastructure-constrained domain in the world. Hospitals operate at the intersection of three forces no existing platform handles well together: regulatory compliance (HIPAA, GDPR, FDA 21 CFR Part 11, EU AI Act), the physical reality of intermittent or absent cloud connectivity in clinical settings, and rapidly growing demand for real-time AI inference at the point of care.

| Metric | Figure |
|---|---|
| Global healthcare IT spend | $390B projected by 2028 |
| Top AI adoption barrier | 78% of hospitals cite infrastructure complexity |
| FDA-cleared AI/ML devices | 6,000+ as of 2025, growing 40% year-on-year |

### Primary use cases

| Use case | How Strata enables it |
|---|---|
| **Radiology AI** | Inference on GPU-backed VMs with model weights served from JuiceFS local cache. Sub-second latency, fully air-gapped from cloud. |
| **Genomics pipelines** | I/O-intensive bioinformatics on SPDK NVMe with NUMA-pinned CPU processes. No cloud egress costs or data governance risks. |
| **Clinical edge** | Single-node Strata on a clinic workstation. Runs containers and a local LLM for clinical summarization. No internet required. |
| **Research clusters** | Multi-node GPU clusters at academic medical centers. Federated learning across institutions over RoCEv2 fabric. |

---

## 9. Strata Harness — the trust fabric

When a hospital deploys an agentic system, the infrastructure question is only half the problem. The other half is trust: who approved this skill, what can this agent decide autonomously, what must it escalate, and how do we prove to a regulator that every inference was within bounds?

Strata Harness answers those questions. It sits above the compute fabric and below the clinical application. It owns the agent lifecycle, the skill and tool registry, the guardrail enforcement pipeline, the inference routing policy, the human escalation queue, and the compliance audit trail — all running locally, with no patient data leaving the facility.

> **The harness is not a deployment tool. It is a trust fabric. It is what makes it safe to give an AI agent access to a hospital's systems — not by trusting the model, but by enforcing the envelope around it.**

| Component | Role |
|---|---|
| **Agent registry** | Versioned, signed agent definitions — model, skill set, tool bindings, guardrail profile, data access scope. Every agent is an immutable artifact deployed from registry; never run ad-hoc. |
| **Skill registry** | Discrete approved capabilities — drug interaction lookup, FHIR query, imaging classifier, note summarizer. Each skill is independently versioned, approved, and scoped. |
| **Tool bindings** | System integrations — EHR API, PACS, lab systems. Bound at deploy time with least-privilege SPIFFE credentials. Revocable independently of the agent. |
| **Guardrail engine** | Five-layer enforcement: input classification, scope check, clinical boundary rules, output validation, escalation triggers. Enforced in-process — no network round-trip, no model override possible. |
| **Inference router** | Selects model and hardware target per request based on data classification. PHI-tagged requests are never routed off-cluster — enforced by the runtime, not by convention. |
| **Audit ledger** | Append-only, tamper-evident log of every agent invocation. Queryable for regulatory reporting without manual assembly. |

---

## 10. Deploying agentic systems — the hospital use case

A hospital deploys a diagnostic support agent. It can query a patient's medication list, run a drug interaction check, invoke a radiology classifier, and produce a clinical summary — but it must not issue a diagnosis autonomously, must not recommend a drug dosage without a clinician in the loop, and must never route patient data outside the facility.

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
| Chest X-ray classifier v1.4.2 | Rad-AI Inc. | FDA 510(k) | DICOM images |
| Clinical note summarizer v3.0.1 | MedScribe Co. | Hospital approved | PHI — local only |
| FHIR patient query v1.0.0 | Strata | Hospital approved | PHI — audit required |
| Lab result interpreter v0.9.0 | Internal | Pending review | Lab data only |

### Inference routing — local first, never PHI off-site

| Routing target | Applicable requests | Conditions |
|---|---|---|
| **On-cluster (default)** | All PHI-tagged requests, clinical decision support, radiology inference, note summarization, drug interaction checking | Models served from Strata model store on local GPU nodes |
| **External (allowlisted only)** | De-identified research queries, administrative summarization, code generation for pipelines | Data class = non-PHI; destination in approved endpoint list; BAA in place. Never automatic — requires explicit policy rule. |

---

## 11. Guardrail engine — layered enforcement

Guardrails are not prompts. They are enforced in the harness runtime, in-process on the node running the agent. The model cannot override them. A guardrail violation does not produce a model output — it produces a structured exception that goes to the audit ledger and, if configured, to the human escalation queue.

| Layer | Name | What it checks | When it runs |
|---|---|---|---|
| L1 | Input classification | Data sensitivity (PHI, PII, de-identified) and clinical context. Gates which models and skills are eligible. | Before any model or skill is invoked |
| L2 | Scope enforcement | Agent's declared skill set and tool bindings vs. the request. Structural check — does not depend on model behavior. | Before each tool call and skill invocation |
| L3 | Clinical boundary rules | Domain-specific hard rules: no autonomous diagnosis, no autonomous dosage recommendation, consent checks. Stored as signed policy objects. | Against both inputs and candidate outputs |
| L4 | Output validation | Fast local classifier (sub-100ms): hallucinated drug names, contradictions with structured context, confidence below threshold. | On every model completion before output reaches caller |
| L5 | Escalation & circuit breaker | Routes borderline cases to human escalation queue. Hard-stops policy violations. Auto-suspends agents exceeding violation rate threshold. | On policy violations — hard stops are instantaneous |

### A normal invocation — trace

```
00:00.000  AUDIT  session=a3f9 user=dr.chen agent=diagnostic-support-v2.1
00:00.001  GUARD  L1 input classification → PHI=true context=diagnostic
00:00.002  ROUTE  inference_target=on-cluster model=clinical-llm-v4.2 node=gpu-node-03
00:00.003  GUARD  L2 scope check → skill=drug-interaction-check ✓
00:00.041  SKILL  drug-interaction-check(metoprolol, lisinopril, heparin) → 0 critical interactions
00:00.042  AUDIT  skill_call logged result_hash=9f3a2c
00:00.043  GUARD  L3 boundary check → no diagnosis assertion in output candidate ✓
00:00.387  INFER  model inference complete 343ms · 1,204 tokens · clinical-llm-v4.2
00:00.388  GUARD  L4 output validation → confidence=0.91 no hallucinated drug names ✓
00:00.390  AUDIT  response delivered output_hash=7b1e4d latency=390ms guardrails_passed=5
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

## 12. Human escalation queue

When a guardrail fires, the model's output is stopped — but the clinical situation that prompted the query has not stopped. The escalation queue bridges that gap. It routes the event to the right human, with complete context, within a time-bound SLA, and records what the human decided and why.

It is not an alerting system. It is a **structured decision workflow**: every escalation event is an atomic unit of clinical governance with an owner, a deadline, a set of permissible responses, and an immutable resolution record.

### Content provided to every reviewer

Every field is populated automatically by the harness at the moment of guardrail trigger:

- Agent identity and version
- Requesting clinician identity and department
- Patient reference (anonymised above department level)
- Guardrail layers triggered, with rule names and severity
- The original query text (what the clinician asked)
- The blocked model output (not shown to the clinician)
- Structured context that was provided to the agent (vitals, medication list, etc.)
- SLA countdown with secondary escalation path on miss

### Reviewer response options

| Response | Output state | Audit record | Policy effect |
|---|---|---|---|
| **Approve & release** | Delivered unchanged to clinician | Reviewer identity, timestamp, notes, output hash | None. One-time session override. |
| **Modify & release** | Reviewer edits output; modified version delivered | Original output hash + modified hash + full diff | None. Diff permanently archived. |
| **Reject** | Not delivered; clinician receives structured error | Rejection reason, reviewer identity, timestamp | None. Clinician may re-ask differently. |
| **Flag for policy review** | Rejected for now | Policy review ticket opened, linked to this event | Opens a formal policy amendment workflow. Guardrail rule updated if approved. |

The "flag for policy review" path is how guardrail policy evolves without informal workarounds. A clinician who legitimately needs a capability does not find a way around the guardrail — they trigger an escalation, a reviewer flags it for policy review, the clinical informatics team opens an amendment, and the capability is formally added to the approved scope. Every step is in the audit ledger.

### SLA configuration and secondary escalation

| Severity | Primary SLA | Primary assignee | On SLA miss |
|---|---|---|---|
| Critical — patient safety | 5 minutes | On-call attending | Dept. chief + compliance incident |
| High — clinical boundary | 15 minutes | Clinical informatics on-call | Pharmacy / specialty on-call |
| Medium — confidence low | 2 hours | AI governance team | CMIO daily digest |
| Low — policy advisory | 24 hours | Clinical informatics team | Weekly governance report |

Missed SLAs are themselves events — they trigger secondary escalation and surface in the compliance dashboard as unresolved escalations. Governance teams see aggregate queue trends without seeing individual patient contexts.

---

## 13. Federated harness policy

A health system is not a hospital. Large systems operate dozens of hospitals across multiple states, each with its own credentialing, formularies, and patient population — and therefore its own policy requirements. But they share a system-wide obligation: a guardrail approved at the system level should not be re-approved at each site, and a policy violation at one site should be visible to system governance.

> **Policy is inherited downward and cannot be weakened. It can only be strengthened or specialized at lower levels. This invariant is enforced by the runtime — it is not convention.**

### Three-tier hierarchy

| Tier | Actor | Authority | What they configure |
|---|---|---|---|
| Tier 1 — Health system | e.g. UC Health | Mandatory floor — cannot be overridden | System-wide guardrail rules, approved skill baseline, cross-site compliance reporting |
| Tier 2 — Hospital | e.g. UCSF Medical Center | Can restrict, not relax | Site-specific guardrail additions, skill scope opt-in, escalation SLA and reviewer config |
| Tier 3 — Department | e.g. Cardiology | Can restrict within site scope only | Specialty guardrail rules, agent configuration, local escalation routing |

### Policy inheritance rules

- **System floor** rules are cryptographically signed and verified by each harness node at startup. A hospital cannot delete or soften a system rule — the node refuses to start with a policy that fails signature verification.
- **Hospital additions** may only restrict further. Attempting to add a rule that relaxes a system-level constraint is rejected by the policy engine.
- **Department additions** are scoped to agents deployed in that department only. Dept. rules cannot reference skills or tools the site has not already approved.
- **Aggregation upward** — escalation events, violation counts, SLA metrics, and audit digests aggregate upward. System governance sees a unified view across all hospitals without seeing individual patient context.

### Policy propagation — observe, warn, enforce

A new system-level rule is never applied instantaneously fleet-wide. Strata propagates policy through a three-phase rollout with explicit sign-off required at each transition:

| Phase | Behaviour | Promotion trigger |
|---|---|---|
| **Observe (shadow)** | Violations logged but not blocked. Teams see what would have been caught before any enforcement impact. | System governance reviews shadow report and signs promotion |
| **Warn** | Violations allowed through, but a structured warning is appended and a low-severity escalation event is opened. | System CMIO signs enforcement promotion — not automatic |
| **Enforce** | Full enforcement. Violations block output and trigger the escalation queue. | Rollback to previous phase takes effect within one Raft heartbeat (~2 seconds) |

```
Day 0   POLICY  system signs new rule: no-aPTT-check-before-heparin-output v1.0
Day 0   PROP    rule replicated via Raft to all 6 hospital harness clusters
Day 0   PROP    phase=observe · shadow mode active across all sites
Day 1   SHADOW  UCSF: 3 would-have-been violations · logged · not blocked
Day 3   REVIEW  system governance promotes to warn
Day 5   CONFIG  UCSF cardiology updates agent config to include aPTT pre-check skill
Day 7   SIGN    system CMIO signs enforcement promotion · policy_id=7e3a9f
Day 7   PROP    phase=enforce · all 6 clusters · Raft commit latency 1.2s
Day 7   AUDIT   policy lifecycle complete · 7 days · 0 rollbacks
```

### Cross-system federation — research consortia

Some deployments extend beyond a single health system. A research consortium — five academic medical centers collaborating on a federated learning study — needs to share approved skills and coordinate policy without patient data crossing institutional boundaries and without any one institution having administrative authority over another's harness.

Strata supports this through a **peer federation model**. Institutions negotiate a shared policy namespace governed by unanimous consent. No member can push a rule into another's harness unilaterally. All changes follow the same observe → warn → enforce rollout across all members simultaneously. Shared reporting shows aggregate compliance metrics only — individual patient events never cross the institutional boundary.

---

## 14. Compliance as a first-class output

HIPAA, FDA 21 CFR Part 11, the EU AI Act, and emerging state-level AI-in-medicine regulations share a common requirement: auditability. The harness makes this tractable without requiring teams to manually assemble audit trails across disparate systems.

| Requirement | Standard | How harness satisfies it | Artifact |
|---|---|---|---|
| PHI never leaves facility | HIPAA | Inference router enforces data classification at runtime, not by convention | Route log |
| Audit trail of AI outputs | HIPAA · 21 CFR 11 | Append-only ledger, output hash, user identity, model version on every call | Audit ledger |
| Human oversight of clinical AI | FDA AI/ML · EU AI Act | Clinical boundary rules + mandatory escalation for diagnosis assertions | Escalation log |
| Software version traceability | 21 CFR 11 | Every invocation logs model, skill, agent, and harness version | Version manifest |
| Validated software changes | 21 CFR 11 | Approval pipeline with signed attestations, canary rollout, auto-rollback | Approval record |
| Access control | HIPAA | SPIFFE workload identity, per-session consent check, role-based tool binding | Access log |
| Incident response evidence | EU AI Act · HIPAA | Full session replay from audit ledger; tamper-evidence via hash chain | Session replay |

---

## 15. Provider ecosystem

Strata Harness defines an open **provider SDK** — a specification for packaging a skill or model as a signed, self-attesting artifact that carries its own safety documentation, test results, and regulatory status. Any provider can publish: a radiology AI company, a pharma informatics team, a research institution. Hospitals pull the artifact directly into their local harness.

The hospital's compliance team reviews **scope and integration** — not the underlying model safety, which travels with the attestation. This compresses the time from a vendor receiving regulatory clearance to a hospital deploying their capability inside an approved agentic workflow from months to days.

The attestation schema is the key network effect. If Strata defines the open standard for how a clinical AI provider packages regulatory documentation as a machine-readable artifact, then every hospital using Strata benefits from every provider approval. Network effects accrue around the standard — which is open — not around a proprietary marketplace.

### Provider SDK components

- **Open spec** for packaging skills as signed artifacts with embedded attestations. Rust and Python libraries. Versioned, reproducible builds verified by the harness.
- **Attestation schema** — standardized fields: regulatory status, validation dataset, known failure modes, contraindicated contexts. Machine-readable. Travels with the artifact into every audit ledger.
- **Marketplace** — curated registry of approved providers. One-click pull into any Strata Harness instance. Pricing transparent per invocation or flat license.
- **Canary rollout** — new skill versions roll to 5% of traffic automatically. Automated rollback on guardrail violation rate increase. Hospital approval required to reach 100%.

---

## 16. Community and governance

Strata's growth strategy is modeled on Kubernetes: seed the community with a working, opinionated v0, release it early, and let practitioners shape v1. We donate the compute fabric core to CNCF under a Sandbox proposal and the agentic infrastructure primitives — the harness protocol, the provider SDK, the attestation schema — to the Linux Agent Foundation. We retain the hosted Strata Cloud harness as the commercial product.

We seed the community by open-sourcing the core before our Series A, partnering with three to five academic medical centers as design partners, and presenting at KubeCon and at AMIA and HIMSS in year one. Design partners receive the platform free in exchange for public case studies and co-authored architecture contributions that feed directly into the CNCF Sandbox proposal.

> **We are not building a walled garden. We are building the commons — and building a business on top of the commons by being the best operators of it.**

---

## 17. Roadmap

| When | Milestone | Scope |
|---|---|---|
| **Now** | v0 — core fabric | Embedded Raft control plane. Process and container compute backends. Local NVMe via SPDK. eBPF networking. Zero-downtime rolling upgrades. SPIFFE identity. CLI and gRPC API. Runs on bare metal Linux. Harness: skill registry, guardrail engine, audit ledger — single hospital, no federation. |
| **Q4 2026** | v1 — community release | VM backend (cloud-hypervisor + Firecracker). Distributed block storage (Linstor). GPU support — Nvidia MIG, AMD ROCm. Kubernetes mode. CNCF Sandbox submission. Harness: human escalation queue, inference router, provider SDK v1, three medical design partners in production. |
| **H1 2027** | v1.5 — harness GA | Strata Cloud multi-tenant harness: federated policy (hierarchical and peer models), fleet upgrades, compliance reporting, HIPAA audit trail export, workload marketplace. Enterprise support tier. First paid customers. |
| **H2 2027** | v2 — intelligence fabric | Federated learning primitives. Inference-optimized scheduling (disaggregated prefill/decode). ARM SVE2 and Intel AMX first-class. Edge form factors — NVIDIA Jetson, Apple Silicon, Qualcomm. Multi-institution research cluster support. Provider SDK donated to Linux Agent Foundation. |

---

## 18. Why now

Three forces are converging:

1. **The inference era** is making GPU and accelerator management a first-class infrastructure problem — every organization running AI needs to solve placement, isolation, and utilization, and no open platform does it well today.

2. **Regulatory pressure** — EU AI Act, FDA AI/ML guidance, state-level AI-in-medicine legislation — is pushing healthcare organizations toward auditable, on-premise infrastructure that cloud-only solutions cannot provide.

3. **The Rust systems ecosystem** has matured to the point where a production-grade distributed systems stack — Raft, NVMe userspace I/O, eBPF, VMM, agentic runtime — can be built and maintained by a small team with high confidence in correctness and safety.

The window to define the open standard for AI-era infrastructure is open. Kubernetes defined the container scheduling standard in 2016. Strata intends to define the compute fabric and agentic trust standard for the inference era — one owned by the community that runs it, one that extends from a clinic edge node to a thousand-GPU research cluster, and one where the compliance audit trail is built in from day one rather than assembled by hand after the fact.

---

*Strata · Vision Paper v0.2 · June 2026*  
*OSS core → CNCF · Agentic primitives → Linux Agent Foundation · Harness → commercial product*  
*Pre-publication draft — confidential*
