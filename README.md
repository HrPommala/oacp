# OACP — Open Agent Communication Protocol (working draft)

**Version:** 0.1 (working draft)  
**Status:** Draft for review  
**License:** Apache 2.0  
**Reference implementation:** Ojas (pending)  

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)

> *"OACP is a transport-neutral, contract-first protocol for governed agent‑runtime communication."*

OACP defines a typed envelope, deterministic processing gates, and a two‑axis profile system that makes agent communication **auditable, replay‑safe, and policy‑enforceable**. It is the missing governance and audit plane for autonomous AI agents.

## Table of Contents

- [Why OACP exists](#why-oacp-exists)
- [What OACP is (and is not)](#what-oacp-is-and-is-not)
- [Frozen architectural invariants](#frozen-architectural-invariants)
- [Key features](#key-features)
- [How OACP compares to other frameworks](#how-oacp-compares-to-other-frameworks)
- [The OACP envelope and processing gates](#the-oacp-envelope-and-processing-gates)
- [Conformance](#conformance)
- [Safety boundary](#safety-boundary)
- [Repository structure](#repository-structure)
- [Getting started](#getting-started)
- [Contributing](#contributing)
- [Versioning and roadmap](#versioning-and-roadmap)
- [License and IPR](#license-and-ipr)
- [Contact](#contact)

---

## Why OACP exists

Existing agent communication protocols solve important but narrow problems:

- **A2A** (Agent‑to‑Agent) – discovery and task delegation.  
- **MCP** (Model Context Protocol) – tool integration.  
- **ACNBP** – capability negotiation.  

None of them prescribe what *governed* agent communication looks like:

- **What makes a message replay‑safe?**  
- **What constitutes an approval vs. a proposal?**  
- **How do you enforce tenant isolation across references?**  
- **What audit chain must a runtime produce?**  

OACP fills that gap. It layers governance semantics on top of existing protocols without replacing them. It is the **governance plane** that enterprises and regulated industries need to run AI agents safely.

---

## What OACP is (and is not)

| ✅ OACP is … | ❌ OACP is not … |
|--------------|------------------|
| A transport‑neutral protocol for governed agent communication | A replacement for A2A, MCP, or ACNBP (it layers on top) |
| A deterministic processing model with 15 validation gates | A specification of agent reasoning or model behavior |
| A proposal/decision boundary that prevents self‑approval | A transport‑layer security mechanism (TLS is assumed) |
| A typed reference model with tenant‑scoped resolution | An identity provider or credential issuer |
| A suite of conformance tests and bindings | A generic LLM wrapper or prompt registry |

---

## Frozen architectural invariants

These ten invariants are **frozen for v0.1** and will not be relaxed in any future minor version. They are the strongest commitments OACP makes.

1. **Identity is sender‑authored and immutable; verification is receiver‑authored and appended.** The identity block is signed by the sender; receivers append `verificationTrail` entries.  
2. **Tenant authorization lives in `verification.tenantScope`, not in `identity`.** Senders cannot self‑assert tenant authority.  
3. **Proposals are recommendations; approvals require separate decision artifacts.** A proposal alone never authorizes an effect.  
4. **Review feedback never finalizes a decision.** Reviewers comment; deciders decide.  
5. **Every proposal closure produces a decision record** — including expiry and supersession, not only explicit approval/rejection.  
6. **Hash chains are run‑scoped.** They never cross `runId` boundaries; cross‑run continuity uses `causation` and `proposalRefs[]`, not hashes.  
7. **Tenant isolation is a protocol‑level invariant.** Every reference is tenant‑checked at four enforcement points.  
8. **Extensions cannot weaken core.** Extensions can only add stricter validation; any extension that disables a core gate is rejected with `EXTENSION_WEAKENS_CORE`.  
9. **Self‑decision is forbidden.** A decider MUST NOT equal the proposer (the only exception is runtime auto‑approval, where the runtime is a distinct participant).  
10. **Manifests are immutable; lifecycles are unified** across all registry artifacts (manifests, bindings, policies, decisions).

---

## Key features

**At a glance:**

- [Deterministic processing gates](#1-deterministic-processing-gates) (15 steps, fixed order)  
- [Two‑axis profile system](#2-two‑axis-profile-system) (base hierarchy + additive capabilities)  
- [Typed references with tenant isolation](#3-typed-references-with-tenant-isolation) (5 subtypes, 4 enforcement points)  
- [Proposal/decision separation](#4-proposaldecision-separation) (10 auto‑approval preconditions, lifecycle)  
- [Run‑scoped hash chains & integrity](#5-run‑scoped-hash-chains--integrity) (signatures, age limits)  
- [Extensions that cannot weaken core](#6-extensions-that-cannot-weaken-core) (4‑tier precedence)  
- [v0.2 candidates](#7-v02-candidates-coming-soon)

### 1. Deterministic processing gates

Every OACP message passes through 15 governance steps **before** any business effect occurs. Failure at any gate produces a structured error and halts processing.

1. Verify identity  
2. Verify tenant/workspace/system scope *(see note below)*  
3. Verify participant authority  
4. Verify capability binding  
5. Verify authorization decision  
6. Idempotency / retry / replay state  
7. Broker state  
8. Response correlation  
9. Causation DAG  
10. Conversation scope  
11. Reference scope and integrity  
12. Profile / extension precedence  
13. Proposal / review / decision state  
14. Message age / hash chain / signature  
15. Payload validation  

> **Note on gate 2 (scope).** Three scope kinds are recognized: `TENANT` (default), `WORKSPACE` (sub‑tenant grouping), and `SYSTEM`. `SYSTEM`‑scoped messages MUST carry `systemScopeAuthorized: true` and MUST NOT reference tenant artifacts without an explicit cross‑scope policy. Tenant‑scope verification is **never skippable** when tenant scoping is configured.
>
> **Note on numbering.** The spec lists this as gate 1.5; this README renumbers it as gate 2 for readability. The gate order and semantics are identical to the spec.

OACP defines approximately **150 reserved error codes** mapped to specific gate failures (see spec §2a–§27).

### 2. Two‑axis profile system

- **Base profiles** (strict hierarchy, exactly one per message): `OACP-Core` → `OACP-Operational` → `OACP-Governed` → `OACP-Regulated`  
- **Capability profiles** (additive, zero or more): `OACP-Capability`, `OACP-Binding`, `OACP-Reliable`, `OACP-Tool`, `OACP-Review`, `OACP-Proposal`, `OACP-Interop`

The base profile registry is **major‑version‑closed**; the capability registry is **controlled‑additive**.

**Recommended message‑age limits per profile:**

| Profile | Default `maxMessageAgeSeconds` |
|---------|-------------------------------|
| `OACP-Core` | unlimited (trusted in‑process flows) |
| `OACP-Operational` | 86 400 (24 hours) |
| `OACP-Governed` | 3 600 (1 hour) |
| `OACP-Regulated` | 300 (5 minutes) |

### 3. Typed references with tenant isolation

| Subtype | Points to | Tenant scope |
|---------|-----------|--------------|
| `ArtifactRef` | External content (documents, tool inputs, evidence) | Required |
| `MessageRef` | Another OACP message | Implicit or explicit |
| `RecordRef` | Registry records (manifests, bindings, policies) | Tenant or `GLOBAL` |
| `DecisionRef` | Decision artifacts (authorization, approvals, expirations) | Tenant |
| `AuditRef` | Audit chain entries | Tenant |

**Four tenant‑isolation enforcement points:**

1. At message receipt — `scope.tenantId` must lie within the receiver's authorized scope.  
2. At identity verification — the identity's `tenantScope` must include `tenantId`.  
3. At reference resolution — every reference's tenant must match the message's tenant.  
4. At binding scope check — the binding's tenant must match the message's tenant.

**Cross‑tenant references require a pre‑registered `crossTenantPolicyRef`.** Embedding policies inline is forbidden — the policy MUST exist as a registry record before the cross‑tenant reference is allowed.

### 4. Proposal/decision separation

- Agents emit **proposals** (recommendations).  
- Approvals require separate **decision artifacts**: `ExternalDecisionRef` (human/external reviewer) or `RuntimeDecisionRef` (runtime auto‑approval).  
- **Self‑approval is forbidden.** The decider participant MUST NOT equal the proposer.  
- **Every proposal closure produces an immutable decision record** — including expiry, supersession, or invalidation.

**Decision lifecycle:** `ISSUED` → `ACCEPTED` → ( `REVOKED` | `SUPERSEDED` | `INVALIDATED` ).

**Auto‑approval requires all ten preconditions** to be simultaneously satisfied (see spec §13.5):

1. Risk class is `LOW`.  
2. Calibration threshold met for the deciding policy.  
3. All 15 validation gates passed.  
4. Evidence quality meets the policy's required floor.  
5. Capability binding is valid and within scope.  
6. Tenant is enrolled in the auto‑approval policy.  
7. Auto‑approval is explicitly permitted for this action class.  
8. No active review is open against the proposal.  
9. No active escalation flag is set on the run.  
10. An auditable runtime decision record is produced.

### 5. Run‑scoped hash chains & integrity

| Dimension | Field(s) | Rule |
|-----------|----------|------|
| Per‑message integrity | `payloadHash`, `envelopeHash` | Cover canonical‑serialised payload/envelope (excluding `integrity` block). |
| Message age | `createdAt` + `maxMessageAgeSeconds` | Reject if older than the profile limit. |
| Hash chain | `previousMessageHash` | `previousMessageHash[N] = envelopeHash[N-1]`. Run‑scoped only. |
| Signature | `signatureRef` | Covers `envelopeHash` at minimum. |

- **Hash chains are run‑scoped.** They never cross `runId` boundaries. Cross‑run continuity uses `causation` and `proposalRefs[]`, not hashes.  
- **Signatures bind canonicalization and hash algorithm.** A signature MUST cover the `canonicalization` method identifier and `hashAlgorithm` identifier to prevent algorithm‑downgrade attacks.  
- **Replay is governed.** Replay metadata requires explicit authorization with verified authority.

### 6. Extensions that cannot weaken core

- Namespaced extensions: `x-org-*`, `x-vendor-*`, `x-regulated-*`, `x-experimental-*`. Bare `x-*` is permitted only at `OACP-Core`/`OACP-Operational` and is rejected at `OACP-Governed`+.  
- Required extensions can impose **stricter** validation but never disable a core gate.  
- **Extensions MUST be preserved by intermediaries.** Stripping or rewriting extensions is a protocol violation (`EXTENSION_STRIPPED`).  
- `requiredExtensions[]` is the single source of truth for what a receiver must understand.

**Four‑tier precedence hierarchy** (higher tier wins on conflict):

| Tier | Name | Examples |
|------|------|----------|
| **A** | Envelope authority | `oacpVersion`, `messageKind`, `scope.tenantId`, `from.identity` |
| **B** | Governance metadata | `authorizationDecision`, `policySnapshotId` |
| **C** | Payload | All fields under `payload` |
| **D** | Extensions | All fields under `extensions` |

### 7. v0.2 candidates (coming soon)

Building on community feedback, v0.2 will add:

- **Trajectory‑level authorization & session risk memory** (closes the "slow‑burn" attack gap)  
- **Pre‑execution run‑intent gate** (gate runs before any resource consumption)  
- **Reasoning trace evidence** (auditable chain‑of‑thought)  
- **Data handling block** with consent and purpose limitation  
- **Kill‑switch / emergency stop** semantics  
- **Tool side‑effect classes**, **evidence quality scoring**, and **policy simulation**.

See [`V0.2_CANDIDATES.md`](./V0.2_CANDIDATES.md) for the full list.

---

## How OACP compares to other frameworks

| Framework / Protocol | Core Focus | OACP relationship |
|----------------------|------------|-------------------|
| **Microsoft Agent Governance Toolkit** | Runtime policy enforcement, execution sandbox | OACP defines the governance contract that such toolkits can **implement** |
| **AEGIS (pre‑execution firewall)** | Tool‑call validation, approval workflows | OACP provides the **protocol‑level** authorization and audit layer that AEGIS can enforce |
| **MCP / A2A / ACNBP** | Connectivity, tool integration, capability negotiation | OACP layers **governance envelopes** on top; they complement, not compete |
| **Zep / Mem0 / Letta** | Vector memory, session memory | OACP defines **governed memory retrieval** (permission‑first, policy‑aware) |
| **Superagent** | Guardrails, safety policies | OACP's policy memory and authorization decisions provide a protocol‑level home for such guardrails |

**Key differentiator:** OACP is the only **vendor‑neutral, transport‑neutral specification** for agent governance. Others are implementations or tool‑specific libraries.

---

## The OACP envelope and processing gates

A minimal OACP envelope looks like:

```json
{
  "oacpVersion": "0.1",
  "messageId": "msg_001",
  "messageKind": "PROPOSAL",
  "messageType": "PROPOSAL_EMITTED",
  "createdAt": "2026-05-06T10:00:00Z",
  "scope": { "tenantId": "tenant_001" },
  "from":      { "/* identity, role, participantId */": "..." },
  "integrity": { "/* envelopeHash, payloadHash, previousMessageHash */": "..." },
  "payload":   { "/* proposal-specific fields */": "..." }
}
```

For a complete walkthrough, see the [examples](./examples/) and the full specification [`OACP_SPEC.md`](./OACP_SPEC.md).

---

## Conformance

OACP defines five conformance levels (see spec §22.2). An implementation MAY claim multiple levels.

| Conformance level | Minimum profile | Key requirements |
|-------------------|-----------------|------------------|
| **L1** | `OACP-Core` | Envelope structure, payload validation |
| **L2** | `OACP-Operational` | + Identity verification, message age enforcement |
| **L3** | `OACP-Governed` | + Tenant isolation, capability binding, authorization decisions |
| **L4** | `OACP-Regulated` | + Cryptographic signatures, hash chain enforcement, signed decisions, immutable audit |
| **L5** | `OACP-Regulated` + capabilities | + All capability profiles claimed (Tool, Review, Proposal, Interop, Reliable) |

Conformance is established by passing the test vectors in [`conformance/vectors/`](./conformance/vectors). Each vector is a `(input envelope, expected outcome)` pair covering valid envelopes, gate failures, and edge cases.

---

## Safety boundary

OACP is an **operational AI governance protocol**. It helps make agent actions attributable, scoped, policy‑bound, reviewable, and auditable.

It does **not**:

- Guarantee that a model is aligned, truthful, or safe.  
- Replace model‑level safety filters or jailbreak defences.  
- Provide transport‑level confidentiality (TLS is assumed).  
- Act as an identity provider.

> **Model guarantees remain the responsibility of the model provider, training processes, and runtime sandboxing.**

---

## Repository structure

```text
oacp/
├── README.md                  ← this file
├── LICENSE                    ← Apache 2.0
├── OACP_SPEC.md               ← full specification (approx. 180 KB)
├── GOVERNANCE.md              ← protocol governance
├── CONTRIBUTING.md            ← how to contribute
├── SECURITY.md                ← threat model and vulnerability reporting
├── IPR.md                     ← intellectual property posture
├── CHANGES.md                 ← version history
├── V0.2_CANDIDATES.md         ← planned v0.2 features
│
├── schemas/
│   ├── core/                  ← envelope‑block schemas (17 files)
│   └── payloads/              ← payload‑type schemas (10 files)
│
├── bindings/
│   ├── OACP_CLOUDEVENTS_BINDING.md
│   ├── OACP_ASYNCAPI.yaml
│   ├── OACP_OPENAPI.yaml
│   └── OACP_OTEL_MAPPING.md
│
├── conformance/
│   ├── OACP_CONFORMANCE.md
│   ├── runner-spec.md
│   ├── valid-envelope.json
│   └── vectors/               ← test vectors
│
└── examples/
    ├── 01-basic-proposal.md
    ├── handoff-example.json
    ├── tool-request-example.json
    └── ...
```

---

## Getting started

1. **Read the specification** – [`OACP_SPEC.md`](./OACP_SPEC.md) §0 (Foreword) and §1 (Conventions).  
2. **Explore the examples** – [`examples/`](./examples) contains step‑by‑step walks and standalone JSON envelopes.  
3. **Run conformance tests** – Use the vectors in [`conformance/vectors/`](./conformance/vectors) to validate your implementation.  
4. **Check the threat model** – See spec §26 for eight threat surfaces and their mitigations.  

### Quick example – Validating a proposal envelope

Use the JSON Schema validator of your choice (`ajv`, `check-jsonschema`, `jsonschema`, etc.):

```bash
# Example using the python `jsonschema` CLI
jsonschema -i examples/proposal-example.json schemas/oacp-envelope-v1.schema.json
```

For a practical step‑by‑step of an agent run, see [`examples/01-basic-proposal.md`](./examples/01-basic-proposal.md).

---

## Contributing

We welcome contributions that:

- Clarify ambiguities in the specification.  
- Fill gaps in the threat model.  
- Add conformance test vectors.  
- Provide new transport bindings.  
- Report real‑implementation experience.

Please read [`CONTRIBUTING.md`](./CONTRIBUTING.md) for detailed guidelines.  
For security issues, follow the process in [`SECURITY.md`](./SECURITY.md). **Do not open a public issue for security findings.**

---

## Versioning and roadmap

**Pre‑1.0 (v0.x):**  

- Backward compatibility is best‑effort.  
- Breaking changes are permitted and documented in [`CHANGES.md`](./CHANGES.md).  
- **Pin to an exact version** (e.g., `oacpVersion: "0.1"`) in production deployments.  

**Post‑1.0:**  

- Strict backward compatibility within a major version.  
- Minor releases add optional features; patches fix bugs and clarify wording.  

| Tier | Description | Status |
|------|-------------|--------|
| Tier 1 – Protocol Works | Identity, binding, idempotency, broker, conversation | ✅ Frozen |
| Tier 2 – Protocol Is Precise | Profiles, references, tenant isolation, review, proposal/decision | ✅ Frozen |
| Tier 3 – Protocol Is Publishable | Threat model, versioning, schemas, bindings, conformance | ✅ Frozen (v0.1) |
| Tier 4 – Governance‑Rich | Obligations, data handling, side‑effect classes, budgets, kill‑switch, trajectory auth | 🔜 v0.2 |
| Tier 5 – Ecosystem‑Ready | Federation, formal registry, advanced regulated profiles | 🔜 v0.3+ |

See [`CHANGES.md`](./CHANGES.md) for release notes and [`V0.2_CANDIDATES.md`](./V0.2_CANDIDATES.md) for the v0.2 candidate list.

---

## License and IPR

OACP v0.1 is published under the **Apache License 2.0**.  
A formal IPR policy with patent non‑assertion commitments is deferred until v1.0. See [`IPR.md`](./IPR.md) for details.

---

## Contact

For questions about the specification or governance:

- Open a [GitHub Discussion](https://github.com/Pommala-LLC/oacp/discussions) (preferred).  
- Reach out via the maintainer contact in [`GOVERNANCE.md`](./GOVERNANCE.md).  

For security issues, use the process in [`SECURITY.md`](./SECURITY.md).

---

**OACP is a working draft. We welcome your feedback and contributions to make governed agent communication a reality.**
