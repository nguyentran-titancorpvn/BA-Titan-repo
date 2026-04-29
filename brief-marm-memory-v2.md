# Implementation Brief: MARM 2.0 Memory

**Module:** knowledge-memory-service | **Priority:** P0 | **Milestone:** M1
**Status:** In Review (interim — 15 items pending product team)
**Version:** v1 (sandbox, not yet published)
**FRs:** SRS-FR-MARM-2.001 to SRS-FR-MARM-2.011 (11 FRs)
**Test Cases:** [tc-marm-memory.md](../test-cases/tc-marm-memory.md)
**Last revised:** 2026-04-29 (re-validation of v1; addresses R-001..R-004, R-006, R-007)

## Glossary

Terms and acronyms used throughout this brief:

| Term | Meaning |
|---|---|
| **MARM** | Memory Anchor Retrieval Module — the persistent memory subsystem of AYITA |
| **DTX** | Product team (reviewing the proposed solutions for the gaps surfaced in this brief) |
| **M1 (Backbone)** | Milestone 1 — the foundational layer of Epic 3 delivery; must ship before dependent milestones |
| **Alpha gate** | System-level sign-off checkpoint at which P0 ACs + specific Success Criteria must pass |
| **R_final** | Composite risk score assembled across the DP-Risk pipeline; gates memory writes, retrieval filters, and response modes |
| **POL-RET-001** | Audit-retention policy — defines what must be logged and how long it must be retained for anchor operations |
| **LoreBook** | Canonical long-term memory store for IDENTITY, PREFERENCE, FACT, and RULE entries (source for mirror anchors) |
| **Circuit closure** | Full query → retrieval → context assembly → response → trace loop, required for SC-08 sign-off |
| **Canonical context snapshot** | The ordered, deterministic anchor + LoreBook bundle delivered to the model for a given query; identical inputs MUST produce an identical hash (per SRS-AC-P0-02) |

## Executive Summary

MARM 2.0 Memory is AYITA's persistent memory substrate — it extracts, stores, and manages typed memory anchors from user conversations and governs their full lifecycle (creation, decay, promotion, demotion, archival). Anchors enable the AI to remember facts, decisions, goals, and preferences across sessions without relying on model context. All 11 P0 FRs are M1 prerequisites; without them, the "Memory Loop" in AYITA's cognitive circuit cannot close.

This v2 re-validation (2026-04-29) confirms no SRS / domain-model / mock-API / mockup changes since v1 (2026-04-21) — sources stable. The 15 interim decisions D-001..D-015 remain 🟡 (no DTX response yet); confirm-by dates carry forward to 2026-05-27. Six Pending Revisions (R-001..R-004, R-006, R-007) targeting this brief have been addressed in v2: SRS-AC-P0-02 and SRS-AC-P0-03 are now mapped, SRS-AC-P0-01 verbatim quote corrected, BA-AC-MARM-022 (user-initiated soft delete) and BA-AC-MARM-023 (canonical context hash determinism) added, mock API path corrected to `routers/memory.py`, BR Coverage applied with Pass 0.5 categorization. Q1 (definition of "access") remains the lone open question with a working interpretation.

## Scope

### In Scope
- Anchor lifecycle state machine — `ACTIVE → DECAYING → DORMANT → EXPIRED → DELETED` (+ `STALE` for mirrors). [SRS-FR-MARM-2.001](#srs-fr-marm-2001)
- Per-type decay function and TTL management. [SRS-FR-MARM-2.002](#srs-fr-marm-2002), [SRS-FR-MARM-2.003](#srs-fr-marm-2003)
- Promotion and demotion between lifecycle states. [SRS-FR-MARM-2.004](#srs-fr-marm-2004), [SRS-FR-MARM-2.005](#srs-fr-marm-2005)
- Context Broker anchor assembly, scoring, capacity enforcement, **and canonical-snapshot hash determinism**. [SRS-FR-MARM-2.006](#srs-fr-marm-2006), [SRS-FR-MARM-2.007](#srs-fr-marm-2007)
- Audit trail for anchor operations. [SRS-FR-MARM-2.008](#srs-fr-marm-2008)
- L1 Session Memory (volatile) and L2 User Memory (persistent). [SRS-FR-MARM-2.009](#srs-fr-marm-2009), [SRS-FR-MARM-2.010](#srs-fr-marm-2010)
- Scheduled cleanup — TTL expiry, state transitions, quota enforcement. [SRS-FR-MARM-2.011](#srs-fr-marm-2011)
- User-facing Memory page — anchor CRUD, verification/rejection, pinning, settings, **user-initiated soft delete (incl. EXPIRED → DELETED per D-014)**.
- 12-type anchor taxonomy (4 mirrors + 8 independent). 🟡 — see [Data Model section](#business-level-data-model) and D-015.

### Out of Scope
- LoreBook CRUD operations themselves (canonical storage layer) — see [brief-lorebook-crud.md](brief-lorebook-crud.md).
- Cognitive Trace immutable audit writer — this brief produces trace *events*; the trace storage/reader is [brief-system-guarantees.md](brief-system-guarantees.md).
- Context Navigator advanced features — only FR-CTX-001/003/004 are in P0 scope (see D-008); FR-CTX-002/005-008 deferred to Pilot Ready. Their detailed specs live in [brief-system-guarantees.md](brief-system-guarantees.md), not here.
- Per-type `dormant_threshold` values for GOAL/TASK/DECISION/FACT — accepted as P1 follow-up per D-002.
- Full §3.5.6 promotion criteria (10 uses across 2 sessions, 30 days VERIFIED, confidence ≥ 0.85) — P0 uses simplified 3-occurrence threshold per FR-MARM-2.004 text.
- Semantic conflict detection requiring vector similarity computation — P1 if vector similarity infrastructure not ready in M1.
- L3 Shared Memory (project-scoped sharing with ACL enforcement) — P1 (FR-MARM-2.101 / AC-P1-12).

## Stability Legend

Every AC, business rule, and GUI Spec row in this brief carries a status icon.

| Icon | Meaning | Dev action |
|---|---|---|
| 🟢 **Stable** | Product team has confirmed this behavior | Build freely |
| 🟡 **Interim** | Team-agreed, awaiting product team confirmation | Build, but keep reversible |
| 🔴 **Open** | No team consensus | Do not build — flagged in Open Questions |

🟢 may only be granted when (a) product team has confirmed AND (b) every numeric value cited in Spec Evidence agrees across cited sources (Numeric Reconciliation passed). A product-confirmed item with unresolved numeric disagreement remains 🟡.

**This brief's overall stability:** 🟡 **Interim — 15 team-agreed items pending product team + 1 open question (Q1) with working interpretation.** See [decision-log.md](../decision-log.md) for D-001..D-015 status and [anchor-lifecycle-spec-gaps-and-proposed-solutions-2026-04-20.md](../product-team-proposals/anchor-lifecycle-spec-gaps-and-proposed-solutions-2026-04-20.md) for the external ask.

## Source Traceability

| Source | Location |
|--------|----------|
| SRS §1.5 | [Success Criteria](../../../specs/srs/1.%20PART-1_CHAPTER_1_2/SRS-EPIC-3-1_INTRODUCTION.md) |
| SRS §3.3 | [MARM 2.0 Architecture](../../../specs/srs/2.%20PART-2_CHAPTER_3/SRS-EPIC-3_3.3.md) |
| SRS §3.5 | [Memory Lifecycle & Update Rules](../../../specs/srs/2.%20PART-2_CHAPTER_3/SRS-EPIC-3_3.5.md) |
| SRS §3.6 | [Context Broker](../../../specs/srs/2.%20PART-2_CHAPTER_3/SRS-EPIC-3_3.6.md) |
| SRS §3.7 | [DP-Risk gating](../../../specs/srs/2.%20PART-2_CHAPTER_3/SRS-EPIC-3_3.7.md) |
| SRS §3.8 | [Audit](../../../specs/srs/2.%20PART-2_CHAPTER_3/SRS-EPIC-3_3.8.md) |
| SRS §4.3.3 | [Functional Requirements — MARM 2.0](../../../specs/srs/3.%20PART-3_CHAPTER_4_5_6_7/SRS-EPIC-3-4_FUNCTIONAL_REQUIREMENTS.md) |
| SRS §6.2.1 | [Acceptance Criteria — AC-P0-01..03 (Memory & Context)](../../../specs/srs/3.%20PART-3_CHAPTER_4_5_6_7/SRS-EPIC-3-6_ACCEPTANCE%20CRITERIES.md) |
| Appendix C.2 | [LIFECYCLE_CONFIGS](../../../specs/srs/4.%20PART-4_APPENDIXES/APPENDIX-C_METHODOLOGICAL%20NOTES.md) |
| Appendix D | [Delivery scope](../../../specs/srs/4.%20PART-4_APPENDIXES/) |
| Domain Model | [domain_memory.yaml](../../../specs/domain_model/domain_memory.yaml) |
| Data Model | [004_knowledge_memory_schema.sql](../../../specs/data_model/grouped/004_knowledge_memory_schema.sql) |
| Mockup source | [frontend2/src/pages/memory/](../../../frontend2/src/pages/memory/) + [frontend2/src/components/](../../../frontend2/src/components/) |
| Mock API | [memory.py router](../../../tools/tools/src/routers/memory.py), [memory_settings.py](../../../tools/tools/src/routers/memory_settings.py) (corrected per R-006: was `routes/*.ts`) |
| Inconsistency log | [srs-inconsistency-log.md](../srs-inconsistency-log.md) |
| Related analysis | [anchor-lifecycle-reference.md](../yaml-summaries/anchor-lifecycle-reference.md) (team-internal gap analysis feeding D-001..D-015) |

## What This Module Does

Provides the persistent memory substrate for AYITA's cognitive layer. Users and the system create typed memory anchors — 12 total types (4 mirrors + 8 independent 🟡 per D-015) covering facts, decisions, goals, tasks, constraints, emotions, preferences, and identity — that persist across dialog sessions. The system automatically extracts anchors from conversations (LLM-based for most types; deterministic pattern-mapping for IDENTITY per D-013), manages their lifecycle through decay and promotion, and makes them available for context assembly so the AI can provide personalized, context-aware responses. Users can manually create, edit, verify, reject, pin, archive, and delete anchors through a dedicated Memory UI. The Context Broker that consumes these anchors must produce a deterministic, identical "canonical context snapshot hash" for repeated runs with identical inputs (per SRS-AC-P0-02).

## Functional Requirements

| FR-ID | Description | Priority | Source | Stability |
|-------|-------------|----------|--------|:-:|
| [SRS-FR-MARM-2.001](#srs-fr-marm-2001) | System SHALL support memory lifecycle: ACTIVE → DECAYING → DORMANT → EXPIRED → DELETED (+ STALE for mirrors) | MUST (P0) | §3.5.1 | 🟡 |
| [SRS-FR-MARM-2.002](#srs-fr-marm-2002) | System SHALL implement TTL for anchors (type-specific default_ttl per YAML `lk_anchor_type`) | MUST (P0) | §3.5.2 | 🟡 |
| [SRS-FR-MARM-2.003](#srs-fr-marm-2003) | System SHALL implement decay function (per-type half-life; DECISION min_weight lowered to 0.20) | MUST (P0) | §3.5.3 | 🟡 |
| [SRS-FR-MARM-2.004](#srs-fr-marm-2004) | System SHALL support promotion (default: 3 occurrences — simplified P0 subset of §3.5.6) | MUST (P0) | §3.5.4 | 🟢 |
| [SRS-FR-MARM-2.005](#srs-fr-marm-2005) | System SHALL support demotion (`non_use_threshold`, per-type) | MUST (P0) | §3.5.5 | 🟡 |
| [SRS-FR-MARM-2.006](#srs-fr-marm-2006) | Context Broker SHALL assemble prompt from anchors by relevance, freshness, confidence, R_final | MUST (P0) | §3.6 | 🟢 |
| [SRS-FR-MARM-2.007](#srs-fr-marm-2007) | Context Broker SHALL enforce max anchors in prompt (default: 10) | MUST (P0) | §3.6.3 | 🟢 |
| [SRS-FR-MARM-2.008](#srs-fr-marm-2008) | System SHALL maintain audit trail for anchor operations conforming to POL-RET-001 | MUST (P0) | §3.8 | 🟢 |
| [SRS-FR-MARM-2.009](#srs-fr-marm-2009) | System SHALL implement L1 Session Memory (volatile) | MUST (P0) | §3.3.3 | 🟢 |
| [SRS-FR-MARM-2.010](#srs-fr-marm-2010) | System SHALL implement L2 User Memory (persistent) | MUST (P0) | §3.3.4 | 🟢 |
| [SRS-FR-MARM-2.011](#srs-fr-marm-2011) | MARM 2.0 cleanup scheduler SHALL execute scheduled cleanup | MUST (P0) | §3.5.6 | 🟢 |

### Spec Evidence

#### Numeric Reconciliation summary (Numeric Reconciliation pre-pass)

Two parameter families appear in ≥2 sources and require reconciliation before the per-FR Spec Evidence below. Single-value items skip this step.

**Per-type `default_ttl`:**

| Source | Parameter | Value (per type) |
|---|---|---|
| YAML `lk_anchor_type` (D-006 target) | `default_ttl` column | NOT YET ADDED to YAML; pending D-006 |
| §3.3.6 prose | "Typical TTL" | DECISION 30d, FACT 7d, GOAL 14d, EMOTION 4h |
| Appendix C.2 `LIFECYCLE_CONFIGS` (verbatim) | `default_ttl` | mirrors 24h, FACT/DECISION 30d, GOAL 14d, TASK 14d, CONSTRAINT 8h, EMOTION 4h |
| FR-MARM-2.002 text | "default: 30 days, type-specific" | 30d global + per-type override |

→ Disagreement: §3.3.6 vs Appendix C.2 vs FR text (already documented in v1, fed D-006/D-010). Items citing this remain 🟡 until DTX confirms YAML-canonical pattern.

**DECISION `min_weight`:**

| Source | Parameter | Value |
|---|---|---|
| Appendix C.2 (pre-decision) | `DECISION.min_weight` | 0.40 |
| D-003 team proposal | `DECISION.min_weight` | 0.20 |

→ Disagreement; 🟡 pending DTX. SRS-FR-MARM-2.003 stays 🟡.

**`non_use_threshold` naming:**

| Source | Parameter | Value (canonical naming) |
|---|---|---|
| §3.5.3 transition table | `dormant_threshold` (default 7d) | one global value |
| YAML `memory_config` | `demotion_after_non_use_days` | one global value |
| Appendix C.2 `LIFECYCLE_CONFIGS` | `non_use_threshold` | per-type (2h EMOTION, 4h CONSTRAINT, 3d TASK, 7d GOAL, 14d DECISION/FACT, 30d mirrors) |

→ Three names for the same concept; D-004 selects `non_use_threshold` from Appendix C.2 as canonical. SRS-FR-MARM-2.005 stays 🟡.

**Context Broker `max_anchors_in_prompt`:**

| Source | Parameter | Value |
|---|---|---|
| FR-MARM-2.007 text | `max_anchors_in_prompt` | default 10 |
| Mock API memory_settings (default) | `max_anchors_in_prompt` | 10 |
| Mockup Settings modal (range) | `max_anchors_in_prompt` | min 1, max 50; default 10 |

→ All sources agree on default 10. SRS-FR-MARM-2.007 → 🟢 numeric-clear.

---

<a id="srs-fr-marm-2001"></a>
**SRS-FR-MARM-2.001** 🟡 — state names updated per D-001:
**[SRS §4.3.3.1](../../../specs/srs/3.%20PART-3_CHAPTER_4_5_6_7/SRS-EPIC-3-4_FUNCTIONAL_REQUIREMENTS.md)** — verbatim FR text: "System SHALL support memory lifecycle: new → candidate → confirmed → stale → archived"
**[SRS §3.3.5](../../../specs/srs/2.%20PART-2_CHAPTER_3/SRS-EPIC-3_3.3.md)** — normative state machine: "ACTIVE — Fresh, high relevance [...] DECAYING — Aging, reduced weight [...] DORMANT — Low activity, preserved [...] EXPIRED — Past TTL, pending deletion [...] DELETED — User/admin deleted, pending purge"
**[YAML `lk_anchor_state`](../../../specs/domain_model/domain_memory.yaml)** — codes: `ACTIVE, DECAYING, DORMANT, EXPIRED, DELETED, STALE`. Notes: "Persistent lifecycle states (tier 1). Transient states (NEW, CANDIDATE, CONFIRMED) are runtime-only, not persisted."

> **🟡 Interim decision (2026-04-20):** Use `ACTIVE → DECAYING → DORMANT → EXPIRED → DELETED` (+ STALE) per §3.3.5 and YAML. `NEW/CANDIDATE/CONFIRMED` treated as transient runtime-only states in Anchor Extractor pipeline. FR text to be updated accordingly.
> **Source:** Team analysis in [anchor-lifecycle-reference.md §12.1](../yaml-summaries/anchor-lifecycle-reference.md). §3.3.5 and YAML are normative for schemas.
> **Pending product team on:** Confirm state name canonicalization.
> **Confirm-by:** 2026-05-27.
> **Logged as:** [D-001](../decision-log.md).

<a id="srs-fr-marm-2002"></a>
**SRS-FR-MARM-2.002** 🟡 — per-type TTL per D-006:
**[SRS §4.3.3.1](../../../specs/srs/3.%20PART-3_CHAPTER_4_5_6_7/SRS-EPIC-3-4_FUNCTIONAL_REQUIREMENTS.md)** — verbatim FR text: "System SHALL implement TTL for anchors (default: 30 days, type-specific)"
**[SRS §3.5.3](../../../specs/srs/2.%20PART-2_CHAPTER_3/SRS-EPIC-3_3.5.md)** — "Any → EXPIRED: `weight < min_weight` OR `now > expires_at`"
**[Appendix C.2 `LIFECYCLE_CONFIGS.default_ttl`](../../../specs/srs/4.%20PART-4_APPENDIXES/APPENDIX-C_METHODOLOGICAL%20NOTES.md)** — per-type values (24h mirrors, 30d FACT/DECISION, 14d TASK/GOAL, 8h CONSTRAINT, 4h EMOTION)

> **🟡 Interim decision (2026-04-20):** `default_ttl` becomes a per-type column on YAML `lk_anchor_type`. §3.3.6 and Appendix C.2/C.2.6 kept as historical reference with pointer to YAML as source of truth.
> **Pending product team on:** Confirm YAML-canonical pattern for TTL values.
> **Confirm-by:** 2026-05-27.
> **Logged as:** [D-006](../decision-log.md), [D-010](../decision-log.md).

<a id="srs-fr-marm-2003"></a>
**SRS-FR-MARM-2.003** 🟡 — DECISION `min_weight` lowered per D-003:
**[SRS §4.3.3.1](../../../specs/srs/3.%20PART-3_CHAPTER_4_5_6_7/SRS-EPIC-3-4_FUNCTIONAL_REQUIREMENTS.md)** — verbatim FR text: "System SHALL implement decay function (default half-life: 7 days)"
**[SRS §3.5.3](../../../specs/srs/2.%20PART-2_CHAPTER_3/SRS-EPIC-3_3.5.md)** — "weight(t) = initial_weight × decay_factor^(t / half_life)"

> **🟡 Interim decision (2026-04-20):** DECISION `min_weight` lowered from 0.40 to 0.20 to create observable DECAYING window (~144h before expiry instead of 0h).
> **Pending product team on:** Confirm parameter change; review other types proportionally.
> **Confirm-by:** 2026-05-27.
> **Logged as:** [D-003](../decision-log.md).

<a id="srs-fr-marm-2004"></a>
**SRS-FR-MARM-2.004:**
**[SRS §4.3.3.1](../../../specs/srs/3.%20PART-3_CHAPTER_4_5_6_7/SRS-EPIC-3-4_FUNCTIONAL_REQUIREMENTS.md)** — verbatim FR text: "System SHALL support promotion (default: 3 occurrences)"
**[SRS §3.5.6](../../../specs/srs/2.%20PART-2_CHAPTER_3/SRS-EPIC-3_3.5.md)** — Auto-promotion eligibility (full criteria, P1): "Usage count ≥ 10 times across ≥ 2 different sessions [...] Duration in VERIFIED > 30 days [...] Confidence ≥ 0.85"
**[Data model](../../../specs/data_model/grouped/004_knowledge_memory_schema.sql)** — Config param `promotion_threshold`: default 3, min 1, max 10.

<a id="srs-fr-marm-2005"></a>
**SRS-FR-MARM-2.005** 🟡 — `non_use_threshold` canonical name per D-004:
**[SRS §4.3.3.1](../../../specs/srs/3.%20PART-3_CHAPTER_4_5_6_7/SRS-EPIC-3-4_FUNCTIONAL_REQUIREMENTS.md)** — verbatim FR text: "System SHALL support demotion (default: 14 days non-use)"
**[SRS §3.3.5](../../../specs/srs/2.%20PART-2_CHAPTER_3/SRS-EPIC-3_3.3.md)** — "DECAYING → DORMANT: No access for configured period"
**[Appendix C.2](../../../specs/srs/4.%20PART-4_APPENDIXES/APPENDIX-C_METHODOLOGICAL%20NOTES.md)** — per-type `non_use_threshold` (2h EMOTION, 4h CONSTRAINT, 3d TASK, 7d GOAL, 14d DECISION/FACT)

> **🟡 Interim decision (2026-04-20):** `non_use_threshold` (from Appendix C.2) is the canonical name. §3.5.3's `dormant_threshold` and YAML's `demotion_after_non_use_days` retired or repurposed.
> **Pending product team on:** Confirm naming + per-type values.
> **Confirm-by:** 2026-05-27.
> **Logged as:** [D-004](../decision-log.md).

<a id="srs-fr-marm-2006"></a>
**SRS-FR-MARM-2.006:**
**[SRS §4.3.3.1](../../../specs/srs/3.%20PART-3_CHAPTER_4_5_6_7/SRS-EPIC-3-4_FUNCTIONAL_REQUIREMENTS.md)** — verbatim FR text: "Context Broker SHALL assemble prompt from anchors by relevance, freshness, confidence, R_final"
**[SRS §3.3.4](../../../specs/srs/2.%20PART-2_CHAPTER_3/SRS-EPIC-3_3.3.md)** — "Context Broker: Prompt assembly from multiple memory sources + Skills artifacts"
**[SRS §3.5.4](../../../specs/srs/2.%20PART-2_CHAPTER_3/SRS-EPIC-3_3.5.md)** — Scope precedence: "Narrowest scope first (SESSION > DIALOG > PROJECT > USER > GLOBAL)"

<a id="srs-fr-marm-2007"></a>
**SRS-FR-MARM-2.007:**
**[SRS §4.3.3.1](../../../specs/srs/3.%20PART-3_CHAPTER_4_5_6_7/SRS-EPIC-3-4_FUNCTIONAL_REQUIREMENTS.md)** — verbatim FR text: "Context Broker SHALL enforce max anchors in prompt (default: 10)"
**Mockup source** — Memory Settings modal, Context Assembly section: "Max anchors in prompt" input (min 1, max 50; default 10) — `frontend2/src/pages/memory/` Settings modal. Numeric Reconciliation: all three sources agree on default = 10.

<a id="srs-fr-marm-2008"></a>
**SRS-FR-MARM-2.008:**
**[SRS §4.3.3.1](../../../specs/srs/3.%20PART-3_CHAPTER_4_5_6_7/SRS-EPIC-3-4_FUNCTIONAL_REQUIREMENTS.md)** — verbatim FR text: "System SHALL maintain audit trail for anchor operations conforming to POL-RET-001"

<a id="srs-fr-marm-2009"></a>
**SRS-FR-MARM-2.009:**
**[SRS §4.3.3.1](../../../specs/srs/3.%20PART-3_CHAPTER_4_5_6_7/SRS-EPIC-3-4_FUNCTIONAL_REQUIREMENTS.md)** — verbatim FR text: "System SHALL implement L1 Session Memory (volatile)"
**[SRS §3.3.1](../../../specs/srs/2.%20PART-2_CHAPTER_3/SRS-EPIC-3_3.3.md)** — "LAYER 1: ANCHORS (Working Memory) — Session-specific goals, constraints, decisions [...] Persistence: Minutes to weeks (policy-driven per type)"

<a id="srs-fr-marm-2010"></a>
**SRS-FR-MARM-2.010:**
**[SRS §4.3.3.1](../../../specs/srs/3.%20PART-3_CHAPTER_4_5_6_7/SRS-EPIC-3-4_FUNCTIONAL_REQUIREMENTS.md)** — verbatim FR text: "System SHALL implement L2 User Memory (persistent)"
**[SRS §3.3.1](../../../specs/srs/2.%20PART-2_CHAPTER_3/SRS-EPIC-3_3.3.md)** — "LAYER 2: LOREBOOK (Canonical Long-term Memory) — User identity (name, role, organization) — CANONICAL SOURCE [...] Persistence: Indefinite (user-controlled)"

<a id="srs-fr-marm-2011"></a>
**SRS-FR-MARM-2.011:**
**[SRS §4.3.3.1](../../../specs/srs/3.%20PART-3_CHAPTER_4_5_6_7/SRS-EPIC-3-4_FUNCTIONAL_REQUIREMENTS.md)** — verbatim FR text: "MARM 2.0 cleanup scheduler SHALL execute scheduled cleanup"
**[SRS §3.3.4](../../../specs/srs/2.%20PART-2_CHAPTER_3/SRS-EPIC-3_3.3.md)** — "Memory Manager: Timer, Every 5 min (configurable) — GC cycle: expire TTL, transition states, enforce quotas"

## Business Rules

> All BRs in v2 carry a category tag per Pass 0.5 (R-007). Verification route per category: [FUNCTIONAL] → ≥1 BA-AC; [CROSS-CUTTING] → cited in ≥1 TC; [INFRA]/[POLICY] → audit/attestation pointer.

1. `[FUNCTIONAL]` **Two orthogonal status axes** 🟡 (D-001) — Anchors have two independent dimensions: lifecycle **state** (`ACTIVE/DECAYING/DORMANT/EXPIRED/DELETED` + `STALE` for invalidated mirrors) governs decay and retrieval eligibility; verification **status** (`PENDING/VERIFIED/PROMOTED/REJECTED`) governs trust level and promotion eligibility. Both are independent. State names per D-001. — [SRS §3.5.2](../../../specs/srs/2.%20PART-2_CHAPTER_3/SRS-EPIC-3_3.5.md), [Domain model DD-68](../../../specs/domain_model/domain_memory.yaml). Canonical state list: `lk_anchor_state` (6 codes).

2. `[FUNCTIONAL]` **Memory Isolation Principle** 🟢 — Anchors are personal-only by default (`visibility=PRIVATE`). Team memory flows through LoreBook, not through shared anchors. Context assembly order: personal anchors → canon (LoreBook) → documents (KB/RAG), filtered by ACL. — [Domain model DD-91](../../../specs/domain_model/domain_memory.yaml)

3. `[FUNCTIONAL]` **DELETED is terminal for system transitions** 🟡 — Once an anchor is system-transitioned to DELETED, it cannot transition to any other state. **Exception per D-014:** users can explicitly delete an EXPIRED anchor (permits user-initiated EXPIRED → DELETED transition). Purge happens after retention period (default 365 days). — [SRS §3.3.5](../../../specs/srs/2.%20PART-2_CHAPTER_3/SRS-EPIC-3_3.3.md). Canonical state list: `lk_anchor_state`.

4. `[FUNCTIONAL]` **Mirror anchor taxonomy and inheritance** 🟡 (D-007, D-010, D-015) — Four mirror types reflect canonical LoreBook entries: IDENTITY_MIRROR, PREFERENCE_MIRROR, FACT_MIRROR, RULE_MIRROR. Mirror types MUST have `canonical_ref` NOT NULL; independent types MUST have `canonical_ref` NULL. Mirrors have short TTL (24h) — regenerated from LoreBook on expiry. Mirrors invalidate (→ STALE) when source LoreBook entry is modified. **FACT_MIRROR and RULE_MIRROR inherit lifecycle parameters (incl. `initial_weight`) from the source anchor at promotion time** (per D-007). — [SRS §3.5.3](../../../specs/srs/2.%20PART-2_CHAPTER_3/SRS-EPIC-3_3.5.md), [Domain model](../../../specs/domain_model/domain_memory.yaml). Canonical mirror list: `lk_anchor_type` (4 mirrors of 10 entries today; 4-of-12 post-D-015).

5. `[FUNCTIONAL]` **Low-confidence extraction guard** 🟢 — Anchors with confidence < 0.7 receive `verification_status = PENDING` and are excluded from default retrieval (included only if `include_pending=true`). Model-generated anchors require confidence ≥ 0.85 for VERIFIED status. — [SRS §3.5.7.1](../../../specs/srs/2.%20PART-2_CHAPTER_3/SRS-EPIC-3_3.5.md)

6. `[FUNCTIONAL]` **Rate limiting on extraction** 🟢 — Maximum 20 new anchors per dialog turn to prevent anchor flooding. Configurable via `max_anchors_per_turn` (range 1-50). — [SRS §3.5.7.1](../../../specs/srs/2.%20PART-2_CHAPTER_3/SRS-EPIC-3_3.5.md)

7. `[FUNCTIONAL]` **Scope precedence in retrieval** 🟢 — When multiple anchors match on the same semantic key, narrowest scope wins: `SESSION > DIALOG > PROJECT > USER > GLOBAL`. Within same scope, conflict resolution applies: higher confidence wins, then more recent, then deterministic hash. — [SRS §3.5.4](../../../specs/srs/2.%20PART-2_CHAPTER_3/SRS-EPIC-3_3.5.md)

8. `[FUNCTIONAL]` **Memory budget enforcement** 🟢 — Budget limits by scope: Session max 50 active, Dialog max 200, Project max 500, User max 1000. Backpressure: at 80-95% capacity increase decay 1.5x for PENDING; at 95-100% pause extraction; at 100% evict lowest-scored before new insertion. — [SRS §3.5.8](../../../specs/srs/2.%20PART-2_CHAPTER_3/SRS-EPIC-3_3.5.md)

9. `[FUNCTIONAL]` **Anchor Extractor classification** 🟡 (D-013) — LLM-based classification for seven independent types (DECISION, FACT, GOAL, TASK, CONSTRAINT, EMOTION, PREFERENCE); deterministic pattern mapping for IDENTITY (regex / structured extraction on name, role, location, contact information) feeding LoreBook IDENTITY entry. Canonical independent-type list: `lk_anchor_type` codes where `is_mirror=false` (current YAML: 6 entries; post-D-015: 8 — adds IDENTITY, PREFERENCE). — [SRS §3.3.1](../../../specs/srs/2.%20PART-2_CHAPTER_3/SRS-EPIC-3_3.3.md), [domain_memory.yaml `lk_anchor_type`](../../../specs/domain_model/domain_memory.yaml)

10. `[CROSS-CUTTING]` **Weight invariant** 🟢 — Current weight MUST NOT exceed `initial_weight`. Enforced by database CHECK constraint: `weight <= initial_weight`. Applies to every anchor read/write/decay computation. — [Data model](../../../specs/data_model/grouped/004_knowledge_memory_schema.sql)

11. `[FUNCTIONAL]` **Anchor content length limit** 🟢 — Maximum anchor content length: 500 characters to prevent context stuffing. — [SRS §3.5.7.4](../../../specs/srs/2.%20PART-2_CHAPTER_3/SRS-EPIC-3_3.5.md)

12. `[FUNCTIONAL]` **Conflict resolution** 🟢 — When two anchors conflict on the same semantic content: (1) compare confidence — higher wins; (2) if tied, compare timestamps — newer wins; (3) if still tied, use `selection_seed` for deterministic choice. Conflicting anchors increase mutual decay rate by 2x. Max 3 unresolved conflicts per semantic key. — [SRS §3.3.4](../../../specs/srs/2.%20PART-2_CHAPTER_3/SRS-EPIC-3_3.3.md), [SRS §3.5.5](../../../specs/srs/2.%20PART-2_CHAPTER_3/SRS-EPIC-3_3.5.md)

13. `[CROSS-CUTTING]` **DP-Risk policy integration** 🟡 (D-005) — DP-Risk thresholds gate memory writes and retrieval. Pending consolidation into a single canonical SRS table, the P0 implementation uses: `dp_risk > 0.7` → SESSION anchors only, no promotion, no LoreBook writes; `dp_risk > 0.9` → no memory writes at all. Applies to every write path (extraction, manual create, promotion). — [SRS §3.3.4](../../../specs/srs/2.%20PART-2_CHAPTER_3/SRS-EPIC-3_3.3.md), [SRS §3.7.2](../../../specs/srs/2.%20PART-2_CHAPTER_3/SRS-EPIC-3_3.7.md)

14. `[FUNCTIONAL]` **Auto-extracted anchors are read-only content** 🟢 — Content of auto-extracted (`source=EXTRACTED`) anchors cannot be edited by the user. Users must reject and recreate manually to correct content. Scope and metadata (tags, expiration) remain editable. — [frontend2 EditAnchorModal](../../../frontend2/src/pages/memory/EditAnchorModal/EditAnchorModal.tsx)

15. `[FUNCTIONAL]` **TASK and CONSTRAINT non-promotable** 🟡 (D-012) — TASK and CONSTRAINT are non-promotable because they are not in §3.5.6.1's eligible list. Canonical eligible list: §3.5.6.1 (DECISION, FACT, GOAL, IDENTITY-promotion-via-mirror, PREFERENCE-promotion-via-mirror per D-015). No YAML change required to codify this; behavior is controlled by the eligibility list. — [SRS §3.5.6.1](../../../specs/srs/2.%20PART-2_CHAPTER_3/SRS-EPIC-3_3.5.md), [D-012 entry](../decision-log.md)

16. `[CROSS-CUTTING]` **Canonical context snapshot determinism** 🟢 — For repeated runs with identical inputs (same dialog state, anchor pool, snapshot_time, query, R_final inputs), the Context Broker MUST produce an identical canonical snapshot ordering and an identical hash. Hash is computed over the ordered selected anchor IDs + LoreBook IDs + truncation policy. Cited in TCs covering BA-AC-MARM-023. — [SRS §6.2.1 AC-P0-02](../../../specs/srs/3.%20PART-3_CHAPTER_4_5_6_7/SRS-EPIC-3-6_ACCEPTANCE%20CRITERIES.md), R-003

17. `[INFRA]` **Encryption at rest for anchor content** 🟢 — Anchor content and LoreBook entries at rest SHALL conform to platform encryption policy. No AC required; verified by infra audit + DoD attestation. — [SRS §3.8](../../../specs/srs/2.%20PART-2_CHAPTER_3/SRS-EPIC-3_3.8.md). Audit surface: POL-ENC-001 (platform-level) + DoD attestation item below.

18. `[POLICY]` **Audit retention POL-RET-001** 🟢 — All anchor operations (create/update/state-transition/delete/promote/reject) emit audit events retained per POL-RET-001 (see FR-MARM-2.008). No AC required; verified by policy attestation. — [SRS §3.8](../../../specs/srs/2.%20PART-2_CHAPTER_3/SRS-EPIC-3_3.8.md). Audit surface: POL-RET-001 attestation in DoD.

## BR Coverage

| BR | Category | Covered by |
|---|---|---|
| BR-1 Two orthogonal status axes | [FUNCTIONAL] | [BA-AC-MARM-001](#ba-ac-marm-001), [BA-AC-MARM-014](#ba-ac-marm-014), [BA-AC-MARM-015](#ba-ac-marm-015) |
| BR-2 Memory Isolation Principle | [FUNCTIONAL] | [BA-AC-MARM-016](#ba-ac-marm-016) |
| BR-3 DELETED is terminal (with D-014 user exception) | [FUNCTIONAL] | [BA-AC-MARM-012](#ba-ac-marm-012), [BA-AC-MARM-022](#ba-ac-marm-022) |
| BR-4 Mirror anchor taxonomy and inheritance | [FUNCTIONAL] | [BA-AC-MARM-021](#ba-ac-marm-021) |
| BR-5 Low-confidence extraction guard | [FUNCTIONAL] | [BA-AC-MARM-014](#ba-ac-marm-014), [BA-AC-MARM-007](#ba-ac-marm-007) |
| BR-6 Rate limiting on extraction | [FUNCTIONAL] | [BA-AC-MARM-018](#ba-ac-marm-018) |
| BR-7 Scope precedence in retrieval | [FUNCTIONAL] | [BA-AC-MARM-007](#ba-ac-marm-007) |
| BR-8 Memory budget enforcement | [FUNCTIONAL] | [BA-AC-MARM-018](#ba-ac-marm-018) |
| BR-9 Anchor Extractor classification | [FUNCTIONAL] | [BA-AC-MARM-013](#ba-ac-marm-013) (manual path; LLM extraction in D-013 covered by BA-AC-MARM-014/015) |
| BR-10 Weight invariant | [CROSS-CUTTING] | Cited as precondition / DB CHECK in BA-TC-MARM-004, 008, 020 |
| BR-11 Anchor content length limit | [FUNCTIONAL] | [BA-AC-MARM-019](#ba-ac-marm-019) |
| BR-12 Conflict resolution | [FUNCTIONAL] | [BA-AC-MARM-020](#ba-ac-marm-020) |
| BR-13 DP-Risk policy integration | [CROSS-CUTTING] | Cited as precondition in BA-TC-MARM-007, 013 (write-path gating) |
| BR-14 Auto-extracted anchors are read-only content | [FUNCTIONAL] | [BA-AC-MARM-013](#ba-ac-marm-013), [BA-AC-MARM-014](#ba-ac-marm-014) |
| BR-15 TASK and CONSTRAINT non-promotable | [FUNCTIONAL] | [BA-AC-MARM-005](#ba-ac-marm-005) |
| BR-16 Canonical context snapshot determinism | [CROSS-CUTTING] | Cited as precondition in BA-TC-MARM-023; covered functionally by [BA-AC-MARM-023](#ba-ac-marm-023) |
| BR-17 Encryption at rest | [INFRA] | POL-ENC-001 platform attestation; DoD audit item |
| BR-18 Audit retention POL-RET-001 | [POLICY] | POL-RET-001 attestation; DoD audit item; BA-AC-MARM-009 verifies emission |

## Business-Level Data Model

An **Anchor** is the atomic unit of MARM memory. It belongs to one User (owner) within a Workspace, has a type (**12 types: 4 mirrors + 8 independent** 🟡 per D-015 — current YAML has 10), a lifecycle state (ACTIVE/DECAYING/DORMANT/EXPIRED/DELETED/STALE), a verification status (PENDING/VERIFIED/PROMOTED/REJECTED), and a scope (SESSION/DIALOG/PROJECT/USER/GLOBAL). Anchors carry a confidence score (extraction reliability), a weight (current relevance, decaying over time), and an `initial_weight` (type-dependent starting value).

**Anchor type taxonomy (12 types)** 🟡 per D-015:

| Category | Types | Behavior |
|---|---|---|
| Mirrors (4) | IDENTITY_MIRROR, PREFERENCE_MIRROR, FACT_MIRROR, RULE_MIRROR | Reflect canonical LoreBook entries; short TTL (24h); regenerate from LoreBook; params inherited from source (FACT_MIRROR, RULE_MIRROR per D-007) |
| Independent (8) | IDENTITY *(new — D-015)*, PREFERENCE *(new — D-015)*, GOAL, TASK, CONSTRAINT, DECISION, FACT, EMOTION | Extracted from conversation (LLM or deterministic — D-013); promotable types promote into LoreBook or respective mirror |

> **🟡 Interim decision (2026-04-20):** Expanded taxonomy from YAML's current 10 types (4 mirrors + 6 independent) to 12 types by adding IDENTITY and PREFERENCE as new independent types that promote into their respective mirrors. Completes symmetry: every mirror has a matching source independent type.
> **Source:** Team analysis of FR-MARM-2.012 ("9 types"), §3.3.6 (8 types), YAML (10 types) — three-way inconsistency. Canonical enumeration source: `lk_anchor_type` in domain_memory.yaml (currently 10 codes; D-015 adds 2).
> **Pending product team on:** Confirm 12-type taxonomy or pick different scope.
> **Confirm-by:** 2026-05-27.
> **Logged as:** [D-015](../decision-log.md).

A **LoreBook Entry** is a canonical long-term semantic fact (IDENTITY, PREFERENCE, FACT, or RULE). LoreBook entries use logical versioning (all versions share the same `logical_id`). Mirror-type anchors point to LoreBook entries via `canonical_ref`.

**Anchor Evidence** records provenance — which dialog message or file sourced the anchor. An anchor may have multiple evidence records (reinforced across turns).

**Anchor Relations** form typed edges between anchors: RELATED, SUPERSEDES, CONTRADICTS, SUPPORTS — forming a knowledge graph foundation.

**Memory Config** uses EAV-pattern configuration with scope-based resolution (USER > WORKSPACE > TENANT > INSTANCE > SYSTEM — most-specific wins).

[Full domain model](../../../specs/domain_model/domain_memory.yaml) | [Full data model](../../../specs/data_model/grouped/004_knowledge_memory_schema.sql)

## UI Behavior — GUI Spec Table

| ID | Screen/Area | Field Name | Description | Control Type | Data Type | Default | Required | Rules & Validation | Source FR | Mockup Ref | Visible When | Stability |
|----|-------------|-----------|-------------|-------------|-----------|---------|----------|-------------------|-----------|------------|-------------|:-:|
| 1 | Sidebar | Overview Section | Total count + User-created count | OverviewSection | — | Total selected | — | Total uses unfiltered stats; User-created filters by `source=MANUAL` | — | `frontend2/src/pages/memory/`, left panel | Always | 🟢 |
| 2 | Sidebar | Type Filter | Filter by anchor type (grouped) | SidebarFilter | Enum (multi) | All Types | N | Groups: Fact (FACT + FACT_MIRROR), Goal, Task, Constraint (CONSTRAINT + RULE_MIRROR), Decision, Emotion, **Identity (IDENTITY + IDENTITY_MIRROR)** 🟡, **Preference (PREFERENCE + PREFERENCE_MIRROR)** 🟡. Mirror types hidden from user, counted under base type | — | left panel | Always | 🟡 (D-015) |
| 3 | Sidebar | Verification Filter | Filter by verification status | SidebarFilter | Enum | All | N | Options: All, Pending, Verified, Rejected. PROMOTED hidden from user, merged into Verified count | — | left panel | Always | 🟢 |
| 4 | Sidebar | State Filter | Filter by lifecycle state | SidebarFilter | Enum | All | N | Options: All, Active, Decaying, Dormant, **Expired** 🟡, Stale. DELETED hidden from user. **Expired now shown per D-014** (previously hidden by mock API default) | — | left panel | Always | 🟡 (D-014) |
| 5 | Toolbar | Search | Search anchors by text | TextInput | String | "" | N | Debounced, searches title + content | — | top | Always | 🟢 |
| 6 | Toolbar | Filter Dropdown | Additional toolbar filters | DropdownMenu | — | — | N | Toolbar-level filter options | — | top | Always | 🟢 |
| 7 | Toolbar | Sort Dropdown | Sort anchors | SortDropdown | Enum | — | N | Sort by: updated_at, created_at, confidence, weight, type | — | top | Always | 🟢 |
| 8 | Toolbar | View Mode Toggle | Grid or List view | ToggleModeButtons | Enum | grid | — | Options: Grid, List | — | top | Always | 🟢 |
| 9 | Toolbar | Settings Button | Open Memory Settings modal | Button | — | — | — | Opens MemorySettingsModal | — | top | Always | 🟢 |
| 10 | Toolbar | More Options | Import, Export, Clear All | DropdownMenu | — | — | — | Import: JSON file upload. Export: JSON download. Clear All: requires confirmation modal | — | top | Always | 🟢 |
| 11 | Toolbar | Add Entry Button | Create new anchor manually | Button | — | — | — | Opens NewAnchorModal | — | top-right | Always | 🟢 |
| 12 | Content Area | Anchor Card (Grid) | Displays anchor in card format | Card | Object | — | — | Shows: type badge, state badge, verification badge, LoreBook badge, scope badge, pin icon, title, content, tags, confidence %, weight, expiration, creator, updated_at | [SRS-FR-MARM-2.010](#srs-fr-marm-2010) | center | viewMode=grid | 🟢 |
| 13 | Content Area | Anchor Table (List) | Displays anchors in table format | SortableTable | Array | — | — | Columns: Type, Content, Confidence, Verification, Source, Updated, Scope. Sortable by column header click | [SRS-FR-MARM-2.010](#srs-fr-marm-2010) | center | viewMode=list | 🟢 |
| 14 | Content Area | Empty State | No anchors exist | EmptyState | — | — | — | Shows icon, title "No memory entries yet", description, "Add Entry" button | — | center | When 0 anchors | 🟢 |
| 15 | New Anchor Modal | Type | Anchor type selection | Select | Enum | First option | Y | User-creatable types: 8 independent types per D-015 (GOAL, TASK, CONSTRAINT, DECISION, FACT, EMOTION, **IDENTITY**, **PREFERENCE**). Mirrors excluded | [SRS-FR-MARM-2.010](#srs-fr-marm-2010) | Modal | When modal open | 🟡 (D-015) |
| 16 | New Anchor Modal | Scope | Anchor scope | Select | Enum | USER | Y | Options: Session, User, Project (Dialog filtered out on Memory page). Shows contextual hint text per scope | [SRS-FR-MARM-2.009](#srs-fr-marm-2009) | Modal | When modal open | 🟢 |
| 17 | New Anchor Modal | Project | Project selector | Select | UUID | Global selector value | Conditional | Required when scope=PROJECT. Shows project list from user's projects | [SRS-FR-MARM-2.010](#srs-fr-marm-2010) | Modal | When scope=PROJECT | 🟢 |
| 18 | New Anchor Modal | Title | Anchor display title | TextInput | String | "" | N | Max 200 chars | [SRS-FR-MARM-2.010](#srs-fr-marm-2010) | Modal | When modal open | 🟢 |
| 19 | New Anchor Modal | Content | Anchor semantic content | TextArea | String | "" | Y | Cannot be empty. Max 500 chars (BR-11). This is what MARM "remembers" | [SRS-FR-MARM-2.010](#srs-fr-marm-2010) | Modal | When modal open | 🟢 |
| 20 | New Anchor Modal | Expiration | TTL policy | Select | Enum | First option | Y | Options: Session, Day, Week, Month, Quarter, Year, Auto (per-type from YAML 🟡 D-006) | [SRS-FR-MARM-2.002](#srs-fr-marm-2002) | Modal | When modal open | 🟡 (D-006) |
| 21 | New Anchor Modal | Tags | Freeform tags | TagInput | Array | [] | N | Enter/comma to add, Backspace to remove last. Autocomplete from existing tags | — | Modal | When modal open | 🟢 |
| 22 | New Anchor Modal | Confidence | Confidence slider | ProgressSlider | Number | 100 | Y | Range 0-100%, stored as 0.0-1.0 | — | Modal | When modal open | 🟢 |
| 23 | Edit Anchor Modal | Type Dropdown | Change anchor type | AnchorTypeDropdown | Enum | Current type | — | Read-only for non-MANUAL sources | [SRS-FR-MARM-2.010](#srs-fr-marm-2010) | Modal | When modal open | 🟢 |
| 24 | Edit Anchor Modal | Verification Badge | Shows current verification status | Badge | Enum | — | — | Read-only display | — | Modal header | Always | 🟢 |
| 25 | Edit Anchor Modal | Source Badge | Shows creation source | GradientBadge | Enum | — | — | User-created (blue), Auto-extracted (yellow), Imported (slate), LoreBook-linked (emerald), System (slate) | — | Modal header | Always | 🟢 |
| 26 | Edit Anchor Modal | Title | Editable title | TextInput | String | Current | N | Max 200 chars. Read-only for non-MANUAL | [SRS-FR-MARM-2.010](#srs-fr-marm-2010) | Modal | Always | 🟢 |
| 27 | Edit Anchor Modal | Content | Anchor content | TextArea | String | Current | Y | Read-only for non-MANUAL sources (EXTRACTED, IMPORTED, PROMOTED, SYSTEM) (BR-14) | [SRS-FR-MARM-2.010](#srs-fr-marm-2010) | Modal | Always | 🟢 |
| 28 | Edit Anchor Modal | Confidence | Editable confidence | ProgressSlider | Number | Current | — | Read-only for non-MANUAL | — | Modal | Always | 🟢 |
| 29 | Edit Anchor Modal | Scope | Editable scope | Select | Enum | Current | — | Always editable. Project picker shows when scope=PROJECT | [SRS-FR-MARM-2.009](#srs-fr-marm-2009) | Modal | Always | 🟢 |
| 30 | Edit Anchor Modal | Expiration | Editable expiration | Select | Enum | Current | — | Read-only for non-MANUAL | [SRS-FR-MARM-2.002](#srs-fr-marm-2002) | Modal | Always | 🟡 (D-006) |
| 31 | Edit Anchor Modal | Weight | Current / Initial display | ReadOnly | String | — | — | Format: "0.85 / 1.00". Color-coded: green >0.7, yellow >0.3, red ≤0.3 | [SRS-FR-MARM-2.003](#srs-fr-marm-2003) | Modal | Always | 🟡 (D-003 DECISION floor) |
| 32 | Edit Anchor Modal | State | Lifecycle state display | ReadOnly | String | — | — | Shows display label (Active, Decaying, Dormant, Expired, Stale — DELETED hidden) | [SRS-FR-MARM-2.001](#srs-fr-marm-2001) | Modal | Always | 🟡 (D-001) |
| 33 | Edit Anchor Modal | Version | Version counter | ReadOnly | String | — | — | Format: "v1", "v2" | — | Modal | Always | 🟢 |
| 34 | Edit Anchor Modal | Tags | Editable tags | TagInput | Array | Current | N | Same behavior as New Anchor. Read-only for non-MANUAL in tag removal | — | Modal | Always | 🟢 |
| 35 | Edit Anchor Modal | Source Dialog | Provenance link | Source | Object | — | — | Shows dialog title, timestamp, project name. Clickable link to original message | [SRS-FR-MARM-2.008](#srs-fr-marm-2008) | Modal | When `source_dialog_id` present | 🟢 |
| 36 | Edit Anchor Modal | Verify/Reject Banner | Review prompt for extracted anchors | GradientInfoBox | — | — | — | "AYITA extracted this entry. Please review and confirm or reject." with Reject and Verify buttons | — | Modal | When source=EXTRACTED AND verification=PENDING | 🟢 |
| 37 | Edit Anchor Modal | Archive Button | Archive the anchor | Button | — | — | — | Opens ConfirmArchiveModal | [SRS-FR-MARM-2.005](#srs-fr-marm-2005) | Modal footer | When not showing Restore | 🟢 |
| 38 | Edit Anchor Modal | Restore Button | Restore rejected anchor | Button | — | — | — | Opens ConfirmRestoreModal | — | Modal footer | When source=EXTRACTED AND verification=REJECTED | 🟢 |
| 39 | Edit Anchor Modal | Pin Button | Pin/Unpin anchor | Button | — | — | — | Pinned = always included in context, exempt from decay. Disabled when verification=PENDING | [SRS-FR-MARM-2.010](#srs-fr-marm-2010) | Modal footer | Always | 🟢 |
| 40 | Edit Anchor Modal | Save Changes Button | Save edits | Button | — | — | — | Disabled when no changes detected | [SRS-FR-MARM-2.010](#srs-fr-marm-2010) | Modal footer | When state != DELETED | 🟢 |
| 41 | Edit Anchor Modal | Delete Button | Soft delete anchor | Button | — | — | — | Opens ConfirmDeleteModal. Visible for MANUAL anchors, EXTRACTED+VERIFIED, **and EXPIRED** 🟡 per D-014 (covered by [BA-AC-MARM-022](#ba-ac-marm-022)) | [SRS-FR-MARM-2.001](#srs-fr-marm-2001) | Modal footer | Conditional | 🟡 (D-014) |
| 42 | Settings Modal | Auto-Extraction: Enabled | Toggle auto-extraction | ToggleSwitch | Boolean | true | — | Enable/disable automatic anchor extraction from conversations | [SRS-FR-MARM-2.009](#srs-fr-marm-2009) | Settings modal | Always | 🟢 |
| 43 | Settings Modal | Auto-Extraction: Min Confidence | Minimum confidence threshold | InputNumber | Float | 0.70 | — | Range 0.50-0.99, step 0.05. Anchors below this require manual review (BR-5) | [SRS-FR-MARM-2.003](#srs-fr-marm-2003) | Settings modal | Always | 🟢 |
| 44 | Settings Modal | Auto-Extraction: Max Anchors/Turn | Rate limit per turn | InputNumber | Integer | 20 | — | Range 1-50 (BR-6) | [SRS-FR-MARM-2.009](#srs-fr-marm-2009) | Settings modal | Always | 🟢 |
| 45 | Settings Modal | Auto-Extraction: Suggestion TTL | Review period for suggestions | InputNumber | Integer | 7 | — | Range 1-365 days. Unreviewed suggestions expire after this | — | Settings modal | Always | 🟢 |
| 46 | Settings Modal | Lifecycle: Default TTL | Global fallback TTL (per-type overrides in YAML) | InputNumber | Integer | 30 | — | Range 1-365 days. **Per-type `default_ttl` from YAML takes precedence when set** 🟡 per D-006 | [SRS-FR-MARM-2.002](#srs-fr-marm-2002) | Settings modal | Always | 🟡 (D-006) |
| 47 | Settings Modal | Lifecycle: Decay Half-Life | Weight decay period (global fallback) | InputNumber | Integer | 7 | — | Range 1-90 days. Per-type `half_life` from YAML takes precedence | [SRS-FR-MARM-2.003](#srs-fr-marm-2003) | Settings modal | Always | 🟢 |
| 48 | Settings Modal | Lifecycle: Promotion Threshold | Recurrence for promotion | InputNumber | Integer | 3 | — | Range 1-365 times | [SRS-FR-MARM-2.004](#srs-fr-marm-2004) | Settings modal | Always | 🟢 |
| 49 | Settings Modal | Lifecycle: `non_use_threshold` | Inactivity before demotion (renamed per D-004) | InputNumber | Integer | 14 | — | Range 1-365 days. Per-type values from YAML (2h EMOTION, 4h CONSTRAINT, 3d TASK, 7d GOAL, 14d DECISION/FACT); this is global fallback only | [SRS-FR-MARM-2.005](#srs-fr-marm-2005) | Settings modal | Always | 🟡 (D-004) |
| 50 | Settings Modal | Lifecycle: Cleanup Schedule | Cleanup frequency | Select | Enum | daily | — | Options: Hourly, Daily, Weekly | [SRS-FR-MARM-2.011](#srs-fr-marm-2011) | Settings modal | Always | 🟢 |
| 51 | Settings Modal | Context: Max Anchors in Prompt | Max anchors per LLM request | InputNumber | Integer | 10 | — | Range 1-50 | [SRS-FR-MARM-2.007](#srs-fr-marm-2007) | Settings modal | Always | 🟢 |
| 52 | Settings Modal | Context: Include Dormant | Fallback to dormant anchors | ToggleSwitch | Boolean | false | — | When enabled, dormant anchors used as fallback. Note: DORMANT only reachable for CONSTRAINT/EMOTION in P0 🟡 per D-002 | [SRS-FR-MARM-2.006](#srs-fr-marm-2006) | Settings modal | Always | 🟡 (D-002) |
| 53 | Settings Modal | Context: Privacy-Aware Retrieval | DP-Risk filtering toggle | ToggleSwitch | Boolean | false | — | Apply DP-Risk sensitivity filtering per canonical threshold table 🟡 D-005 | [SRS-FR-MARM-2.006](#srs-fr-marm-2006) | Settings modal | Always | 🟡 (D-005) |
| 54 | Settings Modal | Usage: Active Anchors (User) | Quota usage bar | ProgressBar | — | — | — | Read-only. Shows current / limit | — | Settings modal | Always | 🟢 |
| 55 | Settings Modal | Usage: Active Anchors (Project) | Quota usage bar | ProgressBar | — | — | — | Read-only | — | Settings modal | Always | 🟢 |
| 56 | Settings Modal | Usage: LoreBook Entries | Quota usage bar | ProgressBar | — | — | — | Read-only | — | Settings modal | Always | 🟢 |
| 57 | Settings Modal | Usage: Pending Suggestions | Pending count with review badge | ProgressBar | — | — | — | Read-only. Shows "review" badge when count > 0 | — | Settings modal | Always | 🟢 |
| 58 | Settings Modal | Stats: State Distribution | Active/Decaying/Dormant/LoreBook | StatGroup | — | — | — | Read-only counts per state | — | Settings modal | Always | 🟢 |
| 59 | Settings Modal | Reset Defaults Button | Reset all settings | Button | — | — | — | Requires browser confirm() dialog. Does not affect existing anchors | — | Settings modal footer | Always | 🟢 |

## Dependencies & Interactions

| Direction | Module | Relationship | Stability |
|-----------|--------|-------------|:-:|
| Depends on | Dialog Management (FR-DIALOG-001) | Dialog messages (H0) are the source material for anchor extraction. `source_dialog_id` links anchor to originating dialog | 🟢 |
| Depends on | Async Core (FR-ASYNC-001) | Anchor Extractor runs as async job after STORE stage; Memory Manager GC is a scheduled async job | 🟢 |
| Depends on | Model Registry (FR-MDL-001) | Anchor Extractor uses model inference for typed extraction and classification | 🟢 |
| Depends on | DP-Risk Module (FR-PRIV-*) | Memory Policy Engine consults `dp_risk` score to gate writes and retrieval (canonical threshold table pending 🟡 D-005) | 🟡 (D-005) |
| Feeds into | Context Assembly (FR-CTX-001 §3.6.6, FR-CTX-003 §3.6.8, FR-CTX-004 §3.6.9 — all 🟡 P0 per D-008) | Context Broker reads anchors + LoreBook entries for prompt assembly; canonical hash determinism (BA-AC-MARM-023) is a contract on this interface | 🟡 (D-008) |
| Feeds into | Cognitive Trace (FR-OBS-*) | All anchor operations (create, update, state transition, delete) emit trace events. Alpha-gate uses "light circuit closure" per D-009 | 🟡 (D-009) |
| Feeds into | LoreBook (§3.3.10) | Promotion flow creates LoreBook entries from high-value anchors; mirrors reflect LoreBook back to anchors. IDENTITY/PREFERENCE now promote via mirrors per D-015 | 🟡 (D-015) |
| Feeds into | Skills Framework (FR-SKILL-*) | Skills may create or query anchors (e.g., "remember this" skill) | 🟢 |

## Acceptance Criteria

### From SRS (linked, not duplicated)

> Source of truth: SRS §6.2.1. These ACs are linked, not redefined here. If SRS changes, this section must be re-synced (see Freshness Guard).

Pass 0 enumeration covered three SRS-AC entries that trace to MARM FRs (R-001 fix — v1 only listed AC-P0-01):

<a id="srs-ac-p0-01"></a>
#### SRS-AC-P0-01 🟡: [Happy] Memory lifecycle functional
- **Source SRS:** [SRS §6.2.1 line 102](../../../specs/srs/3.%20PART-3_CHAPTER_4_5_6_7/SRS-EPIC-3-6_ACCEPTANCE%20CRITERIES.md)
- **FRs:** [SRS-FR-MARM-2.001..005](#srs-fr-marm-2001)
- **Verification method:** Integration tests
- **Spec Evidence (verbatim, R-002 fix):** "All 5 MARM states (NEW → CANDIDATE → CONFIRMED → STALE → ARCHIVED) operational"
- **🟡 Note (D-001):** SRS verbatim still uses pre-D-001 state names; the team-agreed canonical state set per D-001 is `ACTIVE/DECAYING/DORMANT/EXPIRED/DELETED + STALE`. Until DTX confirms D-001 (Confirm-by 2026-05-27), this AC is verified against the **D-001 canonical state set** while flagging the SRS quote drift to inconsistency log.
- **Decomposed observable behaviors:**
  - All 5/6 lifecycle states are reachable in test under P0 parameters (DORMANT only via CONSTRAINT/EMOTION per D-002).
  - Each state transition emits a trace event.
  - Each state's effect on retrieval eligibility is testable.
- **Covered by BA-ACs:** [BA-AC-MARM-001](#ba-ac-marm-001), [BA-AC-MARM-002](#ba-ac-marm-002), [BA-AC-MARM-003](#ba-ac-marm-003), [BA-AC-MARM-004](#ba-ac-marm-004), [BA-AC-MARM-006](#ba-ac-marm-006), [BA-AC-MARM-012](#ba-ac-marm-012), [BA-AC-MARM-022](#ba-ac-marm-022).

<a id="srs-ac-p0-02"></a>
#### SRS-AC-P0-02 🟡: [Happy] Context Navigator correct (R-001 added in v2)
- **Source SRS:** [SRS §6.2.1 line 103](../../../specs/srs/3.%20PART-3_CHAPTER_4_5_6_7/SRS-EPIC-3-6_ACCEPTANCE%20CRITERIES.md)
- **FRs:** FR-CTX-001..003 (P1 in §4 — 🟡 P0 per D-008), FR-SYS-002, plus indirect dependence on [SRS-FR-MARM-2.006](#srs-fr-marm-2006), [SRS-FR-MARM-2.007](#srs-fr-marm-2007) (Context Broker source)
- **Verification method:** Unit + integration
- **Spec Evidence (verbatim):** "Anchor selection matches intent; canonical context snapshot hash identical for repeated runs with identical inputs"
- **Decomposed observable behaviors:**
  - **Clause A — Anchor selection matches intent:** for a given query, the selected anchor set is the highest-scoring one under the documented scoring function.
  - **Clause B — Canonical context snapshot hash determinism:** repeated runs with identical inputs produce an identical hash over the ordered selected anchors + LoreBook + truncation policy.
- **Covered by BA-ACs:** Clause A → [BA-AC-MARM-007](#ba-ac-marm-007), [BA-AC-MARM-008](#ba-ac-marm-008); Clause B → [BA-AC-MARM-023](#ba-ac-marm-023) (R-003 fix — was missing in v1).
- **Note:** Detailed Context Navigator FR specs (FR-CTX-001 §3.6.6 query-aware summarization, FR-CTX-003 §3.6.8 query-aware selection, FR-CTX-004 §3.6.9 token budget profiles) are owned by [brief-system-guarantees.md](brief-system-guarantees.md). This brief covers MARM's input contract to those FRs.

<a id="srs-ac-p0-03"></a>
#### SRS-AC-P0-03 🟢: [Happy] Session + User memory layers (R-001 added in v2)
- **Source SRS:** [SRS §6.2.1 line 104](../../../specs/srs/3.%20PART-3_CHAPTER_4_5_6_7/SRS-EPIC-3-6_ACCEPTANCE%20CRITERIES.md)
- **FRs:** [SRS-FR-MARM-2.009](#srs-fr-marm-2009), [SRS-FR-MARM-2.010](#srs-fr-marm-2010)
- **Verification method:** Manual QA
- **Spec Evidence (verbatim):** "L1 (session) and L2 (user) scopes persist correctly"
- **Decomposed observable behaviors:**
  - **Clause A — L1 session-scoped persistence:** SESSION-scoped anchors are visible only within that session and cleared/expired at session end.
  - **Clause B — L2 user-scoped persistence:** USER-scoped anchors persist across sessions and are visible in subsequent sessions of the same user.
- **Covered by BA-ACs:** Clause A → [BA-AC-MARM-010](#ba-ac-marm-010); Clause B → [BA-AC-MARM-011](#ba-ac-marm-011).

### Added by BA (derived from FRs, mockup, and domain model)

> Source of truth: this brief. These ACs cover behavior visible in mockup or required by business decisions but not yet captured in SRS.

<a id="ba-ac-marm-001"></a>
#### BA-AC-MARM-001 🟡: [Happy] Anchor lifecycle state transitions work
- **FR:** [SRS-FR-MARM-2.001](#srs-fr-marm-2001)
- **Source:** D-001 state names (see marker above); D-002 DORMANT reachability
- **Given** an anchor of type CONSTRAINT or EMOTION in ACTIVE state (types for which DORMANT is reachable under P0 parameters per D-002)
- **When** it is not accessed for the configured `non_use_threshold` (4h CONSTRAINT, 2h EMOTION)
- **Then** its state transitions to DECAYING then DORMANT, with each transition logged to Cognitive Trace. For GOAL/TASK/DECISION/FACT, anchors transition ACTIVE → DECAYING → EXPIRED (DORMANT deferred to P1 per D-002)

<a id="ba-ac-marm-002"></a>
#### BA-AC-MARM-002 🟡: [Happy] Dormant anchor reactivates on access
- **FR:** [SRS-FR-MARM-2.001](#srs-fr-marm-2001)
- **Source:** D-002; **Q1 (open)** — access definition
- **Given** an anchor in DORMANT state (CONSTRAINT or EMOTION, per D-002)
- **When** it is retrieved during context assembly (per Q1 narrow interpretation — included in LLM prompt)
- **Then** its state transitions back to ACTIVE, `last_accessed_at` is updated, and its weight is recalculated
- **🟡 Working interpretation (pending Q1):** "Access" means "anchor included in context assembly for LLM prompt." UI views, API GETs, and search-listing do NOT count. This preserves the usability of the "restore from DORMANT" user action. Dev team proceeds on this interpretation until product team responds.

<a id="ba-ac-marm-003"></a>
#### BA-AC-MARM-003 🟡: [Happy] TTL-based expiration works
- **FR:** [SRS-FR-MARM-2.002](#srs-fr-marm-2002)
- **Source:** D-006 per-type TTL from YAML
- **Given** an anchor with `expires_at` set to its type-specific `default_ttl` from YAML `lk_anchor_type`
- **When** the current time exceeds `expires_at`
- **Then** the Memory Manager transitions the anchor to EXPIRED state and it is excluded from default retrieval

<a id="ba-ac-marm-004"></a>
#### BA-AC-MARM-004 🟡: [Happy] Decay function reduces weight over time
- **FR:** [SRS-FR-MARM-2.003](#srs-fr-marm-2003)
- **Source:** D-003 DECISION parameter update (affects DECISION specifically)
- **Given** a FACT anchor with `initial_weight=0.65` and `half_life=48h`
- **When** 48 hours pass without access
- **Then** the anchor's weight is approximately 0.325 (50% of initial), and it transitions to DECAYING when `weight < initial_weight × 0.7`. For DECISION anchors specifically, `min_weight=0.20` (lowered from 0.40 per D-003) ensures observable DECAYING window of ~144h before expiry.

<a id="ba-ac-marm-005"></a>
#### BA-AC-MARM-005 🟡: [Happy] Anchor auto-promotes after meeting threshold
- **FR:** [SRS-FR-MARM-2.004](#srs-fr-marm-2004)
- **Source:** D-012, D-015 affect the type set
- **Given** a VERIFIED anchor of promotable type (DECISION, FACT, GOAL conditional on pinning, IDENTITY, PREFERENCE — per D-015) with `access_count >= promotion_threshold` (default 3)
- **When** the Memory Manager evaluates promotion eligibility
- **Then** the anchor is queued for LoreBook promotion review. Upon approval, verification_status becomes PROMOTED and a corresponding LoreBook entry (or mirror) is created. TASK/CONSTRAINT/EMOTION/mirror types are non-promotable per D-012 and §3.5.6.1 eligible list

<a id="ba-ac-marm-006"></a>
#### BA-AC-MARM-006 🟡: [Happy] Anchor demotes after non-use
- **FR:** [SRS-FR-MARM-2.005](#srs-fr-marm-2005)
- **Source:** D-002, D-004
- **Given** a DECAYING anchor of type CONSTRAINT or EMOTION (the two types reaching DORMANT per D-002), with `last_accessed_at` exceeding its per-type `non_use_threshold`
- **When** the Memory Manager runs its GC cycle
- **Then** the anchor transitions to DORMANT state and is excluded from default retrieval

<a id="ba-ac-marm-007"></a>
#### BA-AC-MARM-007: [Happy] Context Broker assembles relevant anchors
- **FR:** [SRS-FR-MARM-2.006](#srs-fr-marm-2006)
- **Given** a user query in an active dialog with 20 ACTIVE anchors across SESSION and USER scopes
- **When** the Context Broker assembles the prompt
- **Then** anchors are selected by: (1) scope precedence (SESSION first), (2) relevance to query, (3) weight, (4) freshness; and only ACTIVE/DECAYING anchors with VERIFIED/PROMOTED status are included by default

<a id="ba-ac-marm-008"></a>
#### BA-AC-MARM-008: [Boundary] Max anchors in prompt is enforced
- **FR:** [SRS-FR-MARM-2.007](#srs-fr-marm-2007)
- **Given** 30 eligible ACTIVE anchors exist for a user query
- **When** the Context Broker assembles the prompt with `max_anchors_in_prompt=10`
- **Then** exactly 10 (or fewer) anchors are included, selected by scoring algorithm, and the total token budget is respected

<a id="ba-ac-marm-009"></a>
#### BA-AC-MARM-009: [Happy] Audit trail records all anchor operations
- **FR:** [SRS-FR-MARM-2.008](#srs-fr-marm-2008)
- **Given** a user creates, updates, verifies, and deletes an anchor
- **When** each operation completes
- **Then** a Cognitive Trace event is emitted for each operation containing: `anchor_id`, `operation_type`, `actor`, `timestamp`, and before/after states

<a id="ba-ac-marm-010"></a>
#### BA-AC-MARM-010: [Happy] L1 Session Memory is isolated and volatile
- **FR:** [SRS-FR-MARM-2.009](#srs-fr-marm-2009)
- **Given** a user creates anchors with `scope=SESSION` during a dialog session
- **When** the session ends (timeout or explicit end)
- **Then** SESSION-scoped anchors are either expired or purged (per policy), and they were only visible within that specific session during its lifetime

<a id="ba-ac-marm-011"></a>
#### BA-AC-MARM-011: [Happy] L2 User Memory persists across sessions
- **FR:** [SRS-FR-MARM-2.010](#srs-fr-marm-2010)
- **Given** a user has USER-scoped anchors created in previous sessions
- **When** they start a new dialog session
- **Then** their USER-scoped ACTIVE/DECAYING anchors with VERIFIED status are available for context assembly in the new session

<a id="ba-ac-marm-012"></a>
#### BA-AC-MARM-012: [Happy] Cleanup scheduler executes on schedule
- **FR:** [SRS-FR-MARM-2.011](#srs-fr-marm-2011)
- **Given** the `cleanup_schedule` is set to "daily"
- **When** the scheduled time arrives
- **Then** the Memory Manager: (1) transitions EXPIRED anchors past retention to purge, (2) transitions DELETED anchors past retention to purge, (3) enforces quota limits, (4) logs cleanup results to trace

<a id="ba-ac-marm-013"></a>
#### BA-AC-MARM-013 🟡: [Happy] User creates anchor manually via UI
- **FR:** [SRS-FR-MARM-2.010](#srs-fr-marm-2010)
- **Source:** D-015 expanded type set
- **Given** a logged-in user on the Memory page
- **When** they click "Add Entry", select from the 8 user-creatable independent types (GOAL, TASK, CONSTRAINT, DECISION, FACT, EMOTION, IDENTITY, PREFERENCE), fill in content and scope, and click "Remember this"
- **Then** a new anchor is created with `state=ACTIVE`, `verification_status=VERIFIED`, `source=MANUAL`, the configured confidence and per-type TTL, and it appears in the anchor list

<a id="ba-ac-marm-014"></a>
#### BA-AC-MARM-014: [Happy] User verifies extracted anchor
- **FR:** [SRS-FR-MARM-2.001](#srs-fr-marm-2001)
- **Given** an auto-extracted anchor with `verification_status=PENDING`
- **When** the user clicks "Verify" in the Edit Anchor modal
- **Then** `verification_status` changes to VERIFIED, the anchor becomes eligible for default retrieval, and a trace event is emitted

<a id="ba-ac-marm-015"></a>
#### BA-AC-MARM-015: [Happy] User rejects extracted anchor
- **FR:** [SRS-FR-MARM-2.001](#srs-fr-marm-2001)
- **Given** an auto-extracted anchor with `verification_status=PENDING`
- **When** the user clicks "Reject" and confirms
- **Then** `verification_status` changes to REJECTED, the anchor is excluded from retrieval, and a Restore option becomes available

<a id="ba-ac-marm-016"></a>
#### BA-AC-MARM-016: [Permission] Anchors are private by default
- **FR:** [SRS-FR-MARM-2.010](#srs-fr-marm-2010)
- **Given** user A creates a USER-scoped anchor
- **When** user B in the same workspace queries for anchors
- **Then** user B cannot see or retrieve user A's anchor (`visibility=PRIVATE` enforced)

<a id="ba-ac-marm-017"></a>
#### BA-AC-MARM-017: [Permission] Memory Settings requires admin role
- **FR:** [SRS-FR-MARM-2.002](#srs-fr-marm-2002)
- **Given** a user with "member" role (not admin)
- **When** they attempt to update Memory Settings via API
- **Then** the request is rejected with 403 Forbidden

<a id="ba-ac-marm-018"></a>
#### BA-AC-MARM-018: [Boundary] Memory budget enforcement under pressure
- **FR:** [SRS-FR-MARM-2.001](#srs-fr-marm-2001)
- **Given** a user's anchor count is at 95% of their User scope limit (1000)
- **When** a new anchor extraction is triggered
- **Then** extraction is paused for that scope and only user-confirmed anchors are processed, with a system alert emitted

<a id="ba-ac-marm-019"></a>
#### BA-AC-MARM-019: [Error:Validation] Anchor content exceeds max length
- **FR:** [SRS-FR-MARM-2.010](#srs-fr-marm-2010)
- **Given** a user creates an anchor via the New Anchor modal
- **When** the content exceeds 500 characters
- **Then** validation prevents submission with an appropriate error message

<a id="ba-ac-marm-020"></a>
#### BA-AC-MARM-020: [Concurrency] Conflict detection between anchors
- **FR:** [SRS-FR-MARM-2.006](#srs-fr-marm-2006)
- **Given** two anchors with high semantic similarity (>0.85) but different values in the same scope and type
- **When** both are candidates for context assembly
- **Then** the conflict resolution algorithm selects: higher confidence wins; if tied, more recent wins; and the loser's decay rate is increased by 2x

<a id="ba-ac-marm-021"></a>
#### BA-AC-MARM-021 🟡: [Recovery] Mirror anchor invalidation on LoreBook update
- **FR:** [SRS-FR-MARM-2.010](#srs-fr-marm-2010)
- **Source:** D-011 (RULE_MIRROR cascade implemented by pattern)
- **Given** an IDENTITY_MIRROR, PREFERENCE_MIRROR, FACT_MIRROR, or **RULE_MIRROR** (per D-011, implemented by pattern) anchor linked to a LoreBook entry via `canonical_ref`
- **When** the source LoreBook entry is updated
- **Then** the mirror anchor's state transitions to STALE, awaiting re-sync with the updated LoreBook content

<a id="ba-ac-marm-022"></a>
#### BA-AC-MARM-022 🟡: [Happy] User-initiated soft delete on EXPIRED anchor (R-004 added in v2)
- **FR:** [SRS-FR-MARM-2.001](#srs-fr-marm-2001)
- **Source:** D-014 (EXPIRED → DELETED transition is permitted only on user action); GUI Spec row 41 (Delete Button visibility includes EXPIRED)
- **Given** an anchor in EXPIRED state owned by the current user
- **When** the user opens the Edit Anchor modal, clicks "Delete", and confirms in ConfirmDeleteModal
- **Then** the anchor's state transitions EXPIRED → DELETED, `deleted_at` is set to now, the anchor is excluded from all retrieval, a trace event of type `anchor.user_deleted` is emitted, and the anchor is queued for purge after the retention period (default 365d)

<a id="ba-ac-marm-023"></a>
#### BA-AC-MARM-023 🟢: [Happy] Canonical context snapshot hash determinism (R-003 added in v2)
- **FR:** [SRS-FR-MARM-2.006](#srs-fr-marm-2006), [SRS-FR-MARM-2.007](#srs-fr-marm-2007)
- **Source:** SRS-AC-P0-02 Clause B verbatim ("canonical context snapshot hash identical for repeated runs with identical inputs"); BR-16
- **Given** the same dialog state, anchor pool, snapshot_time, query text, and DP-Risk inputs
- **When** the Context Broker assembles a canonical context snapshot twice (run 1 and run 2)
- **Then** both runs select the same ordered anchor set (by ID) and produce an identical canonical_snapshot_hash. The hash is computed over: ordered selected anchor IDs + ordered LoreBook entry IDs + applied truncation policy code + max_anchors_in_prompt value
- **Note:** Conflict-resolution determinism BR-12 (selection_seed for tied confidence/timestamp) is the underlying mechanism; this AC verifies the end-to-end stability.

## Definition of Done

- [ ] All 23 BA-AC-MARM-* acceptance criteria pass in staging environment → [BA-AC-MARM-001](#ba-ac-marm-001) … [BA-AC-MARM-023](#ba-ac-marm-023)
- [ ] SRS-AC-P0-01 integration test suite passes (all lifecycle states functional; DORMANT tested via CONSTRAINT/EMOTION per D-002) → [SRS-AC-P0-01](#srs-ac-p0-01)
- [ ] SRS-AC-P0-02 hash determinism verified (Clause B) → [BA-AC-MARM-023](#ba-ac-marm-023)
- [ ] SRS-AC-P0-03 L1/L2 persistence verified → [BA-AC-MARM-010](#ba-ac-marm-010), [BA-AC-MARM-011](#ba-ac-marm-011)
- [ ] QA has executed all smoke and regression test cases with 0 Critical/Major defects open → [tc-marm-memory.md](../test-cases/tc-marm-memory.md)
- [ ] Anchor CRUD operations respond in < 500ms at p95 → [SC-04](#applicable-success-criteria)
- [ ] Memory Manager GC cycle completes within 30s for 10,000 anchors → [SC-04](#applicable-success-criteria)
- [ ] Context assembly with anchor selection contributes within response latency target (P95 < 3s total) → [SC-04](#applicable-success-criteria)
- [ ] Sidebar filters correctly reflect faceted counts after any anchor state change → [BA-AC-MARM-001](#ba-ac-marm-001)
- [ ] New Anchor Modal supports 8 independent types (incl. IDENTITY and PREFERENCE per D-015) → [BA-AC-MARM-013](#ba-ac-marm-013)
- [ ] Settings modal surfaces per-type YAML values with global fallback (per D-004, D-006) → [BA-AC-MARM-003](#ba-ac-marm-003), [BA-AC-MARM-006](#ba-ac-marm-006)
- [ ] EXPIRED anchors visible in sidebar filter and deletable via Edit modal (per D-014) → [BA-AC-MARM-022](#ba-ac-marm-022)
- [ ] Import/Export produces valid JSON roundtrip → [BA-AC-MARM-013](#ba-ac-marm-013)
- [ ] Clear All requires confirmation and correctly soft-deletes all anchors → [BA-AC-MARM-022](#ba-ac-marm-022)
- [ ] Anchor encryption at rest verified by infra audit → POL-ENC-001 attestation (BR-17)
- [ ] Anchor audit-event retention verified by policy attestation → POL-RET-001 (BR-18, [BA-AC-MARM-009](#ba-ac-marm-009))

### Applicable Success Criteria ([SRS §1.5](../../../specs/srs/1.%20PART-1_CHAPTER_1_2/SRS-EPIC-3-1_INTRODUCTION.md))

| SC-ID | Criterion | Target | Verification | How This Module Contributes | Stability |
|-------|-----------|--------|-------------|---------------------------|:-:|
| SC-01 | Memory lifecycle | All states functional | Integration test | This IS the module — all state transitions (ACTIVE/DECAYING/DORMANT/EXPIRED/DELETED) must work; DORMANT tested via CONSTRAINT/EMOTION per D-002 | 🟡 (D-001, D-002) |
| SC-02 | Context assembly | Anchor selection correct + canonical hash deterministic | Unit + integration | Provides the anchors that Context Broker selects from; scope precedence and scoring must be correct; canonical snapshot hash MUST be identical for repeated runs with identical inputs (BA-AC-MARM-023) | 🟢 |
| SC-04 | Response latency | P95 < 3s (local inference) | Performance test | Anchor retrieval + scoring contributes to the total query-to-response latency; Memory Manager GC must not block hot path | 🟢 |
| SC-06 | System stability | No crashes 24h | Endurance test | Memory Manager GC cycle and cleanup scheduler under sustained load must not leak or crash | 🟢 |
| SC-08 🟡 | Circuit closure (light) | Query → Trace with Memory Loop | Demo | Dialog message → anchor extraction → context assembly → response → basic trace metadata. Full immutable Cognitive Trace remains P1. Per D-009. | 🟡 (D-009) |

## Risks & Assumptions

### Assumptions
| # | Assumption | Impact if Wrong | Status |
|---|-----------|----------------|--------|
| 1 | State names `ACTIVE/DECAYING/DORMANT/EXPIRED/DELETED` (+ STALE) are canonical per D-001 | If product team reverses to FR-MARM-2.001 original terminology, domain model, data model, and SRS §3.3.5 need rework | 🟡 Interim (D-001) |
| 2 | P0 promotion threshold (3 occurrences) is a simplified subset of full §3.5.6 criteria; complete criteria are P1 | If full criteria are P0, significantly more complex promotion engine needed for M1 | Confirmed (FR-MARM-2.004 text) |
| 3 | Anchor Extractor depends on model inference (FR-MDL-001) for typed extraction + deterministic pattern mapping for IDENTITY per D-013 | If model unavailable in M1, IDENTITY extraction still works via pattern; other types defer to manual creation fallback | Confirmed (SRS §3.3.4, D-013) |
| 4 | Memory Settings are workspace-scoped (all members inherit admin settings) | If per-user settings needed, config resolution logic is more complex | Confirmed (mock API) |
| 5 | 12-type taxonomy per D-015 is accepted scope for P0 | If product team rejects IDENTITY/PREFERENCE independents, Anchor Extractor simplified to 6 types | 🟡 Interim (D-015) |
| 6 | Canonical context snapshot hash includes ordered selected anchor IDs + LoreBook IDs + truncation policy + max_anchors_in_prompt — sufficient input set for SRS-AC-P0-02 Clause B | If hash must include additional inputs (e.g., model version, system prompt revision), hash function expands and BA-AC-MARM-023 needs revision | 🟢 (BA-derived; sources minimal) |

### Risks
| # | Risk | Likelihood | Impact | Mitigation |
|---|------|-----------|--------|------------|
| 1 | Product team picks different option on a P0 item (D-001..D-005) | Medium | High — triggers code rework in state handling, parameters, or taxonomy | Keep interim implementation reversible; flag code paths with `// TODO(D-NNN)` comments |
| 2 | Anchor Extractor depends on FR-MDL-001 (M2) but MARM is M1 | High | Medium — LLM-based auto-extraction not available until model registry ready | Implement manual anchor creation first; IDENTITY pattern-mapping works standalone; LLM extraction behind feature flag |
| 3 | Memory budget enforcement and eviction algorithm complexity | Medium | Medium — edge cases in concurrent extraction + eviction | Implement simple eviction first (weight-based), add full scoring formula in iteration |
| 4 | Semantic conflict detection requires vector similarity computation | Medium | High — needs vector similarity infrastructure (Weaviate per current plan) | Defer full semantic conflict detection to P1 if vector similarity infrastructure not ready in M1 |
| 5 | Large anchor counts (1000+ per user) may impact retrieval latency | Low | Medium — could push response latency beyond SC-04 target | Index optimization + cache layer (Redis) for hot anchors |
| 6 | Q1 (access definition) response is "broad" interpretation, breaking the "restore from DORMANT" UX | Medium | Medium — UI flow redesign for restore action | Working interpretation (narrow) preserves UX; switch to broad if product team requires + redesign restore |
| 7 | Hash determinism (BA-AC-MARM-023) breaks under DB row-ordering, JSON key-ordering, or float precision drift | Medium | High — silently breaks SRS-AC-P0-02 Clause B | Specify total-order over IDs, use stable JSON serialization (sorted keys), document hash algorithm in dev brief |

## Open Questions

The team internal decision process has resolved 15 items as interim (see [decision-log.md](../decision-log.md) rows D-001..D-015) and sent them to product team in [Anchor Lifecycle — Specification Gaps and Proposed Solutions](../product-team-proposals/anchor-lifecycle-spec-gaps-and-proposed-solutions-2026-04-20.md). Product team is reviewing at their own pace; internal escalation target is **2026-05-27** (carried forward from 2026-05-20 because no DTX response was logged by 2026-04-29).

**One remaining open question with working interpretation:**

### Q1 — Definition of "access" for decay clock reset and reactivation
- **Directed to:** product team (see [product-team-proposals/anchor-lifecycle-spec-gaps-and-proposed-solutions-2026-04-20.md](../product-team-proposals/anchor-lifecycle-spec-gaps-and-proposed-solutions-2026-04-20.md) Part B)
- **Working interpretation (used in BA-AC-MARM-002):** narrow — only "included in context assembly for the LLM prompt" counts. UI views, API GETs, and search-listing do NOT count. Preserves the usability of the restore-from-DORMANT flow.
- **Blocking?** No — dev can proceed on working interpretation. Switching to broad interpretation would require Memory UI redesign for the restore action.

## Change History
| Date | What Changed | Source | Impact |
|------|-------------|--------|--------|
| 2026-04-10 | Initial draft | SRS §3.3, §3.5, §4.3.3, domain model, data model, frontend2, mock API | First version — 11 FRs, 21 ACs, 59-row GUI Spec Table |
| 2026-04-20 | Major revision (v1) | devlead picks on [anchor-lifecycle-reference.md §12](../yaml-summaries/anchor-lifecycle-reference.md) → D-001..D-015 interim decisions | Applied Stability Legend + 🟡 markers; renamed IDs (SRS-FR-MARM-*, BA-AC-MARM-*); converted GitHub URLs to repo-relative paths; expanded taxonomy to 12 types; updated DORMANT reachability scope; reclassified FR-CTX-001/003/004 as P0 (D-008); added Q1 working interpretation; consolidated 5 prior Open Questions into decision-log tracking |
| 2026-04-29 | v2 sandbox revision | Re-validation pass (no SRS/domain/mock-API/mockup changes since v1); R-001..R-004, R-006, R-007 from [decision-log.md § Pending Revisions](../decision-log.md) | Added SRS-AC-P0-02, SRS-AC-P0-03 to "From SRS"; corrected SRS-AC-P0-01 verbatim quote per source line 102 (still pre-D-001 wording — flagged); added BA-AC-MARM-022 (user-initiated EXPIRED→DELETED per D-014); added BA-AC-MARM-023 (canonical hash determinism for AC-P0-02 Clause B); fixed Mock API path `routes/*.ts` → `routers/*.py`; added Pass 0.5 BR category tags (15 BRs → BR-1..BR-18 incl. new BR-16/17/18 [CROSS-CUTTING]/[INFRA]/[POLICY]); added BR Coverage table; added Numeric Reconciliation summary; carried-forward all 15 D-NNN with confirm-by 2026-05-27; added DoD trace pointers to BA-ACs/SCs/POL-*; added Risk #7 hash determinism. **Note:** Version line stays "v1 (sandbox)" per agent versioning rule — only AYITA_PROD publication bumps the counter. R-005 (dialog brief) out of scope; not addressed here. |
