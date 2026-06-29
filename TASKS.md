# ewing-eligibility-structuring — TASKS

> Status: Draft · Version: 0.1.0 · Last updated: 2026-06-28 · Owner: TBD (maintainer) ·
> Lane: donated (LLM-assisted extraction may run **funded**, budget-capped) · Risk: medium core
> / **high** patient-facing.
>
> **All tasks carry `verifiedNeed = false`** until a partner/beneficiary requestor is secured.
> **All patient-facing tasks carry `riskTier = high`** and cannot ship without the credentialed
> pediatric-oncologist + patient-advocate **blocking** sign-off (see PLAN.md §7).

---

## How these tasks map to Elyos

Each task below becomes a Task JSON validated against `packages/schema/src/schemas.ts`. Field
mapping:

- **id** — stable slug `ewing-elig-<area>-NNN`.
- **title** — short imperative.
- **project** — `"ewing-eligibility-structuring"`.
- **type** — `code | research | writing | data | design-spec | maintenance`.
- **lane** — `donated` (default) or `funded` (then `fundedBudgetUsd` required, hard cap).
- **priority** — `high | medium | low`.
- **domain** — tags, e.g. `["oncology","clinical-trials","ewing-sarcoma","informatics"]`.
- **riskTier** — `low | medium | high` (**high** for any patient-facing task).
- **urgent** — boolean (default false; no artificial urgency on a no-partner project).
- **deliverable** — `pr | dataset | document` (`translation` unused in v1).
- **tokenEstimate** — `small | medium | large`.
- **status** — `open | in-progress | review | delivered | done` (all start `open`).
- **context / objective / acceptanceCriteria[] / resources[] / output** — task detail.
- **requestor** — partner contact (`TO BE SECURED`).
- **verifiedNeed** — `false` until partner confirms.
- **outputLicense** — `Apache-2.0` (code), `CC-BY-4.0` (schema/data/docs).

Sizes: `small` ≈ ≤½ day; `medium` ≈ 1–3 days; `large` ≈ multi-day / multi-session.

---

## Milestone M0 — Cold-start foundation

Goal: schema v0.1, guardrail + licensing + provenance framework, and a double-reviewed
gold-standard set. No scale-up until M0 exits.

| ID | Title | Type | Size | Risk | Deliverable | Depends on | Reviewer |
|---|---|---|---|---|---|---|---|
| ewing-elig-schema-001 | Design eligibility data model v0.1 (JSON Schema, FHIR-aligned) | design-spec | medium | medium | document | — | Maintainer + clin-informatics |
| ewing-elig-guard-002 | Author guardrail + not-advice + provenance policy | writing | small | medium | document | — | Maintainer |
| ewing-elig-license-003 | Build License Register; verify every M0 source | research | medium | medium | document | — | Maintainer |
| ewing-elig-gold-004 | Manually structure + independently double-review 5 trials (gold standard) | data | medium | medium | dataset | 001, 002, 003 | Two annotators + adjudicator |
| ewing-elig-ci-005 | CI: schema validation + PII-redaction check scaffold | code | small | low | pr | 001 | Engineer |

**Acceptance criteria — key tasks**

- **ewing-elig-schema-001**
  - JSON Schema (draft-07) validates with AJV; consistent style with `packages/schema`.
  - Models: concept (system+code+display), operator, value, unit, temporality, negation,
    age band, performance-status scale, ambiguity flag, confidence, extraction method.
  - **Provenance block is `required`** (trialId, registry, registryVersion, retrievedAt,
    sourceText, sourceTextSha256, reviewer, status).
  - Fixed not-advice + not-eligibility-determination disclaimer field is `required`.
  - Documented FHIR `ResearchStudy`/eligibility alignment + OMOP mapping notes.
- **ewing-elig-license-003**
  - Every source (ClinicalTrials.gov v2, ICTRP, CTIS, ISRCTN, each terminology) has a row:
    license/terms, reuse+derivative permission, attribution, redistribution constraints.
  - Sources with unclear terms are marked **excluded**; NCIt confirmed as primary terminology.
  - Output-license decision recorded (Apache-2.0 code / CC-BY-4.0 annotations).
- **ewing-elig-gold-004**
  - 5 trials structured independently by 2 annotators; Cohen's κ ≥ 0.75; disagreements adjudicated.
  - 100% provenance on every assertion; ambiguous criteria flagged, not guessed.
  - No investigator/contact PII present.

**Definition of Done (M0):** schema validates in CI; License Register complete with all M0 sources
verified open; 5-trial gold standard double-reviewed (κ ≥ 0.75) with full provenance; guardrail/
not-advice/provenance policy ratified; PII-redaction check runs in CI. `verifiedNeed` still false.

---

## Milestone M1 — Pilot pipeline (25 trials)

Goal: build the reproducible ingest→extract→code→validate→review pipeline and release a
provenanced 25-trial pilot dataset.

| ID | Title | Type | Size | Risk | Deliverable | Depends on | Reviewer |
|---|---|---|---|---|---|---|---|
| ewing-elig-ingest-101 | ClinicalTrials.gov v2 ingester (+ PII drop, hash, version capture) | code | medium | medium | pr | M0 | Engineer |
| ewing-elig-rules-102 | Deterministic rule extractor (age, perf-status, labs, washout, negation) | code | large | medium | pr | 101, schema-001 | Engineer + clin-informatics |
| ewing-elig-llm-103 | LLM-assist extractor adapter (funded, budget-capped, review-only output) | code | medium | medium | pr | 102 | Maintainer |
| ewing-elig-code-104 | Terminology coding to NCIt/HGNC/RxNorm/LOINC/ICD-O-3/HPO | code | large | medium | pr | 102 | Clin-informatics |
| ewing-elig-pilot-105 | Release 25-trial pilot dataset (100% provenance) + Zenodo DOI | data | medium | medium | dataset | 102, 104 | Clin-informatics + Maintainer |
| ewing-elig-regress-106 | Wire gold-standard F1 regression gate into CI | code | small | medium | pr | gold-004, 102 | Engineer |

**Acceptance criteria — key tasks**

- **ewing-elig-rules-102**
  - Deterministic extraction of common patterns with unit + temporality + negation handling.
  - Age-appropriate performance status (Lansky <16, Karnofsky/ECOG ≥16) with scale recorded.
  - No criterion auto-marked complete if ambiguous → routed to `human-must-resolve`.
- **ewing-elig-llm-103**
  - Runs only via `packages/runner` with a hard `fundedBudgetUsd` per-trial cap.
  - LLM output is **never auto-merged**; always enters the human-review queue.
  - No secrets/keys in logs, receipts, or output.
- **ewing-elig-pilot-105**
  - 25 trials released; **100% provenance** (validation blocks release otherwise).
  - F1 ≥ 0.85 vs gold standard; ambiguity items withheld; cross-registry dups deduped.
  - Every record carries the not-advice + not-eligibility disclaimer; CC-BY-4.0; Zenodo DOI.

**Definition of Done (M1):** pipeline reproducible from a clean clone; 25-trial pilot released with
100% provenance and F1 ≥ 0.85; gold-standard regression gate green in CI; ambiguity queue working;
DOI minted; no PII, no secrets leaked.

---

## Milestone M2 — Scale & coding

Goal: extend to the full agreed open-registry Ewing/adjacent pediatric-sarcoma set, complete
terminology coding, raise accuracy, and add freshness.

| ID | Title | Type | Size | Risk | Deliverable | Depends on | Reviewer |
|---|---|---|---|---|---|---|---|
| ewing-elig-multireg-201 | Add ICTRP/CTIS/ISRCTN ingest + cross-registry dedup | code | medium | medium | pr | 101, license-003 | Engineer |
| ewing-elig-scale-202 | Structure full open Ewing/pediatric-sarcoma set | data | large | medium | dataset | M1, 201 | Clin-informatics + Maintainer |
| ewing-elig-audit-203 | Random expert accuracy audit (≥95% correct) + error fixes | research | medium | medium | document | 202 | Clin-informatics |
| ewing-elig-fresh-204 | Freshness/refresh job: hash drift + staleness flagging | code | medium | medium | pr | 101, 202 | Engineer |
| ewing-elig-fhir-205 | FHIR ResearchStudy/eligibility export + OMOP mapping doc | code | medium | low | pr | schema-001, 202 | Engineer |

**Acceptance criteria — key tasks**

- **ewing-elig-scale-202** — coverage = agreed open-registry set; F1 ≥ 0.90; 100% provenance;
  coverage ledger published; SNOMED secondary mappings only where consumer is licensed.
- **ewing-elig-fresh-204** — refresh re-pulls sources, compares `sourceTextSha256`, flags drift +
  stale "recruiting" claims; current status deferred to live registry link, never asserted.

**Definition of Done (M2):** full agreed coverage released; F1 ≥ 0.90; random audit ≥ 95% correct;
freshness job live; FHIR/OMOP export available; coverage ledger published.

---

## Milestone M3 — Patient-facing layer (HIGH risk; conditional)

Goal: education-only, plain-language explainers of what eligibility concepts mean. **Starts only
if a partner AND the oncologist + patient-advocate panel are secured.** Every task is `riskTier:
high` and blocked from shipping without both sign-offs.

| ID | Title | Type | Size | Risk | Deliverable | Depends on | Reviewer |
|---|---|---|---|---|---|---|---|
| ewing-elig-pf-spec-301 | Patient-facing content spec + not-advice template + reading-level rules | design-spec | small | high | document | M2, partner secured | Oncologist + advocate (blocking) |
| ewing-elig-pf-write-302 | Draft plain-language eligibility-concept explainers (sourced NCI/COG/ESMO/CCLG) | writing | large | high | document | 301 | Oncologist + advocate (blocking) |
| ewing-elig-pf-review-303 | Blocking expert + advocate sign-off + beneficiary usefulness survey | research | medium | high | document | 302 | Oncologist + advocate (blocking) |

**Acceptance criteria — key tasks**

- **ewing-elig-pf-write-302** — every explainer education only; sourced to vetted bodies with
  citation; carries not-advice + not-eligibility-determination labels; passes reading-level check;
  no individualized advice or recommendation.
- **ewing-elig-pf-review-303** — **both** oncologist and patient advocate sign off with
  name/credential/date recorded; nothing ships without both; partner committed to last-mile +
  runs the usefulness survey (O6).

**Definition of Done (M3):** every patient-facing artifact signed off by oncologist **and**
advocate (recorded), sourced, labeled, reading-level-checked, and handed to the partner workflow.
If the panel/partner are not secured, M3 does not start and M0–M2 stand as the delivered goods.

---

## Milestone M4 — Sustainability & handoff

| ID | Title | Type | Size | Risk | Deliverable | Depends on | Reviewer |
|---|---|---|---|---|---|---|---|
| ewing-elig-ops-401 | Scheduled refresh + release automation + changelog | maintenance | medium | medium | pr | M2 | Maintainer |
| ewing-elig-correct-402 | Public correction/appeals path + triage workflow | code | small | medium | pr | M2 | Maintainer |
| ewing-elig-outcomes-403 | Outcomes loop reporting (O5/O7/O8; O6 post-M3) + partner handoff | maintenance | medium | medium | document | partner secured | Steward |

**Definition of Done (M4):** refresh + release automated; correction path live; outcomes loop
reporting adoption, freshness, and zero-harm; steward + maintainer named; partner handoff complete.

---

## Backlog / future (sized, unscheduled)

- ewing-elig-eu-translate (large, **high**) — hand off vetted explainers to `ewing-info-translations`.
- ewing-elig-snomed-map (medium, medium) — fuller SNOMED CT secondary mapping for licensed consumers.
- ewing-elig-benchmark (medium, medium) — publish an open eligibility-structuring benchmark from the
  gold standard (methodology aligned with the Chia corpus).
- ewing-elig-finder-handoff (medium, **high**) — provide structured data to `ewing-trial-finder`
  (which owns matching; we never assert recruiting status as guidance).
- ewing-elig-adjacent (large, medium) — extend the model to other pediatric sarcomas once proven.

---

## Generated task index

Every milestone-table row and every sized backlog row above now has a corresponding
schema-valid `tasks/<id>.json` (validated against `packages/schema/src/schemas.ts`). The full,
per-task acceptance criteria live in those JSON files; the "Acceptance criteria — key tasks"
blocks above remain the human-readable summary for the most important rows. All tasks carry
`status: open`, `verifiedNeed: false`, and `requestor: "TO BE SECURED"`; code/PR tasks use
`outputLicense: Apache-2.0`, data/doc/spec tasks use `CC-BY-4.0`; all patient-facing (M3 + two
backlog) tasks carry `riskTier: high`.

**Fan-out note (honest, bounded):** no row was fanned out into per-item tasks. The plan sizes the
trial sets by count (gold = 5 trials, pilot = 25 trials, M2 = full open set) but does **not**
enumerate specific trial IDs, and names no concrete language set (translation is the separate
HIGH-risk `ewing-info-translations` project, out of scope in v1). Per the fan-out policy, these
remain single representative tasks; per-trial / per-language items expand only on partner + scope
confirmation. No languages, datasets, trials, or beneficiaries were fabricated.

**Guardrail note:** no task authors refused content. The M3 patient-facing tasks
(`ewing-elig-pf-spec-301`, `-pf-write-302`, `-pf-review-303`) and the two patient-facing backlog
tasks (`ewing-elig-eu-translate`, `ewing-elig-finder-handoff`) are **education-only / structuring
/ handoff** tasks that never produce medical advice, prognosis, eligibility determinations, or
trial rankings. Each preserves the binding HIGH-risk blocking gate (credentialed pediatric
oncologist **and** patient advocate sign-off, recorded by name/credential/date) verbatim in its
`context`/`acceptanceCriteria`, and does not start until a partner + expert panel are secured.

Generated ids (27 total; `ewing-elig-schema-001` is the pre-existing seed):

- **M0:** `ewing-elig-schema-001` (seed), `ewing-elig-guard-002`, `ewing-elig-license-003`,
  `ewing-elig-gold-004`, `ewing-elig-ci-005`
- **M1:** `ewing-elig-ingest-101`, `ewing-elig-rules-102`, `ewing-elig-llm-103`,
  `ewing-elig-code-104`, `ewing-elig-pilot-105`, `ewing-elig-regress-106`
- **M2:** `ewing-elig-multireg-201`, `ewing-elig-scale-202`, `ewing-elig-audit-203`,
  `ewing-elig-fresh-204`, `ewing-elig-fhir-205`
- **M3 (HIGH risk, conditional):** `ewing-elig-pf-spec-301`, `ewing-elig-pf-write-302`,
  `ewing-elig-pf-review-303`
- **M4:** `ewing-elig-ops-401`, `ewing-elig-correct-402`, `ewing-elig-outcomes-403`
- **Backlog/future:** `ewing-elig-eu-translate`, `ewing-elig-snomed-map`, `ewing-elig-benchmark`,
  `ewing-elig-finder-handoff`, `ewing-elig-adjacent`

> Note on `ewing-elig-llm-103` lane: building the LLM-assist adapter is donated code work, so the
> task is `lane: donated`. The PLAN's funded/budget-capped behavior is a runtime property of the
> adapter (it runs only via `packages/runner` under a hard per-trial cap) and is captured in the
> task's acceptance criteria; no `fundedBudgetUsd` cap is asserted because the plan's Open
> Question 5 leaves the per-trial cap undecided (not fabricated here).

---

## Example task JSON (first M0 task)

Schema-valid against `packages/schema/src/schemas.ts` (donated lane, so no `fundedBudgetUsd`).

```json
{
  "id": "ewing-elig-schema-001",
  "title": "Design eligibility data model v0.1 (JSON Schema, FHIR-aligned)",
  "project": "ewing-eligibility-structuring",
  "type": "design-spec",
  "lane": "donated",
  "priority": "high",
  "domain": ["oncology", "clinical-trials", "ewing-sarcoma", "informatics", "fhir"],
  "riskTier": "medium",
  "urgent": false,
  "deliverable": "document",
  "tokenEstimate": "medium",
  "status": "open",
  "context": "Ewing sarcoma trial eligibility lives as dense free-text inclusion/exclusion blocks across registries (ClinicalTrials.gov, ICTRP, CTIS, ISRCTN). To structure it safely we first need an open, versioned data model that represents each criterion as a coded, comparable, fully-provenanced assertion. Cancer guardrails are binding: open-access data only, no patient/identifiable data, provenance on every assertion, and the model must never function as an eligibility determination or medical advice.",
  "objective": "Produce eligibility data model v0.1 as a draft-07 JSON Schema (FHIR ResearchStudy/eligibility-aligned, OMOP-mappable) that makes provenance and a not-advice/not-eligibility disclaimer REQUIRED, and supports concept coding, operators, values/units, temporality, negation, ambiguity flags, and age-appropriate performance status.",
  "acceptanceCriteria": [
    "Schema validates with AJV (draft-07), styled consistently with packages/schema.",
    "Models concept {system,code,display}, operator, value, unit, temporality, negation, ageBandYears, performanceStatus {scale: Lansky|Karnofsky|ECOG}, ambiguityFlag, confidence, extraction.method.",
    "Provenance block is REQUIRED: trialId, registry, registryVersion, retrievedAt, sourceText, sourceTextSha256, reviewer, status.",
    "A fixed not-advice + not-eligibility-determination disclaimer field is REQUIRED on every record.",
    "Includes documented FHIR ResearchStudy/eligibility alignment and an OMOP-CDM mapping note.",
    "Contains no patient-level or contact-PII fields; example records use synthetic/redacted data only."
  ],
  "resources": [
    "C:/code/elyos/packages/schema/src/schemas.ts",
    "ClinicalTrials.gov API v2 documentation",
    "HL7 FHIR ResearchStudy / EBM-on-FHIR eligibility profiles",
    "OHDSI OMOP CDM specification",
    "NCI Thesaurus (NCIt), HGNC, RxNorm, LOINC, ICD-O-3, HPO"
  ],
  "output": "A draft-07 JSON Schema file (eligibility-criterion.schema.json) plus a short spec document describing fields, FHIR alignment, OMOP mapping, and the required provenance + disclaimer model.",
  "requestor": "TO BE SECURED",
  "verifiedNeed": false,
  "outputLicense": "CC-BY-4.0"
}
```
