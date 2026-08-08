# Strata

**The open platform for governing, connecting, and operating AI agents at scale — across any cloud, any edge, any infrastructure**

> Vision Paper · v0.4 · August 2026
> OSS-first · CNCF track · Linux Agent Foundation · Healthcare vertical launch
> *Pre-publication draft — confidential*

---

## Table of contents

1. [The problem](#1-the-problem)
2. [Our answer: Strata](#2-our-answer-strata)
3. [Guiding principles](#3-guiding-principles)
4. [Architecture: the agentic middle layer](#4-architecture-the-agentic-middle-layer)
5. [Intent descriptors — ambient workflow intelligence](#5-intent-descriptors--ambient-workflow-intelligence)
6. [Guardrail engine — layered enforcement](#6-guardrail-engine--layered-enforcement)
7. [Human escalation queue](#7-human-escalation-queue)
8. [Federated harness policy](#8-federated-harness-policy)
9. [Multi-tenancy — isolation at every layer](#9-multi-tenancy--isolation-at-every-layer)
10. [Confidential compute and trust](#10-confidential-compute-and-trust)
11. [Open standards stack](#11-open-standards-stack)
12. [Why healthcare first](#12-why-healthcare-first)
13. [Deploying agentic systems — the hospital use case](#13-deploying-agentic-systems--the-hospital-use-case)
14. [Provider ecosystem and marketplace](#14-provider-ecosystem-and-marketplace)
15. [Monetization](#15-monetization)
16. [Community and governance](#16-community-and-governance)
17. [Roadmap](#17-roadmap)
18. [Why now](#18-why-now)

---

## 1. The problem

Every organization deploying AI agents is independently reinventing the same missing layer.

They build bespoke registries to track what agents exist. They wire ad-hoc trust relationships between agents that need to talk. They maintain separate audit logs per agent system, assembled manually when a regulator or incident review asks what happened. They write one-off orchestration glue every time a new business workflow spans multiple agents. And when something goes wrong — a hallucinated output, a decision that should have been escalated, a policy violated at 2am — there is no coherent record of the cross-agent context, and no systematic path for the human who needs to respond.

The problem compounds with heterogeneity. A hospital runs agents from five vendors on three different infrastructure substrates — AWS Lambda, an on-premise GPU cluster, and edge appliances in clinic rooms. Each has its own deployment model, its own access control, its own notion of what a "guardrail" is. There is no unified operational lens across this estate. There is no governance layer that enforces the same policy everywhere. There is no way to see that the radiology agent, the drug interaction agent, and the clinical note summarizer are all participating in the same patient encounter — and to audit that encounter as a coherent unit.

Meanwhile individual agents are oblivious to the business workflows they participate in. A payment validation agent processes a transaction without knowing it is the third step in a high-priority enterprise order with a 17:00 SLA. A triage agent surfaces a recommendation without knowing that two upstream agents already attempted and deferred the same question. Agents make locally reasonable decisions that are globally suboptimal because they have no ambient awareness of the workflow they are part of.

> **The infrastructure layer is solved — or solvable. The missing layer is above it: a vendor-neutral platform that governs, connects, and operates a heterogeneous agent estate, and gives every agent enough context to act intelligently within the workflows it participates in.**

---

## 2. Our answer: Strata

Strata is the open platform for the agentic middle layer. It sits between infrastructure — which can be any cloud, any Kubernetes cluster, any edge device, any bare metal node — and the agents that organizations deploy on top of it. It does not run the agents. It governs, connects, and operates them.

Strata provides:

- A **universal agent and skill registry** — versioned, signed, OCI-packaged artifacts deployable to any node
- An **intent descriptor system** — structured workflow context propagated automatically to every agent participating in a business workflow, giving agents ambient intelligence about the goal, constraints, participants, and state of the work they are part of
- A **layered guardrail engine** — policy evaluated in-process on the node, with no network round-trip on the hot path, governed by declarative Rego rules version-controlled alongside the agents they govern
- A **fleet control plane** — GitOps-style declarative fleet state across cloud and edge; canary rollouts; drift detection
- An **identity fabric** — SPIFFE/SPIRE workload certificates for every agent; mTLS on every agent-to-agent and agent-to-tool call; revocable per-agent, per-fleet
- An **audit ledger** — append-only, hash-chained, OCSF-compliant; every invocation, every skill call, every policy decision, every escalation, indexed by workflow instance
- A **human escalation queue** — structured decision workflows, not alerting; every escalation carries the full cross-agent context of the workflow that produced it
- A **federated policy system** — hierarchical governance from platform operator down to department, where policy can be tightened but never loosened at lower levels

Strata runs on any infrastructure. It is indifferent to what models run the agents, what cloud hosts the compute, and what vendors supply the skills. It is the constant — the governance, identity, and operational layer that makes a heterogeneous agent estate legible, trustworthy, and manageable at scale.

---

## 3. Guiding principles

| Principle | What it means |
|---|---|
| **Infra-agnostic** | Run on AWS, GCP, Azure, bare metal, K8s, or a Raspberry Pi at a clinic. Same binary, same API, same governance model everywhere. |
| **Model-agnostic** | Govern agents built on any foundation model — Claude, GPT-4o, Gemini, Llama, local fine-tunes. Strata does not care what is inside the agent. It cares about what the agent does. |
| **OSS-first** | Core runtime, protocol specs, and audit schema contributed to CNCF and the Linux Agent Foundation. No proprietary lock-in at the protocol or data layer. |
| **Standards over proprietary** | Every protocol, event format, and identity mechanism is an open standard with an independent standards body. We do not invent when a standard exists. |
| **Ambient context, not central control** | Agents receive workflow context and make better local decisions. The platform does not route every agent call through a central orchestrator. |
| **Governance as infrastructure** | Policy enforcement, audit, and identity are not bolt-on features. They are load-bearing parts of the platform, as fundamental as the registry and the runtime. |
| **Edge-first resilience** | An edge node with no connectivity must still enforce policy, still run agents, and still accumulate audit events. The control plane is not in the hot path. |

---

## 4. Architecture: the agentic middle layer

Strata assumes infrastructure exists. It does not provision compute, storage, or networking. It runs on top of whatever infrastructure the customer already operates.

```
┌───────────────────────────────────────────────────────────────┐
│  Developer / operator plane                                   │
│  CLI · GitOps API · Web console · Provider SDK               │
├───────────────────────────────────────────────────────────────┤
│  Fleet control plane  (SaaS or self-hosted)                  │
│  Agent registry · Skill registry · Policy store              │
│  Audit ledger · Escalation queue · Fleet dashboard           │
├───────────────────────────────────────────────────────────────┤
│  Node runtime  (cloud / on-prem / edge — runs anywhere)      │
│  Agent executor · Skill loader (WASM sandbox)                │
│  Intent descriptor engine · Guardrail engine (OPA in-proc)  │
│  SPIFFE identity sidecar · OTel collector · Policy cache     │
├───────────────────────────────────────────────────────────────┤
│  Heterogeneous infrastructure                                 │
│  AWS · Azure · GCP · bare metal · K8s · edge device          │
└───────────────────────────────────────────────────────────────┘
```

### Core components

| Component | Role |
|---|---|
| **Agent registry** | Versioned, signed agent definitions packaged as OCI artifacts. Agents are always deployed from registry — never ad-hoc. Supports canary rollout and automatic rollback. |
| **Skill/tool registry** | MCP-compatible capability catalog. Skills packaged as WASM modules — run identically on cloud and air-gapped edge nodes. Namespace-scoped: tenants cannot enumerate each other's skills. |
| **Intent descriptor engine** | Instantiates, propagates, and indexes intent descriptors across multi-agent workflows. Injects workflow context into every A2A call and MCP tool invocation transparently. Covered in depth in §5. |
| **Orchestration layer** | Multi-agent workflow coordination using A2A protocol. Durable task graph — survives node restarts and connectivity loss. Stateless agents coordinate through the graph, not through shared memory. |
| **Guardrail engine** | Five-layer OPA/Rego enforcement evaluated in-process. Policy rules version-controlled, signed, and pushed from the fleet control plane. Covered in depth in §6. |
| **Identity fabric** | SPIFFE/SPIRE — every agent receives a workload certificate at startup. mTLS on all agent-to-agent and agent-to-tool calls. Per-fleet and per-agent revocation. |
| **Audit ledger** | Append-only, hash-chained, OCSF-compliant. Every invocation, policy decision, and escalation recorded. Indexed by `workflow_instance_id` — full cross-agent workflow trace in one query. |
| **Human escalation queue** | Structured decision workflows with SLA enforcement. Every escalation carries the intent descriptor of the workflow that produced it. Covered in §7. |
| **Fleet control plane** | GitOps-style declarative fleet state. Push an agent version update and the control plane orchestrates canary rollout across cloud and edge. Drift detection alerts when a node diverges. |
| **Edge node runtime** | Lightweight binary with local policy cache and registry mirror. Functions fully offline. Syncs audit events and telemetry when connectivity returns. |

---

## 5. Intent descriptors — ambient workflow intelligence

When multiple agents collaborate on a business workflow — order fulfillment, patient triage, loan underwriting, incident response — each agent today knows only its local task. It does not know the broader goal, the constraints that govern this specific instance, which other agents are participating, or what they have already done. Agents make locally reasonable decisions that are globally suboptimal. Audit trails require manual correlation across multiple systems.

Intent descriptors solve this. An intent descriptor is a structured, versioned object that is instantiated at workflow start and propagated automatically to every agent that participates. It is not a task assignment. It is **ambient context**: "you are part of this workflow, here is what matters about it at the level above your task."

> **Intent descriptors give agents the business context they need to make better local decisions — without requiring a central orchestrator to mediate every step.**

### Anatomy of an intent descriptor

```json
{
  "schema_version": "strata.workflow/v1",
  "workflow_id": "order-fulfillment-v2",
  "instance_id": "wf-8f3a9c",
  "intent": "Fulfill same-day customer order",
  "goal": "Order #ORD-291847 delivered before 17:00 UTC",
  "priority": "high",
  "initiated_by": {
    "agent": "sales-agent-v3.1",
    "spiffe_id": "spiffe://strata.acme.com/ns/sales/agent/sales-agent"
  },
  "context": {
    "customer_tier": "enterprise",
    "sla_deadline": "2026-08-08T17:00:00Z",
    "regulatory_flags": [],
    "domain": "order-management"
  },
  "participants": [
    {
      "role": "inventory-checker",
      "agent": "inventory-agent-v2.4",
      "status": "complete",
      "completed_at": "2026-08-08T09:14:22Z"
    },
    {
      "role": "payment-validator",
      "agent": "payment-agent-v1.9",
      "status": "active"
    },
    {
      "role": "fulfillment-scheduler",
      "agent": "fulfillment-agent-v3.0",
      "status": "pending"
    }
  ],
  "constraints": {
    "spend_limit_usd": 500,
    "allowed_suppliers": ["supplier-a", "supplier-b"],
    "escalate_on_sla_risk": true,
    "sla_risk_threshold_minutes": 60
  },
  "trace_context": {
    "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"
  }
}
```

### What each agent can do with it

The descriptor is available to every agent in the workflow at invocation time. Agents use it to make better local decisions without needing to be explicitly programmed for every scenario:

- The **fulfillment agent** sees this is a high-priority enterprise order expiring at 17:00 and selects expedited fulfillment without being told explicitly.
- The **payment agent** sees the spend limit and applies it directly — no round-trip to ask.
- Any agent can see that inventory clearance is already complete — no redundant check.
- Any agent can see that `escalate_on_sla_risk` is true and knows to surface an escalation if it detects risk to the deadline, referencing the workflow instance rather than just its local task.
- A guardrail rule can read `workflow.priority == "high"` and `remaining_sla < 60min` and unlock expedited spend approval — contextual policy that would otherwise require a separate orchestration branch.

### How it propagates

Propagation follows **W3C Trace Context + OpenTelemetry Baggage** — the same mechanism that carries trace IDs across service boundaries, extended with business-semantic fields. The intent descriptor is attached to every A2A call and every MCP tool invocation as a signed context header. Agents do not forward it manually — the Strata node runtime injects and extracts it transparently at the transport layer.

The `traceparent` field ties the business workflow to the technical trace: OpenTelemetry spans generated during the workflow carry the same trace ID, so infrastructure latency and business-level workflow events appear in the same observability tool.

### What the audit ledger gains

Every audit event is indexed by `workflow_instance_id`. The full cross-agent trace for "Order #ORD-291847" — every agent invocation, every skill call, every policy decision, every escalation — is reconstructable in a single query. A regulator or incident reviewer does not assemble it manually from five separate logs.

```
GET /audit/workflow/wf-8f3a9c

→ 14 events across 3 agents · 4m 37s elapsed · 0 policy violations
  09:14:08  inventory-agent-v2.4   INVOKED   role=inventory-checker
  09:14:09  inventory-agent-v2.4   SKILL     check-inventory(SKU-449) → in_stock=true
  09:14:10  inventory-agent-v2.4   COMPLETE  result=inventory_cleared
  09:14:12  payment-agent-v1.9     INVOKED   role=payment-validator
  09:14:13  payment-agent-v1.9     GUARD     L2 scope check spend_limit=500 ✓
  09:14:14  payment-agent-v1.9     SKILL     validate-payment(amount=312.40) → approved
  ...
```

### Intent descriptors as an open standard

The intent descriptor schema is a candidate for standardization through the Linux Agent Foundation — a "Workflow Context Protocol" alongside MCP and A2A. Any agent framework that implements the propagation contract gains interoperability: a LangGraph agent, an AutoGen agent, and a Strata-native agent can all participate in the same workflow and share the same descriptor. The standard we define is the network effect.

---

## 6. Guardrail engine — layered enforcement

Guardrails are not prompts. They are enforced in the Strata node runtime, in-process, before and after every model call and skill invocation. The model cannot override them. A guardrail violation does not produce an agent output — it produces a structured exception written to the audit ledger and, where configured, routed to the human escalation queue.

Guardrail rules are Rego policies — version-controlled, signed, tested locally before deployment, pushed from the fleet control plane. They can read the intent descriptor: a rule may behave differently for a high-priority workflow nearing its SLA deadline than for a low-priority background task.

| Layer | Name | What it checks |
|---|---|---|
| L1 | Input classification | Data sensitivity and domain context. Gates model selection, skill eligibility, and required isolation tier. |
| L2 | Scope enforcement | Agent's declared skill set and tool bindings vs. the request. Structural check — independent of model behavior. |
| L3 | Domain boundary rules | Hard domain-specific constraints: no autonomous diagnosis, no unapproved spend beyond limit, no data crossing a defined boundary. Stored as signed policy objects. Can reference `intent.constraints`. |
| L4 | Output validation | Fast local classifier (sub-100ms): hallucinated entities, contradictions with structured context, confidence below threshold. The classifier is itself a versioned, approved skill registry artifact. |
| L5 | Escalation & circuit breaker | Routes borderline cases to the human escalation queue. Hard-stops policy violations. Auto-suspends agents exceeding violation rate threshold. |

### Workflow-aware guardrail example

```rego
# Allow expedited supplier spend when SLA is critically close
allow_expedited_spend {
  input.intent.priority == "high"
  remaining_sla_minutes < 30
  input.request.spend_usd <= input.intent.constraints.spend_limit_usd * 1.5
  input.supplier == input.intent.constraints.allowed_suppliers[_]
}

remaining_sla_minutes := m {
  deadline := time.parse_rfc3339_ns(input.intent.context.sla_deadline)
  m := (deadline - time.now_ns()) / 60000000000
}
```

---

## 7. Human escalation queue

When a guardrail fires, the agent output is stopped — but the business situation that triggered the workflow has not stopped. The escalation queue routes the event to the right human, with complete context including the full intent descriptor and workflow audit trail, within a time-bound SLA.

It is a **structured decision workflow**, not an alerting system. Every escalation event is an atomic unit of governance with an owner, a deadline, permissible response options, and an immutable resolution record.

### Reviewer response options

| Response | Output state | Audit record |
|---|---|---|
| **Approve & release** | Delivered unchanged | Reviewer identity, timestamp, notes, output hash |
| **Modify & release** | Reviewer edits; modified version delivered | Original hash + modified hash + full diff — permanently archived |
| **Reject** | Not delivered; caller receives structured error | Rejection reason, reviewer identity, timestamp |
| **Flag for policy review** | Rejected for now | Policy amendment workflow opened, linked to this event |

### SLA tiers

| Severity | Primary SLA | On SLA miss |
|---|---|---|
| Critical — safety or compliance | 5 minutes | Secondary escalation + compliance incident |
| High — domain boundary | 15 minutes | On-call escalation |
| Medium — low confidence | 2 hours | Team digest |
| Low — policy advisory | 24 hours | Weekly governance report |

The context provided to every reviewer is assembled automatically by the platform at the moment of guardrail trigger: agent identity and version, requesting user, the workflow intent descriptor, guardrail layers triggered with rule names, the blocked output, and SLA countdown with secondary escalation path on miss. The reviewer never queries another system.

---

## 8. Federated harness policy

In a large organization — a health system with dozens of hospitals, a bank with multiple lines of business, a retailer with regional and brand entities — policy governance is a hierarchy, not a single source of truth. Platform leadership sets a mandatory floor. Entities below that tier can restrict or specialize, but never relax.

> **Policy inherits downward and can only be tightened, never loosened. This invariant is enforced at runtime — it is not convention.**

### Three-tier hierarchy

| Tier | Authority | Scope |
|---|---|---|
| Platform operator | Mandatory floor — cannot be overridden | System-wide guardrail rules, required isolation tiers, approved skill baseline, audit retention |
| Organization | Can restrict, not relax | Org-specific guardrail additions, skill scope, escalation SLA, namespace isolation minimums |
| Department / team | Can restrict within org scope | Specialty rules, agent config, local escalation routing |

### Propagation phases

| Phase | Behavior | Promotion trigger |
|---|---|---|
| **Observe** | Violations logged but not blocked — teams see impact before enforcement | Governance review + sign-off |
| **Warn** | Violations allowed through with warning — teams can adapt | Explicit promotion — not automatic |
| **Enforce** | Full enforcement — violations block output and trigger escalation | Rollback takes effect within one Raft heartbeat (~2s) |

### Peer federation

For organizations that need to share policy and audit across institutional boundaries — research consortia, multi-enterprise supply chains — Strata supports a peer federation model governed by unanimous consent. No member can push a rule into another's policy namespace unilaterally. Shared audit views show aggregate compliance metrics; individual events never cross the institutional boundary without explicit consent.

---

## 9. Multi-tenancy — isolation at every layer

A single Strata deployment serves multiple organizations, business units, or regulated workloads simultaneously. The **Namespace** is the first-class tenancy boundary — every agent, skill, policy, and audit record belongs to a namespace. Isolation is enforced at the platform layer across every dimension.

| Layer | Mechanism | What it prevents |
|---|---|---|
| **Identity** | SPIFFE trust domain per namespace. Workload certificates cannot authenticate across namespaces without explicit federation policy. | A compromised agent in Tenant A authenticating to Tenant B's skills. |
| **Policy** | Namespace-scoped policy inheritance. A policy pushed to Namespace A has no effect on Namespace B. | Policy misconfiguration in one tenant affecting another. |
| **Skill registry** | Skills are namespace-scoped. Tenants cannot enumerate or invoke each other's registered skills. | Skill enumeration or invocation across tenant boundaries. |
| **Audit ledger** | Namespace-partitioned. Audit queries are scoped to the requesting namespace. | Audit data leaking between tenants. |
| **Escalation queue** | Escalation events are namespace-scoped. Reviewers only see events from namespaces they are authorized for. | Clinical escalation events visible to unrelated teams. |
| **Workflow context** | Intent descriptors are namespace-scoped. A workflow in Namespace A cannot reference participants or constraints from Namespace B. | Cross-tenant workflow data leakage. |
| **Network (infrastructure layer)** | When deployed on Strata compute fabric: per-tenant eBPF network policy, VXLAN VNIs, deny-by-default between namespaces. | Traffic interception or injection across tenant network segments. |

For deployments requiring infrastructure-level isolation — regulated workloads, third-party vendor code, or environments where platform-layer isolation is insufficient — Strata integrates with the underlying infrastructure to enforce VM-per-tenant or SR-IOV VF assignment at the compute layer.

---

## 10. Confidential compute and trust

Tenant namespace isolation prevents agents from seeing each other. Confidential compute prevents the infrastructure operator from seeing the agents. These are different threat models, and both matter in regulated industries.

A hospital running a third-party diagnostic AI does not want the cloud provider, the managed service operator, or their own infrastructure team to read the model weights or patient data during inference. Hardware trusted execution environments (TEEs) make this possible.

### Trust ladder

| Tier | Isolation boundary | Suitable for |
|---|---|---|
| Standard container / process | cgroups + namespaces | General workloads, non-sensitive data |
| VM-per-tenant | Hypervisor | Regulated workloads, stronger tenant isolation |
| Confidential VM (AMD SEV-SNP / Intel TDX) | CPU memory encryption | PHI processing, operator-untrusted environments |
| Confidential container (kata-cc + SEV-SNP / TDX) | CPU memory encryption + measured container image | Full operator-blind PHI inference, third-party model IP protection |

Strata is TEE-aware. When an agent is deployed into a confidential container, the agent registry records the expected TEE measurement alongside the agent version and skill set. The audit ledger entry for every invocation includes the attestation status of the executing container — attestation is part of the compliance record, not an out-of-band check.

Remote attestation follows the standard flow: hardware TEE measurement → attestation verifier (AMD KDS / Intel PCS, or customer-operated) → sealed key release (HashiCorp Vault + TEE plugin). Only after a verified measurement does the key management system release the decryption key for model weights or PHI. In the highest-trust deployments, the attestation verifier is customer-controlled — Strata provides the software, the customer operates it.

---

## 11. Open standards stack

Every protocol, event format, and identity mechanism in Strata is an open standard with an independent standards body or CNCF project backing it. Vendor-neutral at the protocol layer is not a marketing claim — it is enforced by the technology choices.

| Problem | Standard | Why |
|---|---|---|
| Tool & context access | **Model Context Protocol (MCP)** | Anthropic open spec; adopted by major model providers. De facto tool access standard. |
| Agent-to-agent calls | **A2A protocol** | Google open spec; vendor-neutral agent interoperability. MCP governs tool access, A2A governs agent coordination. |
| Workflow context propagation | **W3C Trace Context + OTel Baggage + Strata WCP** | Intent descriptors ride OTel Baggage headers; trace IDs tie business events to infrastructure spans. WCP (Workflow Context Protocol) proposed for Linux Agent Foundation standardization. |
| Workload identity | **SPIFFE / SPIRE** | CNCF graduated. Zero-trust identity without a service mesh dependency. Works across K8s, bare metal, edge. |
| Policy enforcement | **Open Policy Agent (OPA) + Rego** | CNCF graduated. Declarative, testable, version-controlled. Evaluates in-process — no network round-trip. |
| Observability | **OpenTelemetry** | CNCF. Traces, metrics, logs — model-agnostic, infra-agnostic. Every agent invocation emits OTel spans. |
| Audit event schema | **OCSF** | Backed by AWS, Splunk, IBM, CrowdStrike. Regulatory-grade audit event format. Queryable by any OCSF-aware SIEM without transformation. |
| Skill portability | **WASM / WASI** | Skills compile once, run anywhere. Sandboxed execution — a skill cannot escape its declared capabilities. Natural fit for edge. |
| Artifact packaging | **OCI artifacts** | Agents and skills packaged as OCI images. Any OCI-compliant registry works — Harbor, ECR, GCR, GHCR. |
| Event routing | **CloudEvents** | CNCF. Fleet events (agent deployed, policy violated, escalation triggered, workflow completed) route to any CloudEvents consumer — Kafka, Pub/Sub, EventBridge — without format translation. |
| Human identity | **OIDC / OAuth 2.0** | Federate with enterprise IdPs — Okta, Azure AD, Ping. Reviewer identity in escalation events is enterprise-verifiable. |
| API | **gRPC / protobuf + OpenAPI** | gRPC for node-to-control-plane communication. OpenAPI for the developer-facing REST API and GitOps integration. |

---

## 12. Why healthcare first

Healthcare is the sharpest proof point for every problem Strata solves. Agents operate across heterogeneous infrastructure. Policy is federated across a hierarchy of institutions. Compliance is non-negotiable. PHI demands operator-blind confidential compute. Human oversight of clinical AI is a regulatory requirement, not a preference. And the cost of getting it wrong is measured in patient outcomes.

| Metric | Figure |
|---|---|
| Global healthcare IT spend | $390B projected by 2028 |
| Top AI adoption barrier | 78% of hospitals cite infrastructure and governance complexity |
| FDA-cleared AI/ML devices | 6,000+ as of 2025, growing 40% year-on-year |

### Primary healthcare use cases

| Use case | How Strata enables it |
|---|---|
| **Multi-agent clinical workflows** | Intent descriptors propagate patient encounter context across triage, radiology, drug interaction, and summarization agents. Full cross-agent audit in one query. |
| **Radiology AI** | Third-party FDA-cleared classifier deployed as a confidential container. Model weights sealed to TEE measurement. Operator-blind inference. Attestation status in every audit entry. |
| **Federated hospital governance** | Health system sets mandatory guardrail floor. Hospitals restrict. Departments narrow further. Policy enforced at runtime — no convention or trust required. |
| **Multi-vendor agent estate** | Five vendors' agents, three infrastructure substrates, one governance layer. Same policy, same audit, same identity across all of them. |
| **Edge clinic deployment** | Single-node edge runtime with local policy cache. Functions offline. PHI never leaves the clinic network. Audit events sync when connectivity returns. |

---

## 13. Deploying agentic systems — the hospital use case

A hospital deploys a diagnostic support system consisting of four agents from two vendors: a triage agent, a radiology classifier, a drug interaction checker, and a clinical note summarizer. The agents run on a mix of on-premise GPU nodes and a hospital-operated edge appliance in the radiology suite. The infrastructure team does not change. The agent vendors operate independently. Strata is the layer that makes it governable.

### What Strata provides end-to-end

**At deployment:** each agent is pulled from the Strata skill registry as a signed OCI artifact, deployed to the appropriate node with a SPIFFE identity certificate, and bound to the approved skill set and guardrail profile defined in the agent registry. No ad-hoc installs. No manual credential distribution.

**At runtime:** when a clinician initiates a patient encounter, the triage agent creates a workflow instance and instantiates an intent descriptor containing the encounter context, the patient data classification (PHI), the applicable regulatory flags, and the SLA. Every subsequent agent invocation in the encounter — radiology classifier, drug interaction check, note summarizer — receives that descriptor automatically. Each agent knows it is part of this encounter, what the constraints are, and what upstream agents have already determined.

**At guardrail trigger:** if the clinical note summarizer produces a candidate output that contains a drug dosage recommendation (a clinical boundary violation), the L3 guardrail blocks the output, writes a structured event to the audit ledger, and opens an escalation event in the clinical informatics queue. The reviewer receives the full encounter context — the intent descriptor, the upstream agent outputs, the blocked candidate, and an SLA countdown — without querying any other system.

**At audit:** the encounter audit log is a single coherent record indexed by workflow instance ID — every agent invocation, every skill call, every guardrail decision, every escalation and its resolution. A regulator requesting evidence of AI governance for this encounter receives a complete, tamper-evident record assembled automatically.

### Active skill registry — example

| Skill | Provider | Approval | Data scope | Isolation required |
|---|---|---|---|---|
| Drug interaction check v2.1.0 | Strata | Hospital approved | Medication list | Standard |
| Chest X-ray classifier v1.4.2 | Rad-AI Inc. | FDA 510(k) | DICOM images | Confidential container (kata-cc) |
| Clinical note summarizer v3.0.1 | MedScribe Co. | Hospital approved | PHI | Confidential container + TEE attestation |
| FHIR patient query v1.0.0 | Strata | Hospital approved | PHI | Standard + audit required |

---

## 14. Provider ecosystem and marketplace

Strata defines an open **Provider SDK** — a specification for packaging an agent or skill as a signed, self-attesting artifact that carries its own safety documentation, validation results, regulatory status, and expected TEE measurement for confidential deployments.

When a provider submits a skill artifact, the attestation bundle travels with it. A hospital's compliance team reviews scope and integration — not model safety, which the attestation covers. This compresses the time from regulatory clearance to deployed skill from months to days, and gives the hospital a stronger machine-verifiable guarantee than a vendor attestation letter.

### Marketplace flywheel

A provider with an FDA-cleared clinical classifier wants to reach every hospital running agents. They publish once to the Strata marketplace with a machine-readable attestation bundle. Every Strata fleet can pull it with a one-click deploy, with signature verification and attestation chain checked automatically at install. Strata takes a revenue share and provides the trust infrastructure. More providers attract more customers; more customers attract more providers.

### Canary rollout

New skill versions roll to 5% of traffic automatically. Automated rollback fires if the guardrail violation rate increases beyond threshold. Hospital compliance approval is required to promote to 100%. The full rollout history is in the audit ledger.

---

## 15. Monetization

The open-core model maximizes community adoption and creates clear upgrade pressure at enterprise scale. The free tier is genuinely useful — that is what drives adoption.

### Tiers

| Tier | Target | Delivery | What they get |
|---|---|---|---|
| **Community** | Individual developers, researchers, small teams | OSS, self-hosted | Core runtime, local registry, single-node fleet, basic OPA guardrails, audit ledger |
| **Cloud** | Teams and startups | Hosted control plane | Fleet management, hosted registry, hosted audit ledger, intent descriptor system, canary rollouts, OCSF export |
| **Enterprise** | Mid-to-large organizations | Self-hosted or hosted | Multi-tenant namespaces, advanced policy management, fleet drift detection, enterprise IdP integration, SLA |
| **Regulated** | Healthcare, finance, defense | Enterprise + compliance modules | HIPAA, SOC 2, FDA 21 CFR Part 11, EU AI Act policy packs; confidential compute integration; human-in-the-loop workflows; attestation; compliance report generation |
| **Marketplace** | Skill and agent providers | Revenue share | 20–30% platform take on skill and agent sales; trust infrastructure (attestation verification, signature checking) included |

### Revenue streams

| Stream | Mechanism | Who pays |
|---|---|---|
| **Hosted fleet control plane** | Per active agent / month | Platform teams who want fleet ops without operating a control plane |
| **Compliance packs** | Annual license per fleet | Regulated industries — updated as standards evolve, audit report generation included |
| **Strata Ops** | Per fleet node / month | Enterprises running large agent fleets who want autonomous fleet operations (anomaly detection, upgrade orchestration, capacity planning) |
| **Marketplace revenue share** | 20–30% of skill sales | Providers monetize through the marketplace; platform takes the trust-infrastructure cut |
| **Enterprise support + SLA** | Annual contract | Enterprises with existing agent deployments to migrate and operate at scale |

### What is open source

- Node runtime and agent executor
- Skill loader and WASM sandbox
- OPA guardrail engine integration
- SPIFFE identity sidecar
- OTel collector pipeline
- Local audit ledger
- A2A and MCP client libraries
- Intent descriptor schema and propagation library
- CLI and GitOps tooling

**OSS home:** core runtime and intent descriptor propagation library → CNCF Sandbox. Workflow Context Protocol spec and OCSF audit schema → Linux Agent Foundation.

---

## 16. Community and governance

Strata's community strategy is modeled on Kubernetes: seed with a working, opinionated v0, release early, and let practitioners shape v1. We open-source the core before Series A, partner with three to five design partners in healthcare and enterprise for v0, and present at KubeCon and at HIMSS and AMIA in year one.

The Workflow Context Protocol — the open standard for intent descriptor propagation — is the strategic contribution to the Linux Agent Foundation. If Strata defines the open standard for how agents share business workflow context, every framework that adopts the standard interoperates with Strata's governance layer. That is the network effect at the protocol layer, not the product layer.

> **We are not building a walled garden. We are building the commons — and building a business on top of the commons by being the best operators of it.**

---

## 17. Roadmap

| When | Milestone | Scope |
|---|---|---|
| **Now — v0** | Core platform | Node runtime. Agent and skill registry (OCI + MCP). Intent descriptor engine v1. Guardrail engine L1–L4 (OPA/Rego in-process). SPIFFE identity fabric. Append-only audit ledger (OCSF). Fleet control plane — single region. A2A orchestration. CLI and gRPC API. CloudEvents emission. Edge node with offline mode. |
| **Q4 2026 — v1** | Community release + healthcare GA | Human escalation queue. Federated policy (3-tier hierarchy). Workflow Context Protocol spec submitted to Linux Agent Foundation. Confidential container support (kata-cc + AMD SEV-SNP). Remote attestation service. CNCF Sandbox submission. Three medical design partners in production. Provider SDK v1 and marketplace alpha. HIPAA compliance pack. |
| **H1 2027 — v1.5** | Enterprise and marketplace GA | Multi-tenant namespaces. Fleet drift detection and automated remediation. Enterprise IdP integration. Peer-federated policy (research consortia). Compliance report generation (HIPAA, SOC 2, EU AI Act). Marketplace GA with revenue share. Strata Ops managed service (capacity planning, anomaly detection, upgrade orchestration, incident response). Intel TDX support. First paid customers. |
| **H2 2027 — v2** | Intelligence fabric | Workflow analytics — aggregate intent descriptor data surfaces patterns across workflow instances for operational and business intelligence. Federated learning integration. Multi-enterprise cross-org workflow federation. Edge form factors — NVIDIA Jetson, Apple Silicon, Qualcomm. ARM and Intel AMX skill optimization. WCP adopted as LAF standard. Marketplace reaches 50+ attested providers. |

---

## 18. Why now

Three forces are converging to make the agentic middle layer an urgent infrastructure problem:

**1. Agents are proliferating faster than governance.** Every major enterprise is deploying multiple agent systems — from different vendors, on different infrastructure, built on different models. The heterogeneity is not going away. What is missing is not more agents; it is the layer that makes the estate legible, trustworthy, and operable as a coherent whole.

**2. Regulatory pressure is forcing the governance question.** The EU AI Act, FDA AI/ML guidance, HIPAA AI enforcement, and emerging state-level regulation are creating mandatory requirements for audit trails, human oversight, and explainable agent decisions. Organizations that built without governance infrastructure are now rebuilding it under deadline. Organizations starting now need to build it in from the first day.

**3. The open standards moment is now.** MCP, A2A, SPIFFE, OPA, OCSF, and OpenTelemetry have matured to the point where a vendor-neutral governance layer can be built entirely on open foundations — no proprietary wire protocol, no locked data format. The Workflow Context Protocol we are proposing is the missing piece: the open standard that ties business intent to the technical event stream. The organization that defines and donates that standard shapes the ecosystem the way the OCI image spec shaped the container ecosystem.

The window to define the open governance standard for the agent era is open. We intend to own that standard — not as a proprietary product, but as the commons that every agent framework builds on, with a commercial business built on being its best operator.

---

*Strata · Vision Paper v0.4 · August 2026*
*OSS core → CNCF · Workflow Context Protocol → Linux Agent Foundation · Fleet control plane + compliance packs + marketplace → commercial products*
*Pre-publication draft — confidential*
