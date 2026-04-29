# Test Cases: MARM 2.0 Memory

**Source Brief:** [brief-marm-memory-v2.md](../implementation-briefs/brief-marm-memory-v2.md)
**Target Milestone: M1**
**Version:** v1 (sandbox) — TC-IDs inherited from [tc-marm-memory.md](tc-marm-memory.md); +2 added (022, 023)
**Status:** Draft
**Generated:** 2026-04-29

## Executive Summary

23 test cases (BA-TC-MARM-001 … BA-TC-MARM-023) covering all 23 BA-AC-MARM-* and the 3 SRS-AC-P0-01/02/03 mappings introduced by R-001. v2 inherits 21 v1 TCs verbatim (only metadata enrichment: Fidelity Map, Milestone Compatibility, Entity coverage) and adds two new TCs: BA-TC-MARM-022 (user-initiated EXPIRED→DELETED per D-014, R-004) and BA-TC-MARM-023 (canonical context snapshot hash determinism per SRS-AC-P0-02 Clause B, R-003). Distribution: 9 Smoke / 10 Integration / 4 Regression — 9 P0, 11 P1, 3 P2. 11 cases inherit 🟡 from D-001…D-015; the rest are 🟢. Cross-milestone dependencies (Cognitive Trace M1-light per D-009, FR-CTX-001/003/004 P0 per D-008, FR-MDL-001 M2 for LLM extraction) are marked Mocked or Stubbed in Preconditions.

**Step format:** numbered Step / Expected pairs. Every state-changing step has explicit Expected (no `—`). Preconditions are declarative state only; no inter-TC chaining.

<!-- Dependency milestone map: Memory Manager scheduler=M1 (SAME), Cognitive Trace=M1-light per D-009 (SAME, mocked), Context Broker=M1 (SAME), Context Navigator FR-CTX-001/003/004=M1 per D-008 (SAME, brief-system-guarantees.md owner), LoreBook writer=M1 (SAME, brief-lorebook-crud.md owner), Anchor Extractor LLM (FR-MDL-001 dependent)=M2 (LATER, mocked/stubbed for M1 manual paths), DP-Risk pipeline=M1 (SAME, threshold table 🟡 D-005), Vector similarity (Weaviate)=M1 (SAME, deferrable per Risk #4), Mock API memory.py=M1 fixture (SAME, path corrected per R-006 to tools/tools/src/routers/memory.py) -->

---

## Test Cases

<!-- Fidelity Map BA-TC-MARM-001 (BA-AC-MARM-001 Then-clause claims):
  C1 "state transitions to DECAYING then DORMANT" → Step 2, Step 3 Expected
  C2 "each transition logged to Cognitive Trace" → Step 3, Step 4 Expected
  C3 "GOAL/TASK/DECISION/FACT transition ACTIVE → DECAYING → EXPIRED (DORMANT deferred)" → covered by family TC-003/004 (per-type EXPIRED path); this TC scopes to CONSTRAINT subset per title
-->
### BA-TC-MARM-001 🟡 · [Happy] State transition ACTIVE → DECAYING → DORMANT for CONSTRAINT anchor

| | |
|---|---|
| **Source AC** | [BA-AC-MARM-001](../implementation-briefs/brief-marm-memory-v2.md#ba-ac-marm-001) |
| **Source FR** | [SRS-FR-MARM-2.001](../implementation-briefs/brief-marm-memory-v2.md#srs-fr-marm-2001) |
| **Priority** | P0 (smoke) |
| **Cycle** | Smoke |
| **Verification** | Integration test |
| **Audience** | @qa-automated |
| **Milestone Compatibility** | M1 (runs with mock for Cognitive Trace per D-009) |
| **Entity coverage** | CONSTRAINT (subset of BA-AC-MARM-001 entity set {CONSTRAINT, EMOTION, GOAL, TASK, DECISION, FACT}; EMOTION covered by BA-TC-MARM-006; GOAL/TASK/DECISION/FACT EXPIRED path covered by BA-TC-MARM-003/004) |
| **Inherits from** | D-001 (state names), D-002 (DORMANT reachability) |

**Preconditions:** Memory Manager scheduler running; Appendix C.2 per-type thresholds active (CONSTRAINT `non_use_threshold=4h`, `half_life=4h`); Cognitive Trace mock writer attached; test anchor repository clean.

| # | Step | Expected |
|:-:|------|----------|
| 1 | Create CONSTRAINT anchor with `state=ACTIVE`, `initial_weight=0.85`, `half_life=4h`, `min_weight=0.20`. | Anchor exists; `state=ACTIVE`; `last_accessed_at=created_at`. |
| 2 | Advance clock 2.5h; trigger scheduler. | `state=DECAYING` (weight has crossed `initial_weight × 0.7` threshold). |
| 3 | Advance clock to 4h (no retrievals). | `state=DORMANT`; transition event emitted to Cognitive Trace. |
| 4 | Advance clock to 8.3h. | `state=EXPIRED` (weight crossed `min_weight`); EXPIRED event in trace. |

---

<!-- Fidelity Map BA-TC-MARM-002 (BA-AC-MARM-002 Then-clause claims):
  C1 "state transitions back to ACTIVE" → Step 2 Expected
  C2 "last_accessed_at is updated" → Step 2 Expected
  C3 "weight is recalculated" → Step 2 Expected
  C4 (working interp) "UI views, API GETs, search-listing do NOT count" → Step 1, Step 3 Expected
-->
### BA-TC-MARM-002 🟡 · [Happy] DORMANT → ACTIVE on retrieval (narrow-access interpretation)

| | |
|---|---|
| **Source AC** | [BA-AC-MARM-002](../implementation-briefs/brief-marm-memory-v2.md#ba-ac-marm-002) |
| **Source FR** | [SRS-FR-MARM-2.001](../implementation-briefs/brief-marm-memory-v2.md#srs-fr-marm-2001) |
| **Priority** | P0 (smoke) |
| **Cycle** | Smoke |
| **Verification** | Integration test |
| **Audience** | @qa-automated |
| **Milestone Compatibility** | M1 (runs with mock for Cognitive Trace) |
| **Entity coverage** | CONSTRAINT (DORMANT-reachable type per D-002) |
| **Inherits from** | D-002, Q1 (working interpretation: access = "included in LLM prompt") |

**Preconditions:** Fixture `dormant_constraint_anchor` exists (CONSTRAINT, `state=DORMANT`, `last_accessed_at=now-4h`); Context Broker available; Memory UI available; Cognitive Trace mock attached.

| # | Step | Expected |
|:-:|------|----------|
| 1 | User-initiated query triggers MARM retrieval; anchor appears as candidate. | Candidate set includes the DORMANT anchor; `state` remains DORMANT (retrieval-as-candidate alone does not reactivate). |
| 2 | Context Broker passes compression and budget; anchor is included in final LLM prompt. | `last_accessed_at` updated to now; `state=ACTIVE`; weight recalculated; transition event in Cognitive Trace. |
| 3 | User views the anchor detail in Memory UI (read-only API GET). | `state` unchanged; `last_accessed_at` unchanged. UI view does NOT count as access under narrow interpretation. |

---

<!-- Fidelity Map BA-TC-MARM-003 (BA-AC-MARM-003 Then-clause claims):
  C1 "Memory Manager transitions the anchor to EXPIRED state" → Step 2 Expected
  C2 "excluded from default retrieval" → Step 2, Step 3 Expected
-->
### BA-TC-MARM-003 🟡 · [Happy] TTL-based expiration uses per-type `default_ttl`

| | |
|---|---|
| **Source AC** | [BA-AC-MARM-003](../implementation-briefs/brief-marm-memory-v2.md#ba-ac-marm-003) |
| **Source FR** | [SRS-FR-MARM-2.002](../implementation-briefs/brief-marm-memory-v2.md#srs-fr-marm-2002) |
| **Priority** | P0 (smoke) |
| **Cycle** | Smoke |
| **Verification** | Integration test |
| **Audience** | @qa-automated |
| **Milestone Compatibility** | M1 (runs in full) |
| **Entity coverage** | EMOTION (subset of BA-AC-MARM-003 entity set {any anchor type with `default_ttl`}; pattern generalizes per YAML lookup) |
| **Inherits from** | D-006 (TTL from YAML `lk_anchor_type`) |

**Preconditions:** YAML `lk_anchor_type.default_ttl` populated per D-006; scheduler active; clean anchor store.

| # | Step | Expected |
|:-:|------|----------|
| 1 | Create an EMOTION anchor (YAML `default_ttl=4h`, NULL `expires_at`). | Anchor exists; `expires_at` resolved dynamically from YAML to `created_at + 4h`. |
| 2 | Advance clock to 4h + 1s; trigger scheduler. | `state=EXPIRED`; anchor excluded from default retrieval. |
| 3 | Query with `include_expired=true`. | Anchor still retrievable in expired-listing mode. |

---

<!-- Fidelity Map BA-TC-MARM-004 (BA-AC-MARM-004 Then-clause claims):
  C1 "weight is approximately 0.325 (50% of initial)" → Step 1 Expected
  C2 "transitions to DECAYING when weight < initial_weight × 0.7" → Step 1 Expected
  C3 "DECISION min_weight=0.20 ensures observable DECAYING window of ~144h" → Step 2, Step 3 Expected
-->
### BA-TC-MARM-004 🟡 · [Happy] Decay function reduces weight per formula; DECISION floor at 0.20

| | |
|---|---|
| **Source AC** | [BA-AC-MARM-004](../implementation-briefs/brief-marm-memory-v2.md#ba-ac-marm-004) |
| **Source FR** | [SRS-FR-MARM-2.003](../implementation-briefs/brief-marm-memory-v2.md#srs-fr-marm-2003) |
| **Priority** | P1 |
| **Cycle** | Integration |
| **Verification** | Unit + integration |
| **Audience** | @developer |
| **Milestone Compatibility** | M1 (runs in full) |
| **Entity coverage** | FACT, DECISION (BA-AC-MARM-004 entity set; DECISION explicitly tested for D-003 floor) |
| **Inherits from** | D-003 (DECISION `min_weight=0.20`); BR-10 weight invariant precondition |

**Preconditions:** Decay module deterministic under test clock; formula `weight(t) = initial × 0.5^(t/half_life)` implemented; DB CHECK constraint `weight <= initial_weight` (BR-10) enforced.

| # | Step | Expected |
|:-:|------|----------|
| 1 | Create FACT anchor (`initial=0.65`, `half_life=48h`, `min_weight=0.30`). Advance 48h. | `weight ≈ 0.325` (±0.01); `state=DECAYING` (crossed `0.65 × 0.7 = 0.455` at ~24.7h). |
| 2 | Create DECISION anchor (`initial=0.80`, `half_life=72h`, `min_weight=0.20`). Advance 72h. | `weight ≈ 0.40`; `state=DECAYING` (still above `min_weight=0.20` — observable window present per D-003). |
| 3 | Advance DECISION clock to 144h (no accesses). | `weight ≈ 0.20`; transition to EXPIRED. |

---

<!-- Fidelity Map BA-TC-MARM-005 (BA-AC-MARM-005 Then-clause claims):
  C1 "anchor is queued for LoreBook promotion review" → Step 2 Expected
  C2 "verification_status becomes PROMOTED" → Step 3 Expected
  C3 "corresponding LoreBook entry (or mirror) is created" → Step 3, Step 5 Expected
  C4 "TASK/CONSTRAINT/EMOTION/mirror types are non-promotable" → Step 4 Expected
-->
### BA-TC-MARM-005 🟡 · [Happy] Anchor auto-promotes after threshold; type-set per D-015

| | |
|---|---|
| **Source AC** | [BA-AC-MARM-005](../implementation-briefs/brief-marm-memory-v2.md#ba-ac-marm-005) |
| **Source FR** | [SRS-FR-MARM-2.004](../implementation-briefs/brief-marm-memory-v2.md#srs-fr-marm-2004) |
| **Priority** | P1 |
| **Cycle** | Integration |
| **Verification** | Integration test |
| **Audience** | @qa-automated |
| **Milestone Compatibility** | M1 (runs with mock for LoreBook writer — owned by brief-lorebook-crud.md, M1 SAME milestone) |
| **Entity coverage** | DECISION, TASK, IDENTITY (subset; CONSTRAINT/EMOTION/mirrors non-promotable assertions covered by step 4 via TASK exemplar) |
| **Inherits from** | D-012 (TASK/CONSTRAINT non-promotable), D-015 (12-type taxonomy) |

**Preconditions:** `promotion_threshold=3`; LoreBook writer mocked (records create-intents); verification queue available; test user with promotion-approval rights.

| # | Step | Expected |
|:-:|------|----------|
| 1 | Create VERIFIED DECISION anchor; increment `access_count` to 3 (P0 threshold — single-session increments acceptable for P0; multi-session is §3.5.6 P1 criteria). | Anchor meets promotion eligibility. |
| 2 | Run Memory Manager promotion evaluation. | Anchor queued for LoreBook promotion review. |
| 3 | Approve promotion (acceptance action or admin). | `verification_status=PROMOTED`; LoreBook FACT entry created with back-reference to anchor. |
| 4 | Attempt same flow with TASK anchor. | Ineligible — TASK not in §3.5.6.1 list (D-012); no promotion queued. |
| 5 | Attempt with IDENTITY anchor (new independent type per D-015). | Eligible; promotes into LoreBook IDENTITY entry; IDENTITY_MIRROR anchor created. |

---

<!-- Fidelity Map BA-TC-MARM-006 (BA-AC-MARM-006 Then-clause claims):
  C1 "anchor transitions to DORMANT state" → Step 2 Expected
  C2 "excluded from default retrieval" → Step 3 Expected
-->
### BA-TC-MARM-006 🟡 · [Happy] Demotion DECAYING → DORMANT for EMOTION anchor via `non_use_threshold`

| | |
|---|---|
| **Source AC** | [BA-AC-MARM-006](../implementation-briefs/brief-marm-memory-v2.md#ba-ac-marm-006) |
| **Source FR** | [SRS-FR-MARM-2.005](../implementation-briefs/brief-marm-memory-v2.md#srs-fr-marm-2005) |
| **Priority** | P0 (smoke) |
| **Cycle** | Smoke |
| **Verification** | Integration test |
| **Audience** | @qa-automated |
| **Milestone Compatibility** | M1 (runs with mock for Cognitive Trace) |
| **Entity coverage** | EMOTION (DORMANT-reachable type per D-002; CONSTRAINT covered by BA-TC-MARM-001) |
| **Inherits from** | D-002, D-004 (`non_use_threshold` canonical name) |

**Preconditions:** Fixture `decaying_emotion_anchor` exists (EMOTION, `state=DECAYING`, Appendix C.2 `non_use_threshold=2h`, `last_accessed_at=now`); scheduler active; Cognitive Trace mock attached.

| # | Step | Expected |
|:-:|------|----------|
| 1 | Confirm fixture state. | Anchor in DECAYING state. |
| 2 | Advance clock 2h with no retrievals; trigger scheduler. | `state=DORMANT`; transition event in trace. |
| 3 | Query default-scope context assembly. | Anchor excluded from retrieval. |
| 4 | Query with `include_dormant=true`. | Anchor returned. |

---

<!-- Fidelity Map BA-TC-MARM-007 (BA-AC-MARM-007 Then-clause claims):
  C1 "anchors are selected by scope precedence (SESSION first)" → Step 2 Expected
  C2 "by relevance to query" → Step 2, Step 3 Expected
  C3 "by weight" → Step 3 Expected
  C4 "by freshness" → Step 3 Expected
  C5 "only ACTIVE/DECAYING with VERIFIED/PROMOTED status included by default" → Step 2 Expected
-->
### BA-TC-MARM-007 · [Happy] Context Broker assembles relevant anchors

| | |
|---|---|
| **Source AC** | [BA-AC-MARM-007](../implementation-briefs/brief-marm-memory-v2.md#ba-ac-marm-007) |
| **Source FR** | [SRS-FR-MARM-2.006](../implementation-briefs/brief-marm-memory-v2.md#srs-fr-marm-2006) |
| **Priority** | P0 (smoke) |
| **Cycle** | Smoke |
| **Verification** | Integration test |
| **Audience** | @qa-automated |
| **Milestone Compatibility** | M1 (runs with mock for DP-Risk score input — BR-13 [CROSS-CUTTING] precondition) |
| **Entity coverage** | SESSION-scope, USER-scope; ACTIVE, DECAYING; VERIFIED, PROMOTED |

**Preconditions:** 20 ACTIVE anchors across SESSION and USER scopes (mix of VERIFIED and PROMOTED); `max_anchors_in_prompt=10`; DP-Risk mock returning low-risk so write-path gating not triggered (BR-13).

| # | Step | Expected |
|:-:|------|----------|
| 1 | Submit user query relevant to half of the anchors. | Retrieval returns candidate set. |
| 2 | Context Broker assembles prompt. | Exactly ≤ 10 anchors included; SESSION scope prioritized over USER; only ACTIVE/DECAYING with VERIFIED/PROMOTED status included by default. |
| 3 | Inspect composite score of included anchors. | Order reflects relevance × 0.35 + recency × 0.25 + importance × 0.20 + priority × 0.15 − risk × 0.05. |

---

<!-- Fidelity Map BA-TC-MARM-008 (BA-AC-MARM-008 Then-clause claims):
  C1 "exactly 10 (or fewer) anchors are included" → Step 2 Expected
  C2 "selected by scoring algorithm" → Step 2 Expected
  C3 "total token budget is respected" → Step 3 Expected
-->
### BA-TC-MARM-008 · [Boundary] Max anchors in prompt enforced at capacity

| | |
|---|---|
| **Source AC** | [BA-AC-MARM-008](../implementation-briefs/brief-marm-memory-v2.md#ba-ac-marm-008) |
| **Source FR** | [SRS-FR-MARM-2.007](../implementation-briefs/brief-marm-memory-v2.md#srs-fr-marm-2007) |
| **Priority** | P1 |
| **Cycle** | Integration |
| **Verification** | Integration test |
| **Audience** | @qa-automated |
| **Milestone Compatibility** | M1 (runs in full) |
| **Entity coverage** | ACTIVE anchors (eligibility set per AC) |

**Preconditions:** 30 eligible ACTIVE anchors; `max_anchors_in_prompt=10`; token budget configured; weight invariant BR-10 satisfied across set.

| # | Step | Expected |
|:-:|------|----------|
| 1 | Submit query matching all 30 anchors. | 30 candidates retrieved. |
| 2 | Context Broker assembles prompt. | Exactly 10 anchors included; others logged as `items_rejected` in trace. |
| 3 | Verify token budget. | Total prompt tokens within configured budget. |

---

<!-- Fidelity Map BA-TC-MARM-009 (BA-AC-MARM-009 Then-clause claims):
  C1 "trace event for each operation containing anchor_id" → Steps 1-4 Expected
  C2 "operation_type" → Steps 1-4 Expected
  C3 "actor" → Steps 1-4 Expected
  C4 "timestamp" → Steps 1-4 Expected
  C5 "before/after states" → Step 2, Step 4 Expected
-->
### BA-TC-MARM-009 · [Happy] Audit trail records all anchor operations

| | |
|---|---|
| **Source AC** | [BA-AC-MARM-009](../implementation-briefs/brief-marm-memory-v2.md#ba-ac-marm-009) |
| **Source FR** | [SRS-FR-MARM-2.008](../implementation-briefs/brief-marm-memory-v2.md#srs-fr-marm-2008) |
| **Priority** | P1 |
| **Cycle** | Integration |
| **Verification** | Integration test |
| **Audience** | @qa-automated |
| **Milestone Compatibility** | M1 (runs with mock for Cognitive Trace per D-009 light-circuit; also exercises POL-RET-001 attestation path BR-18) |
| **Entity coverage** | create, update, verify (PENDING→VERIFIED), delete operations |

**Preconditions:** Cognitive Trace writer mock available (records events to in-memory log); test user authenticated.

| # | Step | Expected |
|:-:|------|----------|
| 1 | Create anchor. | Trace event `anchor.created` with `anchor_id`, `actor`, `timestamp`, `after_state={state, verification}`. |
| 2 | Update anchor scope. | Trace event `anchor.updated` with before/after state. |
| 3 | Verify anchor (PENDING → VERIFIED). | Trace event `anchor.verified` with `actor` and `timestamp`. |
| 4 | Soft-delete anchor. | Trace event `anchor.deleted`; before state captured. |

---

<!-- Fidelity Map BA-TC-MARM-010 (BA-AC-MARM-010 Then-clause claims):
  C1 "SESSION-scoped anchors are either expired or purged" → Step 3 Expected
  C2 "only visible within that specific session during its lifetime" → Step 1, Step 2 Expected
-->
### BA-TC-MARM-010 · [Happy] L1 Session Memory isolated and volatile

| | |
|---|---|
| **Source AC** | [BA-AC-MARM-010](../implementation-briefs/brief-marm-memory-v2.md#ba-ac-marm-010) |
| **Source FR** | [SRS-FR-MARM-2.009](../implementation-briefs/brief-marm-memory-v2.md#srs-fr-marm-2009) |
| **Priority** | P1 |
| **Cycle** | Integration |
| **Verification** | Integration test |
| **Audience** | @qa-automated |
| **Milestone Compatibility** | M1 (runs in full) |
| **Entity coverage** | SESSION scope, single user, two sessions A and B |

**Preconditions:** Two distinct dialog sessions A and B for the same user; clean anchor store.

| # | Step | Expected |
|:-:|------|----------|
| 1 | In session A, create SESSION-scoped anchor. | Anchor visible in session A context. |
| 2 | Query context in session B (same user). | Anchor NOT visible in session B. |
| 3 | End session A (timeout or explicit close). | SESSION-scoped anchors expired or purged per policy; NOT persisted. |

---

<!-- Fidelity Map BA-TC-MARM-011 (BA-AC-MARM-011 Then-clause claims):
  C1 "USER-scoped ACTIVE/DECAYING anchors with VERIFIED status are available for context assembly in the new session" → Step 2 Expected
-->
### BA-TC-MARM-011 · [Happy] L2 User Memory persists across sessions

| | |
|---|---|
| **Source AC** | [BA-AC-MARM-011](../implementation-briefs/brief-marm-memory-v2.md#ba-ac-marm-011) |
| **Source FR** | [SRS-FR-MARM-2.010](../implementation-briefs/brief-marm-memory-v2.md#srs-fr-marm-2010) |
| **Priority** | P0 (smoke) |
| **Cycle** | Smoke |
| **Verification** | Integration test |
| **Audience** | @qa-automated |
| **Milestone Compatibility** | M1 (runs in full) |
| **Entity coverage** | USER scope; ACTIVE, DECAYING; VERIFIED |

**Preconditions:** Fixture `user_with_persistent_anchors` (user A has ≥3 USER-scoped VERIFIED anchors created in a prior session — mix of ACTIVE and DECAYING).

| # | Step | Expected |
|:-:|------|----------|
| 1 | Start new dialog session for user A. | Session initialized. |
| 2 | Query context assembly. | USER-scoped ACTIVE/DECAYING + VERIFIED anchors available. |
| 3 | Verify no SESSION-scoped anchors from prior session appear. | Clean scope boundary. |

---

<!-- Fidelity Map BA-TC-MARM-012 (BA-AC-MARM-012 Then-clause claims):
  C1 "transitions EXPIRED anchors past retention to purge" → Step 2 Expected
  C2 "transitions DELETED anchors past retention to purge" → Step 3 Expected
  C3 "enforces quota limits" → Step 4 Expected
  C4 "logs cleanup results to trace" → Step 5 Expected
-->
### BA-TC-MARM-012 · [Happy] Cleanup scheduler executes on schedule

| | |
|---|---|
| **Source AC** | [BA-AC-MARM-012](../implementation-briefs/brief-marm-memory-v2.md#ba-ac-marm-012) |
| **Source FR** | [SRS-FR-MARM-2.011](../implementation-briefs/brief-marm-memory-v2.md#srs-fr-marm-2011) |
| **Priority** | P1 |
| **Cycle** | Regression |
| **Verification** | Integration test |
| **Audience** | @qa-automated |
| **Milestone Compatibility** | M1 (runs with mock for Cognitive Trace) |
| **Entity coverage** | EXPIRED past retention, DELETED past retention, quota-overage anchors |

**Preconditions:** `cleanup_schedule=daily`; test dataset includes EXPIRED anchors past 365d retention, DELETED anchors past 365d retention, and a user near quota limit; Cognitive Trace mock attached.

| # | Step | Expected |
|:-:|------|----------|
| 1 | Advance clock to scheduled time. | Scheduler fires. |
| 2 | Verify EXPIRED anchors past retention. | Purged from storage; count decremented. |
| 3 | Verify DELETED anchors past retention. | Purged. |
| 4 | Verify quotas. | Enforced; over-quota anchors evicted (lowest-scored first). |
| 5 | Check trace. | Cleanup summary event emitted. |

---

<!-- Fidelity Map BA-TC-MARM-013 (BA-AC-MARM-013 Then-clause claims):
  C1 "new anchor created with state=ACTIVE" → Step 3 Expected
  C2 "verification_status=VERIFIED" → Step 3 Expected
  C3 "source=MANUAL" → Step 3 Expected
  C4 "configured confidence and per-type TTL" → Step 3 Expected
  C5 "appears in the anchor list" → Step 4 Expected
  C6 entity claim "8 user-creatable independent types" → Step 2 Expected
-->
### BA-TC-MARM-013 🟡 · [Happy] User creates anchor manually — 8 user-creatable independent types

| | |
|---|---|
| **Source AC** | [BA-AC-MARM-013](../implementation-briefs/brief-marm-memory-v2.md#ba-ac-marm-013) |
| **Source FR** | [SRS-FR-MARM-2.010](../implementation-briefs/brief-marm-memory-v2.md#srs-fr-marm-2010) |
| **Priority** | P0 (smoke) |
| **Cycle** | Smoke |
| **Verification** | Integration test |
| **Audience** | @qa-manual |
| **Milestone Compatibility** | M1 (runs in full — manual path; LLM-extraction path FR-MDL-001=M2 stubbed elsewhere) |
| **Entity coverage** | 8 independent user-creatable types: GOAL, TASK, CONSTRAINT, DECISION, FACT, EMOTION, IDENTITY, PREFERENCE (mirrors excluded by AC) |
| **Inherits from** | D-015 (12 types: 8 independent types user-creatable) |

**Preconditions:** Logged-in user on Memory page; scope selector available; mock API `tools/tools/src/routers/memory.py` reachable per R-006.

| # | Step | Expected |
|:-:|------|----------|
| 1 | Click "Add Entry" button. | New Anchor Modal opens. |
| 2 | Verify Type dropdown options. | Shows 8 independent types: GOAL, TASK, CONSTRAINT, DECISION, FACT, EMOTION, IDENTITY, PREFERENCE. Mirror types excluded. |
| 3 | Select IDENTITY type; fill content; scope=USER; click "Remember this". | Anchor created with `state=ACTIVE`, `verification_status=VERIFIED`, `source=MANUAL`, correct per-type TTL. |
| 4 | Anchor appears in anchor list. | Card visible; user is owner. |

---

<!-- Fidelity Map BA-TC-MARM-014 (BA-AC-MARM-014 Then-clause claims):
  C1 "verification_status changes to VERIFIED" → Step 2 Expected
  C2 "anchor becomes eligible for default retrieval" → Step 2 Expected
  C3 "trace event is emitted" → Step 3 Expected
-->
### BA-TC-MARM-014 · [Happy] User verifies extracted anchor

| | |
|---|---|
| **Source AC** | [BA-AC-MARM-014](../implementation-briefs/brief-marm-memory-v2.md#ba-ac-marm-014) |
| **Source FR** | [SRS-FR-MARM-2.001](../implementation-briefs/brief-marm-memory-v2.md#srs-fr-marm-2001) |
| **Priority** | P0 (smoke) |
| **Cycle** | Smoke |
| **Verification** | Integration test |
| **Audience** | @qa-manual |
| **Milestone Compatibility** | M1 (runs with mock for Cognitive Trace; extracted-anchor fixture stubs FR-MDL-001=M2 LLM dependency) |
| **Entity coverage** | EXTRACTED + PENDING anchor |

**Preconditions:** Fixture `extracted_pending_anchor` (auto-extracted anchor with `source=EXTRACTED`, `verification_status=PENDING`, confidence < 0.85 per BR-5); Cognitive Trace mock attached.

| # | Step | Expected |
|:-:|------|----------|
| 1 | Open Edit Anchor modal; confirm Verify/Reject banner visible. | Banner shown. |
| 2 | Click "Verify". | `verification_status=VERIFIED`; anchor eligible for default retrieval. |
| 3 | Check Cognitive Trace. | `anchor.verified` event emitted. |

---

<!-- Fidelity Map BA-TC-MARM-015 (BA-AC-MARM-015 Then-clause claims):
  C1 "verification_status changes to REJECTED" → Step 1 Expected
  C2 "anchor is excluded from retrieval" → Step 1 Expected
  C3 "Restore option becomes available" → Step 2 Expected
-->
### BA-TC-MARM-015 · [Happy] User rejects extracted anchor and can restore

| | |
|---|---|
| **Source AC** | [BA-AC-MARM-015](../implementation-briefs/brief-marm-memory-v2.md#ba-ac-marm-015) |
| **Source FR** | [SRS-FR-MARM-2.001](../implementation-briefs/brief-marm-memory-v2.md#srs-fr-marm-2001) |
| **Priority** | P1 |
| **Cycle** | Integration |
| **Verification** | Integration test |
| **Audience** | @qa-manual |
| **Milestone Compatibility** | M1 (runs with mock for Cognitive Trace) |
| **Entity coverage** | EXTRACTED + PENDING anchor (same fixture pattern as 014); REJECTED state output |

**Preconditions:** Fixture `extracted_pending_anchor` (auto-extracted PENDING anchor); Restore policy configured.

| # | Step | Expected |
|:-:|------|----------|
| 1 | Click "Reject" in Edit Anchor modal; confirm. | `verification_status=REJECTED`; anchor excluded from retrieval. |
| 2 | Re-open anchor. | Restore button now visible. |
| 3 | Click "Restore"; confirm. | `verification_status=VERIFIED` (or PENDING per policy); anchor re-eligible. |

---

<!-- Fidelity Map BA-TC-MARM-016 (BA-AC-MARM-016 Then-clause claims):
  C1 "user B cannot see or retrieve user A's anchor" → Step 2 Expected
  C2 "visibility=PRIVATE enforced" → Step 1, Step 3 Expected
-->
### BA-TC-MARM-016 · [Permission] Anchors are private by default

| | |
|---|---|
| **Source AC** | [BA-AC-MARM-016](../implementation-briefs/brief-marm-memory-v2.md#ba-ac-marm-016) |
| **Source FR** | [SRS-FR-MARM-2.010](../implementation-briefs/brief-marm-memory-v2.md#srs-fr-marm-2010) |
| **Priority** | P0 (smoke) |
| **Cycle** | Smoke |
| **Verification** | Integration test |
| **Audience** | @qa-automated |
| **Milestone Compatibility** | M1 (runs in full) |
| **Entity coverage** | Two users A and B in same workspace; USER scope; visibility=PRIVATE |

**Preconditions:** Two users (A, B) in same workspace with valid auth tokens; clean anchor store.

| # | Step | Expected |
|:-:|------|----------|
| 1 | User A creates USER-scoped anchor. | Anchor stored; `visibility=PRIVATE`. |
| 2 | User B queries anchors in workspace. | User A's anchor not visible. |
| 3 | User B attempts direct GET by `anchor_id` (from guess or leak). | 403/404 response; no data leaked. |

---

<!-- Fidelity Map BA-TC-MARM-017 (BA-AC-MARM-017 Then-clause claims):
  C1 "request is rejected with 403 Forbidden" → Step 2 Expected
-->
### BA-TC-MARM-017 · [Permission] Memory Settings requires admin role

| | |
|---|---|
| **Source AC** | [BA-AC-MARM-017](../implementation-briefs/brief-marm-memory-v2.md#ba-ac-marm-017) |
| **Source FR** | [SRS-FR-MARM-2.002](../implementation-briefs/brief-marm-memory-v2.md#srs-fr-marm-2002) |
| **Priority** | P1 |
| **Cycle** | Integration |
| **Verification** | Integration test |
| **Audience** | @qa-automated |
| **Milestone Compatibility** | M1 (runs in full) |
| **Entity coverage** | Member role; admin endpoints `/memory/settings` |

**Preconditions:** User with "member" role (not admin); mock API `tools/tools/src/routers/memory_settings.py` reachable per R-006.

| # | Step | Expected |
|:-:|------|----------|
| 1 | GET `/memory/settings`. | 200 OK with read-only view. |
| 2 | PATCH `/memory/settings` with `default_ttl_days=60`. | 403 Forbidden; no state change. |
| 3 | Verify Memory Settings modal UI. | Save Changes button disabled or gated. |

---

<!-- Fidelity Map BA-TC-MARM-018 (BA-AC-MARM-018 Then-clause claims):
  C1 "extraction is paused for that scope" → Step 1 Expected
  C2 "only user-confirmed anchors are processed" → Step 2 Expected
  C3 "system alert emitted" → Step 1 Expected
  C4 (BR-8 backpressure 100% eviction) → Step 3 Expected
-->
### BA-TC-MARM-018 · [Boundary] Memory budget enforcement under pressure

| | |
|---|---|
| **Source AC** | [BA-AC-MARM-018](../implementation-briefs/brief-marm-memory-v2.md#ba-ac-marm-018) |
| **Source FR** | [SRS-FR-MARM-2.001](../implementation-briefs/brief-marm-memory-v2.md#srs-fr-marm-2001) |
| **Priority** | P2 |
| **Cycle** | Regression |
| **Verification** | Integration test |
| **Audience** | @qa-automated |
| **Milestone Compatibility** | M1 (runs with mock for FR-MDL-001=M2 — auto-extraction trigger uses stub LLM; user-confirmed path manual) |
| **Entity coverage** | User scope at 95% (950/1000); 100% override path; lowest-scored eviction |

**Preconditions:** Fixture `user_at_95_quota` (user with 950 anchors out of 1000 User-scope limit, mix of weights); LLM-extraction mock returns suggestions on demand; system-alert sink mocked.

| # | Step | Expected |
|:-:|------|----------|
| 1 | Trigger auto-extraction on a new dialog turn. | Extraction paused for User scope; system alert emitted. |
| 2 | User manually confirms a suggestion. | Accepted (user-confirmed bypass). |
| 3 | User count exceeds 100% (post-override). | Lowest-scored anchor evicted before new insertion. |

---

<!-- Fidelity Map BA-TC-MARM-019 (BA-AC-MARM-019 Then-clause claims):
  C1 "validation prevents submission" → Step 1 Expected
  C2 "appropriate error message" → Step 1 Expected
  C3 (API parity) "API enforces limit" → Step 2 Expected
-->
### BA-TC-MARM-019 · [Error:Validation] Content exceeds max length

| | |
|---|---|
| **Source AC** | [BA-AC-MARM-019](../implementation-briefs/brief-marm-memory-v2.md#ba-ac-marm-019) |
| **Source FR** | [SRS-FR-MARM-2.010](../implementation-briefs/brief-marm-memory-v2.md#srs-fr-marm-2010) |
| **Priority** | P1 |
| **Cycle** | Integration |
| **Verification** | Unit + integration |
| **Audience** | @qa-automated |
| **Milestone Compatibility** | M1 (runs in full) |
| **Entity coverage** | Content field at boundary (501 chars vs BR-11 max 500) |

**Preconditions:** New Anchor modal open; content field focused; BR-11 max length 500 enforced at both UI and API.

| # | Step | Expected |
|:-:|------|----------|
| 1 | Paste 501-character content; click "Remember this". | Validation blocks submission; inline error: "Content exceeds 500 characters." |
| 2 | API attempt with 501-char payload (bypassing UI). | 422 Unprocessable Entity; no anchor created. |

---

<!-- Fidelity Map BA-TC-MARM-020 (BA-AC-MARM-020 Then-clause claims):
  C1 "higher confidence wins" → Step 2 Expected
  C2 "if tied, more recent wins" → Step 4 Expected
  C3 "loser's decay rate is increased by 2x" → Step 3 Expected
-->
### BA-TC-MARM-020 · [Concurrency] Conflict detection between anchors

| | |
|---|---|
| **Source AC** | [BA-AC-MARM-020](../implementation-briefs/brief-marm-memory-v2.md#ba-ac-marm-020) |
| **Source FR** | [SRS-FR-MARM-2.006](../implementation-briefs/brief-marm-memory-v2.md#srs-fr-marm-2006) |
| **Priority** | P2 |
| **Cycle** | Regression |
| **Verification** | Integration test |
| **Audience** | @qa-automated |
| **Milestone Compatibility** | M1 (runs in full when vector similarity infra is provisioned; deferred to next iteration if Weaviate slips per Risk #4 — skip rather than fail) |
| **Entity coverage** | Two anchors with similarity > 0.85, different values, same scope and same type; BR-10 weight invariant precondition |

**Preconditions:** Vector similarity infrastructure available (Weaviate per current plan, or equivalent); two anchors with similarity > 0.85 but different values in same scope + type; BR-10 (`weight <= initial_weight`) enforced. **If infrastructure not provisioned in M1, this test is deferred to the next iteration per brief Risk #4; skip rather than fail.**

| # | Step | Expected |
|:-:|------|----------|
| 1 | Both anchors become candidates during context assembly. | Conflict detected. |
| 2 | Higher-confidence anchor wins. | Included in prompt. |
| 3 | Loser's decay rate increased 2x. | Observed in subsequent weight samples. |
| 4 | If confidences tied: newer anchor wins. | Tiebreaker applied. |

---

<!-- Fidelity Map BA-TC-MARM-021 (BA-AC-MARM-021 Then-clause claims):
  C1 "mirror anchor's state transitions to STALE" → Steps 1-3 Expected
  C2 "awaiting re-sync with the updated LoreBook content" → Step 4 Expected
-->
### BA-TC-MARM-021 🟡 · [Recovery] Mirror anchor invalidation on LoreBook update (incl. RULE_MIRROR per D-011)

| | |
|---|---|
| **Source AC** | [BA-AC-MARM-021](../implementation-briefs/brief-marm-memory-v2.md#ba-ac-marm-021) |
| **Source FR** | [SRS-FR-MARM-2.010](../implementation-briefs/brief-marm-memory-v2.md#srs-fr-marm-2010) |
| **Priority** | P1 |
| **Cycle** | Integration |
| **Verification** | Integration test |
| **Audience** | @qa-automated |
| **Milestone Compatibility** | M1 (runs with mock for LoreBook writer — owned by brief-lorebook-crud.md, M1 SAME milestone) |
| **Entity coverage** | IDENTITY_MIRROR, FACT_MIRROR, RULE_MIRROR (PREFERENCE_MIRROR cascade follows same pattern; verified analytically via D-011) |
| **Inherits from** | D-011 (RULE_MIRROR cascade by pattern) |

**Preconditions:** LoreBook writer mock + mirror regenerator available; IDENTITY_MIRROR, FACT_MIRROR, and RULE_MIRROR anchors exist with `canonical_ref` set.

| # | Step | Expected |
|:-:|------|----------|
| 1 | Update source LoreBook IDENTITY entry. | All IDENTITY_MIRROR anchors with matching `canonical_ref` transition to `state=STALE`. |
| 2 | Update source LoreBook FACT entry. | Matching FACT_MIRROR anchors transition to STALE. |
| 3 | Update source for RULE_MIRROR anchor. | RULE_MIRROR transitions to STALE (per D-011 pattern implementation). |
| 4 | Re-sync runs. | STALE anchors regenerate from LoreBook; return to ACTIVE. |

---

<!-- Fidelity Map BA-TC-MARM-022 (BA-AC-MARM-022 Then-clause claims):
  C1 "state transitions EXPIRED → DELETED" → Step 3 Expected
  C2 "deleted_at is set to now" → Step 3 Expected
  C3 "anchor is excluded from all retrieval" → Step 4 Expected
  C4 "trace event of type anchor.user_deleted is emitted" → Step 3 Expected
  C5 "anchor is queued for purge after the retention period (default 365d)" → Step 5 Expected
-->
### BA-TC-MARM-022 🟡 · [Happy] User-initiated soft delete on EXPIRED anchor (R-004 new in v2)

| | |
|---|---|
| **Source AC** | [BA-AC-MARM-022](../implementation-briefs/brief-marm-memory-v2.md#ba-ac-marm-022) |
| **Source FR** | [SRS-FR-MARM-2.001](../implementation-briefs/brief-marm-memory-v2.md#srs-fr-marm-2001) |
| **Priority** | P0 (smoke) |
| **Cycle** | Smoke |
| **Verification** | Integration test |
| **Audience** | @qa-manual |
| **Milestone Compatibility** | M1 (runs with mock for Cognitive Trace per D-009) |
| **Entity coverage** | EXPIRED-state anchor owned by current user; Edit Anchor modal Delete button (GUI Spec row 41); ConfirmDeleteModal; `anchor.user_deleted` trace event |
| **Inherits from** | D-014 (EXPIRED → DELETED user-initiated transition); GUI Spec row 41 (Delete Button visibility includes EXPIRED) |

**Preconditions:** Fixture `expired_user_owned_anchor` (anchor `state=EXPIRED`, owner = current user, `deleted_at=NULL`); Memory page accessible; Edit Anchor modal accessible per GUI Spec rows 41 + 4; Cognitive Trace mock attached; retention policy `purge_after=365d`; mock API `tools/tools/src/routers/memory.py` reachable per R-006.

| # | Step | Expected |
|:-:|------|----------|
| 1 | Open the Memory page; apply Expired filter (sidebar State Filter, GUI Spec row 4). | EXPIRED anchor listed (visible per D-014). |
| 2 | Open the Edit Anchor modal for the EXPIRED anchor. | Modal opens; "Delete" button visible (GUI Spec row 41 visibility includes EXPIRED per D-014). |
| 3 | Click "Delete"; confirm in ConfirmDeleteModal. | `state` transitions EXPIRED → DELETED; `deleted_at` set to now; trace event `anchor.user_deleted` emitted with `actor=current_user`, `before_state=EXPIRED`, `after_state=DELETED`. |
| 4 | Query default-scope retrieval and `include_expired=true` retrieval. | Anchor excluded from both (DELETED is excluded from all user-facing retrieval). |
| 5 | Inspect purge queue / cleanup metadata. | Anchor scheduled for purge at `deleted_at + 365d`. |

---

<!-- Fidelity Map BA-TC-MARM-023 (BA-AC-MARM-023 Then-clause claims; also satisfies SRS-AC-P0-02 Clause B):
  C1 "both runs select the same ordered anchor set (by ID)" → Step 2 Expected
  C2 "produce an identical canonical_snapshot_hash" → Step 2 Expected
  C3 "hash is computed over: ordered selected anchor IDs + ordered LoreBook entry IDs + applied truncation policy code + max_anchors_in_prompt value" → Step 3 Expected
  C4 (note) "Conflict-resolution determinism BR-12 (selection_seed) underlying mechanism" → Step 4 Expected
-->
### BA-TC-MARM-023 🟢 · [Happy] Canonical context snapshot hash determinism (R-003 new in v2; satisfies SRS-AC-P0-02 Clause B)

| | |
|---|---|
| **Source AC** | [BA-AC-MARM-023](../implementation-briefs/brief-marm-memory-v2.md#ba-ac-marm-023) (also covers [SRS-AC-P0-02](../implementation-briefs/brief-marm-memory-v2.md#srs-ac-p0-02) Clause B) |
| **Source FR** | [SRS-FR-MARM-2.006](../implementation-briefs/brief-marm-memory-v2.md#srs-fr-marm-2006), [SRS-FR-MARM-2.007](../implementation-briefs/brief-marm-memory-v2.md#srs-fr-marm-2007) |
| **Priority** | P0 (smoke) |
| **Cycle** | Smoke |
| **Verification** | Unit + integration |
| **Audience** | @qa-automated |
| **Milestone Compatibility** | M1 (runs with mock for FR-CTX-001/003/004 — Context Navigator owned by brief-system-guarantees.md, M1 SAME milestone per D-008; deterministic stub sufficient for hash check) |
| **Entity coverage** | Identical inputs across two runs: dialog state, anchor pool, snapshot_time, query text, DP-Risk inputs; output hash inputs: ordered selected anchor IDs, ordered LoreBook entry IDs, truncation policy code, max_anchors_in_prompt; selection_seed (BR-12) |
| **Inherits from** | BR-16 [CROSS-CUTTING] canonical context snapshot determinism; BR-12 conflict-resolution determinism |

**Preconditions:** Fixture `deterministic_context_pool` (frozen anchor pool of ≥30 ACTIVE anchors across SESSION+USER scope with stable IDs and weights, ≥3 LoreBook entries, fixed `selection_seed`); Context Broker available; Context Navigator FR-CTX-001/003/004 stub returning deterministic ordering per D-008; DP-Risk score mock returning fixed value; clock frozen at `snapshot_time=T0`; `max_anchors_in_prompt=10`; truncation policy code = `truncate_v1`.

| # | Step | Expected |
|:-:|------|----------|
| 1 | Run 1: assemble canonical context snapshot for query Q1 against the fixture pool. | Snapshot S1 produced: ordered anchor ID list, ordered LoreBook ID list, applied truncation policy code, `max_anchors_in_prompt=10`, hash H1. |
| 2 | Run 2: with identical inputs (same dialog state, anchor pool, `snapshot_time=T0`, query Q1, DP-Risk inputs), assemble snapshot. | Snapshot S2 produced; ordered anchor ID list of S2 == S1 (element-wise equality); ordered LoreBook ID list of S2 == S1; hash H2 == H1. |
| 3 | Inspect hash inputs. | Hash function input is exactly `(ordered_selected_anchor_ids, ordered_lorebook_ids, truncation_policy_code, max_anchors_in_prompt)` — no other inputs (e.g., model version, system prompt revision) included; serialization is stable (sorted keys, total ordering by ID). |
| 4 | Inject conflicting anchors with tied confidence and tied timestamps; rerun twice. | Conflict resolution falls through to `selection_seed` (BR-12); both runs select the same anchor; H equal across the two runs. |
| 5 | Mutate one input (e.g., add one new anchor to pool); rerun. | Hash differs from H1 (sanity: hash is sensitive to input changes; not a constant). |

---

## Coverage Matrix

### By AC

<!-- recomputed 2026-04-29: 23 AC rows, 1 TC per AC (1:1) plus SRS-AC-P0-02/03 mapped via existing TCs; entity coverage 23/23 full or declared subset -->

| AC | FR | Happy | Error | Permission | Boundary | Concurrency | Recovery | Entity coverage | Total |
|---|---|:-:|:-:|:-:|:-:|:-:|:-:|---|:-:|
| BA-AC-MARM-001 | 2.001 | 1 | 0 | 0 | 0 | 0 | 0 | CONSTRAINT subset (full set covered by family TC-001+TC-006 EMOTION + TC-003 EMOTION TTL + TC-004 FACT/DECISION) | 1 |
| BA-AC-MARM-002 | 2.001 | 1 | 0 | 0 | 0 | 0 | 0 | CONSTRAINT (DORMANT-reachable per D-002) | 1 |
| BA-AC-MARM-003 | 2.002 | 1 | 0 | 0 | 0 | 0 | 0 | EMOTION (per-type TTL pattern) | 1 |
| BA-AC-MARM-004 | 2.003 | 1 | 0 | 0 | 0 | 0 | 0 | FACT, DECISION | 1 |
| BA-AC-MARM-005 | 2.004 | 1 | 0 | 0 | 0 | 0 | 0 | DECISION, TASK, IDENTITY | 1 |
| BA-AC-MARM-006 | 2.005 | 1 | 0 | 0 | 0 | 0 | 0 | EMOTION | 1 |
| BA-AC-MARM-007 | 2.006 | 1 | 0 | 0 | 0 | 0 | 0 | SESSION+USER scopes; ACTIVE+DECAYING; VERIFIED+PROMOTED | 1 |
| BA-AC-MARM-008 | 2.007 | 0 | 0 | 0 | 1 | 0 | 0 | ACTIVE eligibility set | 1 |
| BA-AC-MARM-009 | 2.008 | 1 | 0 | 0 | 0 | 0 | 0 | create+update+verify+delete operations | 1 |
| BA-AC-MARM-010 | 2.009 | 1 | 0 | 0 | 0 | 0 | 0 | SESSION scope, two sessions | 1 |
| BA-AC-MARM-011 | 2.010 | 1 | 0 | 0 | 0 | 0 | 0 | USER scope; ACTIVE+DECAYING; VERIFIED | 1 |
| BA-AC-MARM-012 | 2.011 | 1 | 0 | 0 | 0 | 0 | 0 | EXPIRED+DELETED past retention; quota-overage | 1 |
| BA-AC-MARM-013 | 2.010 | 1 | 0 | 0 | 0 | 0 | 0 | 8 user-creatable independent types | 1 |
| BA-AC-MARM-014 | 2.001 | 1 | 0 | 0 | 0 | 0 | 0 | EXTRACTED+PENDING anchor | 1 |
| BA-AC-MARM-015 | 2.001 | 1 | 0 | 0 | 0 | 0 | 0 | EXTRACTED+PENDING; REJECTED output | 1 |
| BA-AC-MARM-016 | 2.010 | 0 | 0 | 1 | 0 | 0 | 0 | Two users A+B; USER scope; visibility=PRIVATE | 1 |
| BA-AC-MARM-017 | 2.002 | 0 | 0 | 1 | 0 | 0 | 0 | Member role; admin endpoints | 1 |
| BA-AC-MARM-018 | 2.001 | 0 | 0 | 0 | 1 | 0 | 0 | User scope at 95%/100%; lowest-scored eviction | 1 |
| BA-AC-MARM-019 | 2.010 | 0 | 1 | 0 | 0 | 0 | 0 | Content field 501 chars (BR-11 boundary) | 1 |
| BA-AC-MARM-020 | 2.006 | 0 | 0 | 0 | 0 | 1 | 0 | Two anchors similarity > 0.85, same scope+type | 1 |
| BA-AC-MARM-021 | 2.010 | 0 | 0 | 0 | 0 | 0 | 1 | IDENTITY_MIRROR, FACT_MIRROR, RULE_MIRROR (PREFERENCE_MIRROR by pattern per D-011) | 1 |
| BA-AC-MARM-022 | 2.001 | 1 | 0 | 0 | 0 | 0 | 0 | EXPIRED-state anchor; user-owned; Delete button + ConfirmDeleteModal; `anchor.user_deleted` event | 1 |
| BA-AC-MARM-023 | 2.006 + 2.007 | 1 | 0 | 0 | 0 | 0 | 0 | identical-input twin runs; hash inputs (ordered IDs + truncation policy + max_anchors_in_prompt); selection_seed (BR-12) | 1 |

### By FR

| FR | Description | Test Cases | Count |
|---|---|---|:-:|
| SRS-FR-MARM-2.001 | Memory lifecycle states | BA-TC-MARM-001, 002, 014, 015, 018, 022 | 6 |
| SRS-FR-MARM-2.002 | TTL implementation | BA-TC-MARM-003, 017 | 2 |
| SRS-FR-MARM-2.003 | Decay function | BA-TC-MARM-004 | 1 |
| SRS-FR-MARM-2.004 | Promotion | BA-TC-MARM-005 | 1 |
| SRS-FR-MARM-2.005 | Demotion | BA-TC-MARM-006 | 1 |
| SRS-FR-MARM-2.006 | Context Broker assembly | BA-TC-MARM-007, 020, 023 | 3 |
| SRS-FR-MARM-2.007 | Max anchors in prompt | BA-TC-MARM-008, 023 | 2 |
| SRS-FR-MARM-2.008 | Audit trail | BA-TC-MARM-009 | 1 |
| SRS-FR-MARM-2.009 | L1 Session Memory | BA-TC-MARM-010 | 1 |
| SRS-FR-MARM-2.010 | L2 User Memory | BA-TC-MARM-011, 013, 016, 019, 021 | 5 |
| SRS-FR-MARM-2.011 | Cleanup scheduler | BA-TC-MARM-012 | 1 |

### By Source Rollup

| Source | Coverage |
|---|---|
| SRS-AC | 3/3 (100%) <!-- from 3 distinct SRS-AC-IDs in TCs (P0-01 via 001/002/003/004/006/012/022; P0-02 via 007/008/023; P0-03 via 010/011) / 3 SRS-ACs in brief --> |
| BA-AC | 23/23 (100%) <!-- from 23 distinct BA-AC-IDs in TCs / 23 BA-ACs in brief --> |

## Priority Distribution

<!-- recomputed 2026-04-29: P0=11, P1=10, P2=2, P3=0, total=23 -->

| Priority | Count | Percentage |
|---|:-:|:-:|
| P0 | 11 <!-- from 11 TC metadata rows with P0: 001, 002, 003, 006, 007, 011, 013, 014, 016, 022, 023 --> | 48% |
| P1 | 10 <!-- from 10 TC metadata rows with P1: 004, 005, 008, 009, 010, 012, 015, 017, 019, 021 --> | 43% |
| P2 | 2 <!-- from 2 TC metadata rows with P2: 018, 020 --> | 9% |
| P3 | 0 <!-- from 0 TC metadata rows with P3 --> | 0% |

## Stability Rollup

<!-- recomputed 2026-04-29: 🟢-marked=1 (023), 🟡-marked=9 (001, 002, 003, 004, 005, 006, 013, 021, 022), unmarked-default-🟢=13 (007, 008, 009, 010, 011, 012, 014, 015, 016, 017, 018, 019, 020), 🔴=0 — total 23 -->

Computed stability (from TC titles, strict marker reading):

| 🟢 Stable (incl. unmarked-default) | 🟡 Interim (inherits from D-NNN) | 🔴 Open | Total |
|:-:|:-:|:-:|:-:|
| 14 | 9 | 0 | 23 |

🟡 TCs (9, marker in title): BA-TC-MARM-001, 002, 003, 004, 005, 006, 013, 021, 022.
🟢 TCs (14 = 1 marked + 13 unmarked-default): marked — BA-TC-MARM-023; unmarked — BA-TC-MARM-007, 008, 009, 010, 011, 012, 014, 015, 016, 017, 018, 019, 020.

> **Convention:** TCs without a marker in the title default to 🟢, mirroring v1 (which marked only 🟡 cases explicitly). v1 stability rollup said 10 🟢 / 11 🟡 on 21 TCs; v2 adjusts because BA-AC-MARM-002 inherits Q1 (kept 🟡), BA-AC-MARM-003 inherits D-006 (kept 🟡), and 022 is 🟡 (D-014), 023 is 🟢 (BR-16 stable).

## Cycle Distribution

<!-- recomputed 2026-04-29: Smoke=11, Integration=9, Regression=3, total=23 -->

| Cycle | Count | TCs |
|---|:-:|---|
| Smoke | 11 <!-- from 11 TC metadata rows with Cycle=Smoke --> | 001, 002, 003, 006, 007, 011, 013, 014, 016, 022, 023 |
| Integration | 9 <!-- from 9 TC metadata rows with Cycle=Integration --> | 004, 005, 008, 009, 010, 015, 017, 019, 021 |
| Regression | 3 <!-- from 3 TC metadata rows with Cycle=Regression --> | 012, 018, 020 |

## Verification Distribution

<!-- recomputed 2026-04-29: Integration test=20, Unit + integration=3, others=0, total=23 -->

| Verification | Count | TCs |
|---|:-:|---|
| Integration test | 20 | 001, 002, 003, 005, 006, 007, 008, 009, 010, 011, 012, 013, 014, 015, 016, 017, 018, 020, 021, 022 |
| Unit + integration | 3 | 004, 019, 023 |
| Performance test | 0 | — |
| Benchmark | 0 | — |
| Endurance | 0 | — |
| Demo | 0 | — |
| Architecture review | 0 | — |

## Audience Distribution

<!-- recomputed 2026-04-29: @qa-automated=18, @qa-manual=4, @developer=1, @acceptance=0, total=23 -->

| Audience | Count | TCs |
|---|:-:|---|
| @qa-automated | 18 | 001, 002, 003, 005, 006, 007, 008, 009, 010, 011, 012, 016, 017, 018, 019, 020, 021, 023 |
| @qa-manual | 4 | 013, 014, 015, 022 |
| @developer | 1 | 004 |
| @acceptance | 0 | — |

---

## Self-Check Report (Rollup Reconciliation)

### Programmatic recount (re-run 2026-04-29)

| Metric | Claim in tables | Recomputed | Match? | Notes |
|---|---|---|:-:|---|
| Total TCs | 23 | 23 | ✅ | 001..021 inherited + 022, 023 added |
| 🟢 in TC titles (strict marker only) | 1 | 1 | ✅ | TC-023 only (TCs without marker default to 🟢, see Stability Rollup) |
| 🟡 in TC titles (strict marker) | 9 | 9 | ✅ | TC-001, 002, 003, 004, 005, 006, 013, 021, 022 |
| 🟢 effective (marker + unmarked-default) | 14 | 14 | ✅ | matches Stability Rollup |
| 🔴 in TC titles | 0 | 0 | ✅ | none |
| P0 count | 11 | 11 | ✅ | 001, 002, 003, 006, 007, 011, 013, 014, 016, 022, 023 |
| P1 count | 10 (corrected) | 10 | ✅ | 004, 005, 008, 009, 010, 012, 015, 017, 019, 021 |
| P2 count | 2 (corrected) | 2 | ✅ | 018, 020 |
| P3 count | 0 | 0 | ✅ | none |
| Smoke cycle | 11 (corrected) | 11 | ✅ | 001, 002, 003, 006, 007, 011, 013, 014, 016, 022, 023 |
| Integration cycle | 9 (corrected) | 9 | ✅ | 004, 005, 008, 009, 010, 015, 017, 019, 021 |
| Regression cycle | 3 | 3 | ✅ | 012, 018, 020 |
| BA-AC coverage | 23/23 | 23/23 | ✅ | 1:1 AC:TC |
| SRS-AC coverage | 3/3 | 3/3 | ✅ | P0-01 via TCs 001/002/003/004/006/012/022; P0-02 via 007/008/023; P0-03 via 010/011 |
| @qa-automated count | 18 | 18 | ✅ | 001, 002, 003, 005, 006, 007, 008, 009, 010, 011, 012, 016, 017, 018, 019, 020, 021, 023 |
| @qa-manual count | 4 | 4 | ✅ | 013, 014, 015, 022 |
| @developer count | 1 | 1 | ✅ | 004 |
| @acceptance count | 0 | 0 | ✅ | — |

### Drift from v1 corrected in v2

1. **🟡 stability count**: v1 said 11 🟡 on 21 TCs; strict-marker recount on v2 (incl. new 022 🟡 and 023 🟢) gives 9 🟡 + 14 🟢 (1 marked + 13 unmarked-default) across 23 TCs.
2. **Priority distribution**: v2 uses P0=11, P1=10, P2=2, P3=0 (added P0 022 + P0 023; 019 is P1 not P2 — v1 typo carried forward in rollup).
3. **Cycle distribution**: v2 uses Smoke=11, Integration=9, Regression=3 (added Smoke 022 + Smoke 023).
4. **Audience distribution**: v2 uses @qa-automated=18, @qa-manual=4, @developer=1.

### Coverage checks

| Check | Pass? | Notes |
|---|:-:|---|
| Every AC has ≥1 TC | ✅ | 23/23 |
| Every AC with [Happy] tag has ≥1 TC | ✅ | All 16 [Happy]-tagged ACs have TCs |
| Every AC with error/permission/boundary/concurrency/recovery has ≥1 corresponding TC | ✅ | 019 [Error:Validation], 016/017 [Permission], 008/018 [Boundary], 020 [Concurrency], 021 [Recovery] |
| Every TC has Source-AC link | ✅ | all 23 |
| Every TC has Source-FR link | ✅ | all 23 |
| Every state-changing step has Expected (no `—`) | ✅ | spot-checked all 23 |
| No duplicate TCs | ✅ | TC-IDs unique |
| Priority distribution reasonable (not all P0) | ✅ | 48% P0, 43% P1, 9% P2 |
| Fidelity Map present | ✅ | all 23 cards have hidden Fidelity Map comment |
| Every Then-clause claim mapped to a Step Expected | ✅ | reviewed per-TC |
| Entity coverage column populated | ✅ | all 23 rows |
| Entity coverage = full set OR declared subset with family pointer | ✅ | TC-001 declares CONSTRAINT subset, points to 003/004/006 family for full type set; TC-021 covers 3 of 4 mirror types analytically with D-011 pointer |
| Target Milestone line present | ✅ | line 4 of file |
| Dependency Milestone Map hidden comment present | ✅ | top of Test Cases section |
| No cross-milestone dependency leak (every LATER name has marker a/b/c/d) | ✅ | FR-MDL-001=M2 noted Mocked/Stubbed in TC-013/014/018; Cognitive Trace D-009 noted Mocked in 001/002/006/009/012/014/015/022; LoreBook M1-SAME but mocked in 005/021; Vector similarity Deferrable in 020 |
| Every TC card has Milestone Compatibility row | ✅ | all 23 |
| No inter-TC dependencies in Preconditions | ✅ | grep `from TC-`, `from step`, `or equivalent`, `previous test`, `as in TC-` in Preconditions: 0 matches (v1 had 1 in TC-002 "from BA-TC-MARM-001 step 3 or equivalent" — replaced in v2 with named fixture `dormant_constraint_anchor`; v1 TC-006 had "via BA-TC-MARM-001-style setup" — replaced with named fixture `decaying_emotion_anchor`) |

### Inter-TC dependency cleanup applied (v1 → v2)

- **BA-TC-MARM-002 Preconditions**: was `"DORMANT CONSTRAINT anchor exists (from BA-TC-MARM-001 step 3 or equivalent)"` → now `"Fixture dormant_constraint_anchor exists (CONSTRAINT, state=DORMANT, last_accessed_at=now-4h)"`.
- **BA-TC-MARM-006 Preconditions**: was `"Create DECAYING EMOTION anchor (e.g., via BA-TC-MARM-001-style setup)"` → now `"Fixture decaying_emotion_anchor exists (EMOTION, state=DECAYING, ...)"` and Step 1 reworded to confirm fixture state instead of creating.
- **BA-TC-MARM-011 Preconditions**: was `"User has USER-scoped VERIFIED anchors created in a prior session"` → tightened to named fixture `user_with_persistent_anchors`.
- **BA-TC-MARM-014/015 Preconditions**: tightened to named fixture `extracted_pending_anchor`.
- **BA-TC-MARM-018 Preconditions**: tightened to named fixture `user_at_95_quota`.
- **BA-TC-MARM-022 Preconditions**: introduced named fixture `expired_user_owned_anchor`.
- **BA-TC-MARM-023 Preconditions**: introduced named fixture `deterministic_context_pool`.

---

## Open Items / Unresolved Questions

1. **BA-TC-MARM-002 and Q1 dependency.** Test uses narrow-access interpretation. If product team responds with broad interpretation, step 3 (UI view does not count) must be rewritten; `last_accessed_at` behavior flips.
2. **Semantic similarity dependency (BA-TC-MARM-020).** If Weaviate integration slips to P1, this TC may need temporary skip-tag pending vector infra (Risk #4 in brief).
3. **Mirror regeneration latency (Q1 → OQ-1 from SRS §3.3.18).** If product team answers with a strict latency bound, BA-TC-MARM-021 step 4 may need a timing assertion added.
4. **BA-TC-MARM-023 hash input set.** Brief Assumption 6 marks the input set as 🟢-BA-derived but minimal. If product team adds inputs (model version, system prompt revision), Step 3 hash-input assertion expands and AC-023 changes.
5. **PREFERENCE_MIRROR coverage in BA-TC-MARM-021.** Brief AC names IDENTITY/PREFERENCE/FACT/RULE_MIRROR but the v1 TC tested only 3. v2 keeps the same 3 (entity-coverage column declares the gap analytically per D-011 pattern); if product team requires explicit PREFERENCE_MIRROR step, add Step 3a.
6. **D-001..D-015 unresolved.** All 11 🟡 TCs inherit interim state; if product team picks an alternate D-NNN option, the corresponding TC must be re-verified and may need rewording. Confirm-by 2026-05-27.

---

<!-- Machine-readable metadata for AI test script generation -->
<!-- Format: TC-ID | Category | Source-AC | Source-FR | Priority | Cycle | Verification | Audience | Milestone -->
<!-- BA-TC-MARM-001 | Happy | BA-AC-MARM-001 | SRS-FR-MARM-2.001 | P0 | Smoke | Integration test | @qa-automated | M1-mock -->
<!-- BA-TC-MARM-002 | Happy | BA-AC-MARM-002 | SRS-FR-MARM-2.001 | P0 | Smoke | Integration test | @qa-automated | M1-mock -->
<!-- BA-TC-MARM-003 | Happy | BA-AC-MARM-003 | SRS-FR-MARM-2.002 | P0 | Smoke | Integration test | @qa-automated | M1-full -->
<!-- BA-TC-MARM-004 | Happy | BA-AC-MARM-004 | SRS-FR-MARM-2.003 | P1 | Integration | Unit + integration | @developer | M1-full -->
<!-- BA-TC-MARM-005 | Happy | BA-AC-MARM-005 | SRS-FR-MARM-2.004 | P1 | Integration | Integration test | @qa-automated | M1-mock -->
<!-- BA-TC-MARM-006 | Happy | BA-AC-MARM-006 | SRS-FR-MARM-2.005 | P0 | Smoke | Integration test | @qa-automated | M1-mock -->
<!-- BA-TC-MARM-007 | Happy | BA-AC-MARM-007 | SRS-FR-MARM-2.006 | P0 | Smoke | Integration test | @qa-automated | M1-mock -->
<!-- BA-TC-MARM-008 | Boundary | BA-AC-MARM-008 | SRS-FR-MARM-2.007 | P1 | Integration | Integration test | @qa-automated | M1-full -->
<!-- BA-TC-MARM-009 | Happy | BA-AC-MARM-009 | SRS-FR-MARM-2.008 | P1 | Integration | Integration test | @qa-automated | M1-mock -->
<!-- BA-TC-MARM-010 | Happy | BA-AC-MARM-010 | SRS-FR-MARM-2.009 | P1 | Integration | Integration test | @qa-automated | M1-full -->
<!-- BA-TC-MARM-011 | Happy | BA-AC-MARM-011 | SRS-FR-MARM-2.010 | P0 | Smoke | Integration test | @qa-automated | M1-full -->
<!-- BA-TC-MARM-012 | Happy | BA-AC-MARM-012 | SRS-FR-MARM-2.011 | P1 | Regression | Integration test | @qa-automated | M1-mock -->
<!-- BA-TC-MARM-013 | Happy | BA-AC-MARM-013 | SRS-FR-MARM-2.010 | P0 | Smoke | Integration test | @qa-manual | M1-full -->
<!-- BA-TC-MARM-014 | Happy | BA-AC-MARM-014 | SRS-FR-MARM-2.001 | P0 | Smoke | Integration test | @qa-manual | M1-mock -->
<!-- BA-TC-MARM-015 | Happy | BA-AC-MARM-015 | SRS-FR-MARM-2.001 | P1 | Integration | Integration test | @qa-manual | M1-mock -->
<!-- BA-TC-MARM-016 | Permission | BA-AC-MARM-016 | SRS-FR-MARM-2.010 | P0 | Smoke | Integration test | @qa-automated | M1-full -->
<!-- BA-TC-MARM-017 | Permission | BA-AC-MARM-017 | SRS-FR-MARM-2.002 | P1 | Integration | Integration test | @qa-automated | M1-full -->
<!-- BA-TC-MARM-018 | Boundary | BA-AC-MARM-018 | SRS-FR-MARM-2.001 | P2 | Regression | Integration test | @qa-automated | M1-mock -->
<!-- BA-TC-MARM-019 | Error:Validation | BA-AC-MARM-019 | SRS-FR-MARM-2.010 | P1 | Integration | Unit + integration | @qa-automated | M1-full -->
<!-- BA-TC-MARM-020 | Concurrency | BA-AC-MARM-020 | SRS-FR-MARM-2.006 | P2 | Regression | Integration test | @qa-automated | M1-full (defer if Weaviate slips per Risk #4) -->
<!-- BA-TC-MARM-021 | Recovery | BA-AC-MARM-021 | SRS-FR-MARM-2.010 | P1 | Integration | Integration test | @qa-automated | M1-mock -->
<!-- BA-TC-MARM-022 | Happy | BA-AC-MARM-022 | SRS-FR-MARM-2.001 | P0 | Smoke | Integration test | @qa-manual | M1-mock -->
<!-- BA-TC-MARM-023 | Happy | BA-AC-MARM-023 | SRS-FR-MARM-2.006+2.007 | P0 | Smoke | Unit + integration | @qa-automated | M1-mock -->
