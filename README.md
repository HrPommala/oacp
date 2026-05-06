# OACP — Open Agent Communication Protocol (working draft)

**Version:** 0.1 (working draft)  
**Status:** Draft for review  
**License:** Apache 2.0  
**Reference implementation:** Ojas (pending)

> **OACP is a transport‑neutral, contract‑first protocol for governed agent‑runtime communication.**  
> It makes agent communication **deterministic, auditable, replay‑safe, and policy‑enforceable**.

---

## Table of Contents

- [Frozen architectural invariants](#frozen-architectural-invariants)
- [What OACP is (and is not)](#what-oacp-is-and-is-not)
- [The processing gates (deterministic order)](#the-processing-gates-deterministic-order)
- [Two‑axis profile system](#twoaxis-profile-system)
- [Typed references and tenant isolation](#typed-references-and-tenant-isolation)
- [Proposal/decision separation](#proposaldecision-separation)
- [Integrity: hashes, age, hash chains, signatures](#integrity-hashes-age-hash-chains-signatures)
- [Extensions and precedence](#extensions-and-precedence)
- [Threat model summary](#threat-model-summary)
- [Repository structure](#repository-structure)
- [Getting started](#getting-started)
- [Versioning and roadmap](#versioning-and-roadmap)
- [License and IPR](#license-and-ipr)

---

## Frozen architectural invariants

These ten invariants are **frozen for v0.1** and will not be relaxed in later versions.

1. **Identity is sender‑authored and immutable; verification is receiver‑authored and appended.**  
2. **Tenant authorization lives in `verification.tenantScope`, not in `identity`.**  
3. **Proposals are recommendations; approvals require separate decision artifacts.**  
4. **Review feedback never finalizes a decision.**  
5. **Every proposal closure produces a decision record.**  
6. **Hash chains are run‑scoped** (they do not cross run boundaries).  
7. **Tenant isolation is a protocol‑level invariant** (every reference is tenant‑checked).  
8. **Extensions cannot weaken core** (they can only add stricter validation).  
9. **Self‑decision is forbidden** (no agent can approve its own proposal).  
10. **Manifests are immutable; lifecycles are unified** across all registry artifacts.

---

## What OACP is (and is not)

| ✅ OACP is … | ❌ OACP is not … |
|--------------|------------------|
| A governance envelope for agent‑runtime communication | A replacement for A2A, MCP, or ACNBP (it layers on top) |
| A deterministic processing model with 15 gates | A specification of agent reasoning or model behavior |
| A proposal/decision separation | A transport‑layer security mechanism (TLS assumed) |
| Typed, tenant‑scoped references | An identity provider |
| A suite of conformance tests and bindings | A general LLM wrapper or prompt registry |

> **Safety boundary:** OACP is an **operational AI governance protocol**. It does **not** guarantee model alignment, truthfulness, or safety. Those remain the responsibility of the model provider and deployment environment.

---

## The processing gates (deterministic order)

Every OACP message passes through these gates **in strict order**. Failure at any gate produces a structured error and halts processing.

```text
1.   Verify identity                               (§3a)
1.5. Verify tenant/workspace/system scope          (§2d)
2.   Verify participant authority                  (§3a.8)
3.   Verify capability manifest and binding        (§3b)
4.   Verify authorizationDecision                  (§3a.6)
5.   Verify idempotency / retry / replay state     (§3c)
6.   Verify broker state, if broker‑managed        (§10)
7.   Verify responseTo, if response/result         (§3d.11)
8.   Verify causation DAG                          (§3d.7)
9.   Verify conversation scope                     (§3d.2)
10.  Verify reference scope and integrity          (§2c, §2d)
11.  Verify profile / extension / precedence       (§2a, §2b)
12.  Verify proposal / review / decision state     (§11a, §13)
13.  Verify message age / hash‑chain / signature   (§3e)
14.  Validate payload                              (§1.3)
15.  Dispatch, reject, quarantine, or emit error
```

Tenant scope (gate 1.5) is **never skippable** when tenant scoping is configured.

---

## Two‑axis profile system

### Base profiles (exactly one per message, strict hierarchy)

| Base profile | Implies | Meaning |
|--------------|---------|---------|
| `OACP-Core` | None | Minimum assurance, trusted in‑process flows |
| `OACP-Operational` | `OACP-Core` | Authentication and message‑age enforcement |
| `OACP-Governed` | `OACP-Operational`, `OACP-Capability`, `OACP-Binding` | Full envelope governance, tenant isolation, capability binding |
| `OACP-Regulated` | `OACP-Governed`, `OACP-Reliable` | Cryptographic attestation, hash‑chain enforcement, signed decisions, immutable audit |

### Capability profiles (additive, zero or more)

- `OACP-Capability` – manifest registration  
- `OACP-Binding` – binding lifecycle  
- `OACP-Reliable` – idempotency, retry, replay  
- `OACP-Tool` – tool broker mediation  
- `OACP-Review` – REVIEW protocol  
- `OACP-Proposal` – proposal/decision protocol  
- `OACP-Interop` – cross‑deployment interoperability  

The base profile registry is **major‑version‑closed**; the capability registry is **controlled‑additive**.

---

## Typed references and tenant isolation

OACP defines five reference subtypes, each answering a different question:

| Subtype | Points to | Contains | Tenant scope |
|---------|-----------|----------|--------------|
| `ArtifactRef` | External content (PDF, evidence, tool input) | Yes (resolvable bytes) | Required |
| `MessageRef` | Another OACP message | No | Implicit or explicit |
| `RecordRef` | Registry record (manifest, binding, policy) | No | Tenant or GLOBAL |
| `DecisionRef` | Decision artifact (authorisation, approval) | No | Tenant |
| `AuditRef` | Audit‑chain entry | Sometimes | Tenant |

**Tenant isolation invariant:** every reference resolved by a receiver MUST be within the receiver's authorized tenant scope. Cross‑tenant references require an explicit registered policy (`crossTenantPolicyRef`).

Four enforcement points:

1. At message receipt (`scope.tenantId` in receiver's scope)  
2. At identity verification (identity's `tenantScope` includes `tenantId`)  
3. At reference resolution (reference's tenant matches message's tenant)  
4. At binding scope check (binding's tenant matches message's tenant)

---

## Proposal/decision separation

- **Agents emit proposals** (recommendations).  
- **Proposals are never approvals.**  
- **Approvals require separate decision artifacts**:
  - `ExternalDecisionRef` – human/external reviewer approves/rejects.  
  - `RuntimeDecisionRef` – runtime auto‑approves, rejects, expires, or supersedes.  
- **Self‑decision is forbidden** (a decider cannot equal the proposer). The only exception is runtime auto‑approval (runtime is a different participant).  

**Every proposal closure produces a decision record** – even expiry or supersession.  
**Every decision has a lifecycle:** `ISSUED` → `ACCEPTED` → (`REVOKED`|`SUPERSEDED`|`INVALIDATED`).

**Auto‑approval requires all ten preconditions** (risk class LOW, calibration threshold met, all validation gates passed, evidence quality met, binding valid, tenant in policy, auto‑approval permitted, no active review, no active escalation, auditable record).

---

## Integrity: hashes, age, hash chains, signatures

| Dimension | Field(s) | Rule |
|-----------|----------|------|
| Per‑message integrity | `payloadHash`, `envelopeHash` | Covers canonical‑serialised payload/envelope (excluding `integrity` block) |
| Message age | `createdAt` + `maxMessageAgeSeconds` | Reject if older than profile limit |
| Hash chain | `previousMessageHash` | Run‑scoped; root message has none; every non‑INITIATOR references prior `envelopeHash` |
| Signature | `signatureRef` | Covers `envelopeHash` at minimum; binding of canonicalisation and algorithm |

**Hash‑chain rule:** `previousMessageHash[N] = envelopeHash[N-1]`.  
**Run‑scoped only:** chains never cross `runId` boundaries.

---

## Extensions and precedence

Extensions are namespaced (`x-org-*`, `x-vendor-*`, `x-regulated-*`, `x-experimental-*`).  
Bare `x-*` is permitted only at `OACP-Core`/`OACP-Operational`; rejected at `OACP-Governed`+.

**Four‑tier precedence hierarchy (higher tier wins in conflicts):**

| Tier | Name | Examples |
|------|------|----------|
| **A** | Envelope authority | `oacpVersion`, `messageKind`, `scope.tenantId`, `from.identity` |
| **B** | Governance metadata | `authorizationDecision`, `policySnapshotId` |
| **C** | Payload | All fields under `payload` |
| **D** | Extensions | All fields under `extensions` |

Extensions **cannot weaken core**; a required extension that says "ignore message age" is rejected.  
`requiredExtensions[]` is the single source of truth.

---

## Threat model summary

OACP addresses eight threat surfaces:

| Surface | Primary mitigation |
|---------|---------------------|
| Identity & impersonation | Identity verification gate, per‑hop `verificationTrail`, profile‑driven assurance |
| Capability misuse | Manifest registry immutability, binding evaluation at receive time |
| Authorization bypass | `authorizationDecision` required at Governed+, scope binding, signatures at Regulated |
| Replay & freshness | `maxMessageAgeSeconds`, idempotency keys, replay metadata with authority verification |
| Audit chain integrity | `previousMessageHash` validation, extension preservation rule, off‑runtime audit at Regulated |
| Tenant isolation breach | Four enforcement points, `verification.tenantScope`, cross‑tenant policy required |
| Decision integrity | Self‑decision rule, authority verification, policy snapshot consistency |
| Extension abuse | Tier hierarchy, `EXTENSION_WEAKENS_CORE`, trusted resolver rules |

**Out of scope for v0.1:** compromised runtime, compromised authorisation engine, side‑channel attacks, network‑layer attacks, DoS, model‑level attacks, post‑quantum crypto, insider admin threats.

See [`SECURITY.md`](./SECURITY.md) and §26 of the spec for full details.

---

## Repository structure

```text
oacp/
├── README.md                  ← this file
├── LICENSE                    ← Apache 2.0
├── OACP_SPEC.md               ← full specification
├── GOVERNANCE.md              ← protocol governance
├── CONTRIBUTING.md            ← contribution guide
├── SECURITY.md                ← security posture
├── IPR.md                     ← IPR posture
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
4. **Check the threat model** – See §26 of the spec or [`SECURITY.md`](./SECURITY.md).  

### Quick validation

```bash
# Validate an example envelope against the OACP schema
jsonschema -i examples/proposal-example.json schemas/oacp-envelope-v1.schema.json
```

---

## Versioning and roadmap

| Tier | Description | Status |
|------|-------------|--------|
| Tier 1 – Protocol Works | Identity, binding, idempotency, broker, conversation | ✅ Frozen |
| Tier 2 – Protocol Is Precise | Profiles, references, tenant isolation, review, proposal/decision | ✅ Frozen |
| Tier 3 – Protocol Is Publishable | Threat model, versioning, schemas, bindings, conformance | ✅ Frozen (v0.1) |
| Tier 4 – Governance‑Rich | Obligations, data handling, side‑effect classes, budgets, kill‑switch, trajectory auth | 🔜 v0.2 |
| Tier 5 – Ecosystem‑Ready | Federation, formal registry, advanced regulated profiles | 🔜 v0.3+ |

See [`CHANGES.md`](./CHANGES.md) and [`V0.2_CANDIDATES.md`](./V0.2_CANDIDATES.md).

---

## License and IPR

OACP v0.1 is published under the **Apache License 2.0**.  
A formal IPR policy with patent non‑assertion commitments is deferred until v1.0. See [`IPR.md`](./IPR.md).

---

## Contact

- For questions or proposals: open a [GitHub Discussion](https://github.com/HrPommala/oacp/discussions).  
- For security issues: follow [`SECURITY.md`](./SECURITY.md) – **do not open a public issue**.

---

**OACP is a working draft. We welcome your feedback and contributions.**
