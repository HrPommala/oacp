# OACP — Open Agent Communication Protocol (working draft)

**Version:** 0.1 (working draft)  
**Status:** Draft for review  
**License:** Apache 2.0  
**Reference implementation:** Ojas (pending)  

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)

> *“OACP is a transport-neutral, contract-first protocol for governed agent‑runtime communication.”*

OACP defines a typed envelope, deterministic processing gates, and a two‑axis profile system that makes agent communication **auditable, replay‑safe, and policy‑enforceable**. It is the missing governance and audit plane for autonomous AI agents.

## Table of Contents

- [Why OACP exists](#why-oacp-exists)
- [What OACP is (and is not)](#what-oacp-is-and-is-not)
- [Key features](#key-features)
- [How OACP compares to other frameworks](#how-oacp-compares-to-other-frameworks)
- [The OACP envelope and processing gates](#the-oacp-envelope-and-processing-gates)
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

## Key features

### 1. Deterministic processing gates
Every OACP message passes through 15 governance steps **before** any business effect occurs:

1. Verify identity  
2. Verify tenant/workspace scope  
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

Each gate is separately auditable and policy‑enforceable.

### 2. Two‑axis profile system
- **Base profiles** (strict hierarchy): `OACP-Core` → `OACP-Operational` → `OACP-Governed` → `OACP-Regulated`  
- **Capability profiles** (additive): `OACP-Capability`, `OACP-Binding`, `OACP-Reliable`, `OACP-Tool`, `OACP-Review`, `OACP-Proposal`, `OACP-Interop`

### 3. Typed references with tenant isolation
- `ArtifactRef` – external content (documents, tool inputs, evidence)  
- `MessageRef` – OACP messages  
- `RecordRef` – registry records  
- `DecisionRef` – decisions (authorization, approvals, expirations)  
- `AuditRef` – audit chain entries  

All references are tenant‑scoped by default. Cross‑tenant flows require explicit registered policies.

### 4. Proposal/decision separation
- Agents emit **proposals** (recommendations).  
- Approvals require separate **decision artifacts** (`ExternalDecisionRef` for humans, `RuntimeDecisionRef` for auto‑approval).  
- Self‑approval is forbidden.  
- Every proposal closure produces an immutable decision record.

### 5. Run‑scoped hash chains & replay protection
- Hash chains are scoped to a single `runId` – each run is an independent audit segment.  
- Message‑age limits (configurable per profile).  
- Replay is governed and requires explicit authorization.

### 6. Extensions that cannot weaken core
- Namespaced extensions (`x-org-*`, `x-vendor-*`, etc.)  
- Required extensions can impose stricter validation but never disable a core gate.

### 7. v0.2 candidates (coming soon)
Building on community feedback, v0.2 will add:
- **Trajectory‑level authorization & session risk memory** (closes the “slow‑burn” attack gap)  
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
| **Superagent** | Guardrails, safety policies | OACP’s policy memory and authorization decisions provide a protocol‑level home for such guardrails |

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
  "from": { ... },
  "integrity": { ... },
  "payload": { ... }
}
