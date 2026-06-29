# ewing-eligibility-structuring — PLAN

> Status: Draft · Version: 0.1.0 · Last updated: 2026-06-28 · Owner: TBD (maintainer) ·
> Lane: donated (LLM-assisted extraction may use the **funded** lane with a hard per-trial
> budget cap — see §6/§7) · Track: 8a (Ewing's Sarcoma) · Roadmap risk: **medium**
> (data/structuring core), with **high** for any patient-facing surface.

> **Read this first.** This project structures the *eligibility criteria* of Ewing sarcoma
> clinical trials into a machine-readable, fully-provenanced form. It is built for families
> facing a rare, aggressive childhood cancer and the clinicians and advocates who help them.
> A wrong "you qualify" / "you don't qualify" can cost a family a trial slot, or send them
> chasing one they can never enter. We therefore treat correctness and humility as safety
> properties, not nice-to-haves. **The structured data is a research artifact; it is never an
> eligibility determination and never medical advice.** Every patient-facing artifact is HIGH
> risk and is blocked from shipping until a credentialed pediatric oncologist *and* a patient
> advocate have signed off (§7).

---

## Executive summary

Ewing sarcoma is a rare bone/soft-tissue cancer that strikes mostly children, adolescents, and
young adults (AYA), driven in ~85% of cases by the *EWSR1–FLI1* fusion (and related *EWSR1–ETS*
fusions). For relapsed or refractory disease, clinical trials are often the most meaningful
path — but the trials are scattered across registries (ClinicalTrials.gov, EU CTIS, WHO ICTRP,
ISRCTN), and their *eligibility criteria* are buried in dense, free-text inclusion/exclusion
blocks written for clinicians, not families. The result: families and even general clinicians
struggle to tell which trials a child could plausibly be a candidate for, and trial-matching
tools have nothing machine-readable to work with for this rare disease.

This project produces three open public goods, in strict risk order:

1. **An open eligibility data model** (`design-spec`, CC-BY-4.0) — a FHIR-aligned, OMOP-mappable
   JSON Schema that represents an eligibility criterion as a coded, comparable, provenanced
   assertion (concept + operator + value + unit + temporality + source span).
2. **An open structured dataset** (`dataset`, CC-BY-4.0 for our annotations) of Ewing /
   pediatric-sarcoma trial eligibility, with **provenance on every single assertion** (registry,
   trial ID, registry version, retrieval timestamp, verbatim source text, source-text hash,
   extraction method, and human reviewer).
3. **A reproducible structuring pipeline** (`code`, Apache-2.0) — ingestion → extraction
   (rule + optional budget-capped LLM assist) → terminology coding → schema validation →
   human-review queue → versioned release.

A **patient-facing plain-language layer** (explaining, in education-only terms, *what a trial's
criteria mean*) is in scope **only as a HIGH-risk, expert-gated phase (M3)** and ships nothing
without oncologist + patient-advocate sign-off.

**Honest status:** no partner organisation or beneficiary requestor is secured yet
(`verifiedNeed = false` on all tasks; partner = **TO BE SECURED**). The plan front-loads the
cancer guardrails, the licensing/provenance framework, and a double-reviewed gold-standard
before any scale-up.

---

## Problem & beneficiaries

**Who is helped (in priority order):**

- **Families of children/AYA with Ewing sarcoma**, especially relapsed/refractory, who are
  trying to understand whether a trial is even plausibly relevant before raising it with their
  care team. They are not the operators of the data — they are the people the work must protect.
- **Treating clinicians and tumour-board coordinators** at centres without a dedicated
  rare-sarcoma trials office, who need a faster way to scan eligibility across trials.
- **Patient advocates and navigators** (e.g. rare-cancer foundations) who help families prepare
  questions for their oncologist.
- **Researchers and trial-matching tool builders** who currently have no open, machine-readable
  eligibility dataset for this rare disease to build on or benchmark against.

**The verified need:** **TO BE SECURED.** The need is strongly *plausible* (rare-disease trial
discovery is a well-documented pain point) but we will not claim a *verified* need until a
partner — a pediatric-oncology cooperative group, a sarcoma foundation, or a trial-navigation
nonprofit — confirms the gap and commits to the last-mile handoff. Candidate partners to
approach (none contacted, none committed): Children's Oncology Group (COG), Alex's Lemonade
Stand Foundation, Sarcoma Foundation of America, QuadW / Ewing-focused foundations, St.
Baldrick's Foundation. **No work that asserts anything to a real family ships until a partner
and an expert review panel are in place.**

**Why machine-readable matters:** free-text eligibility cannot be searched, compared, or safely
surfaced. A structured, coded, provenanced representation lets a clinician filter
("EWSR1-rearranged, relapsed, age 8, adequate organ function, measurable disease") and lets a
navigator explain a criterion in plain language *with a citation back to the exact source text*.

---

## Goals and non-goals

**Goals**

- G1. Publish an open, versioned **eligibility data model** that is FHIR-aligned (HL7 FHIR
  `ResearchStudy` / eligibility, Vulcan/EBM-on-FHIR concepts) and OMOP-CDM mappable, so others
  can adopt it.
- G2. Publish an open, **fully-provenanced structured dataset** of eligibility criteria for the
  open-registry Ewing / pediatric-sarcoma trial set, with a documented coverage scope.
- G3. Ship a **reproducible pipeline** that turns registry free-text into validated structured
  records, with humans in the loop and a gold-standard regression gate.
- G4. Reach a measurable **structuring accuracy** bar against a double-reviewed gold standard
  before any dataset release (see §4).
- G5. **Only if a partner + expert panel are secured**, produce education-only, plain-language
  explainers of eligibility concepts, expert-gated (HIGH risk).

**Non-goals (explicit)**

- N1. **No eligibility determination.** The project never tells a person (or a clinician about a
  specific person) that they *are* or *are not* eligible for a trial. It structures criteria; it
  does not apply them to an individual.
- N2. **No medical advice**, treatment recommendation, prognosis, or trial recommendation.
- N3. **No controlled-access or identifiable data** — no dbGaP/EGA, no individual-level biobank,
  no patient records, no genomic individual data. Registry data only (which is about *trials*,
  not patients).
- N4. **Not a live trial-matching service or trial finder.** Recruiting status is volatile;
  surfacing "this trial is open near you" as a recommendation is out of scope (that lives in the
  separate, also-HIGH-risk `ewing-trial-finder` project). We link to the authoritative registry
  for current status; we do not assert it.
- N5. **No scraping behind paywalls or terms-of-use violations**; no use of sponsor-copyrighted
  protocol documents that are not openly licensed.
- N6. Not a general oncology eligibility tool in v1 — scope is Ewing-centric with adjacent
  pediatric-sarcoma trials, expanding only after the model is proven.

---

## Success metrics (outcomes)

Outcome-based, beneficiary-centric. Vanity counts ("criteria structured") are tracked only as
inputs, never as success.

| # | Outcome metric | Baseline | Target | How measured |
|---|---|---|---|---|
| O1 | **Structuring accuracy** vs double-reviewed gold standard (per-assertion F1: concept + operator + value) | none | ≥ 0.85 F1 before first public release; ≥ 0.90 by M2 | Held-out gold set; CI regression gate |
| O2 | **Provenance completeness** — assertions with full source lineage | 0% | **100%** (hard gate; no assertion ships without it) | Schema validation |
| O3 | **Inter-annotator agreement** on gold standard | none | Cohen's κ ≥ 0.75 (substantial) | Two independent annotators + adjudication |
| O4 | **Expert-confirmed correctness** — sampled records reviewed by clinical informatics/oncology | 0% | ≥ 95% of a random audit sample judged correct | Quarterly expert audit |
| O5 | **Adoption / handoff** — dataset or schema adopted by ≥ 1 partner (foundation, tool, or research group) | 0 | ≥ 1 committed partner using it for a real beneficiary workflow | Signed partner confirmation |
| O6 | **Beneficiary-reported usefulness** (only after patient-facing M3) — advocates/clinicians report it helped them prepare questions/identify candidate trials faster | none | ≥ 70% positive in a small structured survey (n≥10) | Partner-run survey |
| O7 | **Freshness / safety** — % of "recruiting" assertions re-verified against source within SLA | n/a | 100% within 30-day refresh; staleness flagged automatically | Pipeline refresh job |
| O8 | **Zero harm events** — confirmed cases of a family acting on a wrong structured claim | 0 | **0** (any event triggers incident review + correction) | Correction/appeals log |

Inputs tracked but **not** success: # trials ingested, # criteria structured, # PRs merged.

---

## Scope

**In scope**

- Eligibility (inclusion/exclusion) criteria of clinical trials whose **condition** includes
  Ewing sarcoma / Ewing family of tumours, plus adjacent **pediatric/AYA sarcoma** trials that
  enrol Ewing patients, **only from open registries**.
- The data model, the structured dataset (our annotations), and the pipeline code.
- Terminology coding to **open** vocabularies first (NCI Thesaurus / NCIt, HGNC, RxNorm, LOINC,
  ICD-O-3, HPO), with SNOMED CT only as a *mapped* secondary code where the consumer is licensed.
- Reproducibility, versioned releases with changelog + Zenodo DOI, and a public coverage ledger.
- **Conditionally (M3, HIGH risk, partner + expert panel required):** education-only,
  plain-language explainers of what eligibility concepts mean, sourced to vetted bodies.

**Out of scope**

- Individual eligibility determination, advice, recommendations, prognosis (see N1–N2).
- Controlled-access / identifiable / individual-level data of any kind (see N3).
- Live trial matching / "open near me" recommendations / recruiting-status assertions as
  guidance (see N4).
- Sponsor-copyrighted full protocol PDFs that are not openly licensed.
- Languages other than English in v1 (translation is the separate HIGH-risk
  `ewing-info-translations` project).
- Non-sarcoma cancers (v1).

---

## Solution approach & architecture

A staged, human-in-the-loop pipeline. The agent-neutral core stays vendor-neutral per
`CLAUDE.md`; any LLM-assisted extraction lives behind an adapter and, when run with budget,
goes through `packages/runner` on the funded lane with a hard per-trial cap.

**Pipeline stages**

1. **Ingest.** Pull trial records via the **ClinicalTrials.gov API v2** (primary), plus
   **WHO ICTRP**, **EU CTIS/EudraCT**, and **ISRCTN** where openly accessible. Query by
   condition (Ewing sarcoma / PNET / Askin / Ewing family of tumours) and by gene
   (*EWSR1*) for fusion-defined trials. Store the raw record, registry, **registry version**,
   and **retrievedAt** timestamp. Respect rate limits and terms of use.
2. **Segment.** Split the eligibility free-text into atomic inclusion/exclusion *statements*,
   preserving the **verbatim source span** and a **SHA-256 hash** of it for drift detection.
3. **Extract.** Convert each statement into structured assertion(s): concept, operator
   (`=, ≥, ≤, in, exists, absent, within`), value, unit, temporal context, and applicability
   (age band, disease state). Two cooperating extractors:
   - **Rule/grammar extractor** (deterministic, preferred) for common patterns (age, ECOG/
     Lansky/Karnofsky, lab thresholds, washout windows, prior-therapy negations).
   - **LLM-assisted extractor** (adapter; funded lane, budget-capped) for harder prose — output
     is **always** routed to the human-review queue and never auto-merged.
4. **Code.** Map concepts to **open vocabularies first** (NCIt, HGNC, RxNorm, LOINC, ICD-O-3,
   HPO); attach SNOMED CT only as a secondary mapping. Performance status is coded to the
   age-appropriate scale (**Lansky** for <16, **Karnofsky/ECOG** for ≥16) with the scale recorded.
5. **Validate.** AJV/JSON-Schema validation against the eligibility model; terminology validation
   (codes exist in the named version); a **gold-standard regression test** in CI.
6. **Review.** Human review queue with statuses (`extracted → reviewed → adjudicated`). Ambiguous
   or under-specified criteria (e.g. "per protocol", references to a non-public document) are
   flagged `human-must-resolve` and **excluded from release** until resolved or marked unresolvable.
7. **Release.** Versioned dataset + schema with changelog, coverage ledger, and Zenodo DOI.
   Every record carries the **"structured representation, not an eligibility determination; not
   medical advice"** disclaimer.

**Tech stack:** TypeScript + ESM, pnpm workspaces. JSON Schema (draft-07, consistent with
`packages/schema`). AJV for validation. Node for the pipeline. Optional LLM adapter behind the
provider interface (Claude via the funded `packages/runner`, budget-capped; see §14). Outputs as
JSON + JSON-Lines + a FHIR `ResearchStudy`/eligibility profile export; an OMOP-CDM mapping doc.

**Data model (eligibility criterion — abridged):**

```
EligibilityCriterion {
  criterionId            // stable id
  trialId, registry      // e.g. NCT… / EUCT… / ISRCTN…
  registryVersion        // record version / last-updated as published
  retrievedAt            // ISO timestamp
  type                   // "inclusion" | "exclusion"
  sourceText             // verbatim span
  sourceTextSha256       // hash of the span (drift detection)
  concept { system, code, display }   // NCIt/HGNC/RxNorm/LOINC/ICD-O-3/HPO
  operator               // = ≥ ≤ in exists absent within
  value, unit            // typed
  temporal { window, anchor }         // e.g. "within 28 days of enrolment"
  appliesTo { ageBandYears, diseaseState }
  performanceStatus { scale, min }    // Lansky | Karnofsky | ECOG
  negation               // boolean (e.g. "no prior X")
  ambiguityFlag          // boolean + reason
  confidence             // 0..1
  extraction { method: manual|rule|llm-assisted, model?, runId? }
  provenance { extractor, reviewer, reviewedAt, status }
  disclaimer             // fixed not-advice string
}
```

**Key decisions:** (a) open vocabularies first to avoid SNOMED licensing friction; (b)
deterministic rule extraction preferred, LLM only with mandatory review; (c) provenance is a
schema-enforced *requirement*, not an option; (d) ambiguous criteria are withheld, never guessed.

---

## Data, licensing & compliance

> **CANCER GUARDRAILS LEAD HERE (binding).**
> - **ONLY open-access / aggregate / de-identified data.** Clinical-trial *registry* records are
>   about trials, not patients; we use them only from open registries. **Controlled-access
>   (dbGaP, EGA), individual-level biobanks, and ANY identifiable patient data are OUT OF SCOPE.**
> - **Per-source license verification is mandatory before ingestion.** No source enters the
>   pipeline until its license/terms are recorded in the License Register and confirmed to permit
>   reuse and derivatives. If terms are unclear, the source is excluded.
> - **Provenance on every assertion.** No structured assertion is released without full lineage
>   (registry, trial ID, registry version, retrievedAt, verbatim source span, source-text hash,
>   extraction method, human reviewer). This is a schema-enforced hard gate (O2 = 100%).
> - **No medical advice.** Every record and release carries: *"Structured representation of
>   published trial criteria for research/education — informational, not medical advice; not an
>   eligibility determination; consult your care team."*

**Sources & licenses (License Register — to be completed and verified per source):**

| Source | Use | License / terms stance |
|---|---|---|
| ClinicalTrials.gov API v2 | Primary trial + eligibility text | US-Gov work, broadly public; **but** linked/sponsor content may be copyrighted — store verbatim spans, record attribution, exclude any non-open protocol PDFs. Verify current Terms of Use. |
| WHO ICTRP | Cross-registry coverage | Verify ICTRP data-use terms (attribution/redistribution constraints) before redistribution. |
| EU CTIS / EudraCT | EU trials | Verify CTIS public-data terms; some fields restricted — open fields only. |
| ISRCTN | UK/intl trials | Content commonly CC-BY — verify per record. |
| NCI Thesaurus (NCIt) | Primary oncology terminology | Open / freely usable — preferred to avoid SNOMED licensing. Verify version + attribution. |
| HGNC, RxNorm, LOINC, ICD-O-3, HPO | Concept coding | HGNC/RxNorm public-domain-ish (NLM); LOINC + ICD-O-3 require accepting a (free) license; record per-vocabulary terms. |
| SNOMED CT | Secondary mapping only | Affiliate-license restricted; included only as a *mapping* for already-licensed consumers; not redistributed where prohibited. |
| Chia corpus (eligibility annotation reference) | Methodology reference only (not Ewing data) | CC-BY-4.0 — cite if methods reused. |

**Output licensing:** code **Apache-2.0** (or MIT per Elyos default); schema + dataset
annotations **CC-BY-4.0**; verbatim source spans remain under their source terms with attribution.
We license *our annotations and structure*, not the underlying sources.

**Privacy / PII stance:** registry records can contain **personal data of investigators/contacts**
(names, emails, phone). We **do not ingest or republish contact PII**; such fields are dropped at
ingest. No patient-level data is ever touched. A redaction check runs in the pipeline and in CI.

**Attribution:** each record attributes its source registry + trial; releases include a
`SOURCES.md` and a machine-readable provenance file; Zenodo DOI for citation.

---

## Quality, review & risk gates

> **CANCER GUARDRAILS LEAD HERE (binding).** Risk tier is **medium for the data/structuring
> core** and **high for any patient-facing artifact**. For the HIGH-risk patient-facing layer,
> **a credentialed pediatric oncologist AND a patient advocate must both sign off, and that
> sign-off is an explicit BLOCKING gate — nothing patient-facing ships without both.** No
> exceptions, no "temporary" releases.

**Risk tiers in this project**

- **Medium** — schema, pipeline code, and the structured *dataset* (research/data deliverable
  consumed by clinicians/researchers/tools, not asserted to a family). Requires clinical-
  informatics reviewer accuracy review + gold-standard regression.
- **High** — *any* patient-facing surface: plain-language explainers, any artifact a family
  reads, any framing that could read as advice or as an eligibility determination.

**Review gates (in order; each is blocking for its tier):**

1. **Schema/code review** — maintainer + 1 engineer; schema validates, CI green.
2. **Provenance gate** — automated: 100% of assertions have full lineage + hash; else release blocked.
3. **Accuracy gate (medium)** — clinical-informatics reviewer; F1 ≥ target vs gold standard;
   κ ≥ 0.75 on gold set; random audit ≥ 95% correct.
4. **Ambiguity gate** — every `human-must-resolve` item resolved or excluded; no guessed criteria.
5. **HIGH-risk patient-facing gate (BLOCKING):** **credentialed pediatric oncologist sign-off
   AND patient-advocate sign-off**, both recorded with name/credential/date, *before any
   patient-facing deed ships*. Content must (a) be education only, (b) be sourced to vetted bodies
   (NCI / COG / ESMO / CCLG), (c) carry the not-advice + not-eligibility-determination label, and
   (d) pass a plain-language/reading-level check.
6. **Refusal check** — agent applies `CLAUDE.md` refusal guardrails; any high-stakes-advice drift,
   privacy issue, or for-profit-primary-benefit concern stops the task and is surfaced.

**Definition of Shipped (this project):** acceptance criteria met **+** CI green (schema +
gold-standard regression) **+** provenance 100% **+** accuracy gate passed **+** (for patient-facing)
oncologist + advocate sign-off recorded **+** the artifact actually handed to / adopted by the
partner or beneficiary workflow. *Delivered, not merged.*

---

## Roadmap & milestones

| Milestone | Goal | Exit criteria (measurable) |
|---|---|---|
| **M0 — Cold-start foundation** | Schema v0.1, guardrails + License Register, gold standard | Eligibility schema v0.1 validates in CI; License Register complete + every M0 source verified open; **5 trials manually structured and independently double-reviewed** (κ ≥ 0.75); guardrail + not-advice + provenance policy ratified; PII-redaction check in CI. |
| **M1 — Pilot pipeline (25 trials)** | Build ingest→extract→code→validate→review pipeline; pilot dataset | Pipeline reproducible from clean clone; 25-trial pilot released with **100% provenance**; F1 ≥ 0.85 vs gold; gold-standard regression test wired into CI; ambiguity queue working; Zenodo DOI minted. |
| **M2 — Scale & coding** | Full open Ewing/pediatric-sarcoma set; terminology coding; QA | Coverage = the agreed open-registry Ewing/adjacent set; F1 ≥ 0.90; random audit ≥ 95% correct; SNOMED secondary mappings where licensed; freshness/refresh job (O7) live; coverage ledger published. |
| **M3 — Patient-facing layer (HIGH risk; conditional)** | Education-only plain-language explainers — *only if partner + expert panel secured* | **Every** explainer signed off by oncologist **and** advocate (recorded); sourced to NCI/COG/ESMO/CCLG; not-advice + not-eligibility labels present; reading-level check passed; partner committed to last-mile + outcome survey (O6). |
| **M4 — Sustainability & handoff** | Refresh automation, partner handoff, outcome tracking | Scheduled refresh + staleness flagging operating; maintainer + steward named; outcomes loop (O5/O6/O8) reporting; correction/appeals path live. |

M3 does not start until a partner and the expert review panel exist; if they are never secured,
the project still delivers M0–M2 (schema + dataset + pipeline) as standalone research goods.

---

## Work breakdown

The itemized, schema-mapped backlog lives in **`TASKS.md`** — one section per milestone (M0–M4),
each with a task table (`ID | Title | Type | Size | Risk | Deliverable | Depends on | Reviewer`),
acceptance criteria for the most important tasks, a milestone Definition of Done, a backlog, and a
complete schema-valid example Task JSON for the first M0 task. Tasks are sized `small/medium/large`
and map to Elyos task `type` (`code | research | writing | data | design-spec | maintenance`) and
`deliverable` (`pr | dataset | document`). All tasks carry `verifiedNeed = false` until a partner
is secured; all **patient-facing tasks carry `riskTier = high`**.

---

## Governance, roles & stakeholders

- **Maintainer (Owner):** TBD — owns schema, pipeline, releases, and the guardrail policy.
- **Engineering reviewers:** ≥ 1 rotating reviewer for code/schema PRs.
- **Clinical-informatics reviewer:** reviews structuring accuracy (medium-risk gate). TO BE SECURED.
- **Expert review panel (HIGH-risk gate):** a **credentialed pediatric oncologist** + a
  **patient advocate** (ideally with lived Ewing experience). Both are **blocking** sign-offs for
  any patient-facing deed. TO BE SECURED.
- **Steward (last-mile owner):** ensures the dataset/explainers actually reach the partner/
  beneficiary workflow and that outcomes are tracked. TO BE SECURED.
- **Partner / requestor:** a pediatric-oncology group, sarcoma foundation, or trial-navigation
  nonprofit. **TO BE SECURED** (`verifiedNeed = false` until then).
- **Conflict-of-interest / veto:** per Elyos governance — no sponsor/for-profit may primarily
  benefit; COI checklist applied to reviewers and partners.

---

## Dependencies & integrations

- **External data:** ClinicalTrials.gov API v2; WHO ICTRP; EU CTIS/EudraCT; ISRCTN (open fields).
- **Terminologies:** NCIt, HGNC, RxNorm, LOINC, ICD-O-3, HPO (open-first); SNOMED CT (secondary,
  licensed consumers only).
- **Standards:** HL7 FHIR (`ResearchStudy`/eligibility; Vulcan/EBM-on-FHIR); OMOP CDM mapping.
- **Methodology reference:** Chia eligibility corpus (CC-BY-4.0) — methods only, not Ewing data.
- **Elyos pieces:** `packages/schema` (Task schema/conventions), `packages/cli` (workspace + PRs,
  donated lane), `packages/runner` (funded lane, budget-capped LLM extraction), Zenodo (DOI/release).
- **Hosting:** Git repo under the Elyos org; releases via tags + Zenodo.

---

## Risks & mitigations

| Risk | Likelihood | Impact | Mitigation | Owner |
|---|---|---|---|---|
| A family acts on a wrong/outdated structured criterion (harm) | Medium | **High** | Not-advice + not-eligibility labels on every record; no individual determination (N1); patient-facing HIGH-risk + oncologist+advocate blocking gate; freshness SLA; correction/appeals path; incident review on any event (O8) | Expert panel / Maintainer |
| LLM extraction hallucinates or mis-structures a criterion | Medium | High | Deterministic rules preferred; LLM output never auto-merged; mandatory human review; gold-standard regression in CI; ambiguity items withheld | Maintainer |
| Source license misread → republishing restricted content | Medium | High | License Register + per-source verification before ingest; license code Apache-2.0 / data CC-BY-4.0 for *our* annotations only; verbatim spans kept under source terms; exclude unclear sources | Maintainer |
| Contact/investigator PII leaks into dataset | Medium | High | Drop contact fields at ingest; redaction check in pipeline + CI | Maintainer |
| Upstream registry text changes (drift) makes records stale/wrong | High | Medium | Source-text SHA-256 + registry version stored; refresh job re-verifies; staleness flagged; recruiting status never asserted, only linked | Maintainer |
| No partner / no expert reviewers secured | High | Medium | M0–M2 deliver standalone; M3 (patient-facing) gated and deferred until secured; `verifiedNeed=false` until then | Owner |
| For-profit captures the dataset for primary commercial benefit | Low | Medium | Public-benefit framing; CC-BY (attribution); COI/veto checklist; refusal guardrail | Governance |
| Cross-registry duplicates inflate/contradict coverage | Medium | Medium | Dedup by cross-registration IDs; coverage ledger | Maintainer |
| Funded-lane LLM cost overrun | Low | Medium | Hard per-trial budget cap in `packages/runner`; donated lane default | Maintainer |

---

## Security & privacy

- **Threat surface:** public registry APIs (read-only), an extraction pipeline, optional funded
  LLM calls, and published artifacts. No patient data, no auth-gated data, no user accounts in v1.
- **Secrets:** any registry/Zenodo/LLM API keys via environment/secret store only — **never** in
  logs, receipts, commits, or the dataset (per `CLAUDE.md`). Funded runs use `packages/runner`
  with a budget cap and redacted logs.
- **PII:** investigator/contact fields dropped at ingest; redaction enforced in pipeline + CI;
  zero patient-level data by construction (N3).
- **Abuse/misuse prevention:** the dataset is explicitly *not* an eligibility engine; labels and
  docs forbid presenting it as advice; refusal guardrails stop tasks that drift toward
  individualized advice, surveillance, deception, or for-profit primary benefit.
- **Integrity:** source-text hashing + signed-off DCO commits (`git commit -s`); checksummed,
  versioned releases.

---

## Sustainability & maintenance

- **Who maintains:** the named maintainer + reviewer rotation; refresh runs on a schedule (M4).
- **Freshness:** trials change status frequently; an automated refresh re-pulls sources, compares
  source-text hashes, flags drift/staleness, and never lets a stale "recruiting" claim stand as
  guidance (O7). Current status is always deferred to the live registry link.
- **Outcomes tracking:** the outcomes loop reports O5 (adoption), O6 (beneficiary usefulness,
  post-M3), O7 (freshness), O8 (zero-harm) on a public cadence.
- **Handoff:** steward ensures the partner can consume and re-publish updates; if maintenance
  lapses, the dataset is clearly marked "last verified <date>" so no one mistakes stale data for
  current.
- **Correction/appeals:** a public path for clinicians/advocates/families to report an error;
  triaged within the freshness SLA; corrections versioned.

---

## Open questions

1. **Partner & beneficiary:** which org confirms the need and owns the last mile? (Blocks
   `verifiedNeed=true` and all of M3.)
2. **Expert panel:** can we secure a credentialed pediatric oncologist *and* a patient advocate
   for the blocking HIGH-risk gate, and at what cadence?
3. **Registry terms:** do WHO ICTRP / EU CTIS terms permit redistribution of structured
   derivatives at the field-level we need? (License Register must resolve before ingest.)
4. **Vocabulary licensing:** is NCIt sufficient for our oncology concepts, or do we need
   SNOMED CT coverage that we cannot redistribute? Confirm LOINC/ICD-O-3 license acceptance.
5. **Funded lane:** is budget-capped LLM extraction justified vs. deterministic-only? What is the
   per-trial cap and the review-time cost?
6. **Scope boundary with `ewing-trial-finder`:** confirm we never assert recruiting status as
   guidance; finder owns matching, we own structure.
7. **Accuracy bar:** is F1 ≥ 0.85 (release) / 0.90 (M2) the right safety threshold for this domain,
   or should it be higher given the stakes?

---

## References

- Elyos `CLAUDE.md` (work rules, lanes, quality bar, refusal guardrails).
- Elyos `docs/good-deed-definition.md` (5 criteria + risk tiers).
- Elyos `packages/schema/src/schemas.ts` (Task JSON schema).
- Elyos `planning/ROADMAP.md`, Track 8 / 8a (cancer + Ewing guardrails).
- ClinicalTrials.gov API v2 documentation + Terms of Use.
- WHO ICTRP; EU CTIS/EudraCT; ISRCTN data-use terms.
- HL7 FHIR `ResearchStudy` / eligibility; Vulcan / EBM-on-FHIR; OHDSI OMOP CDM.
- NCI Thesaurus (NCIt); HGNC; RxNorm; LOINC; ICD-O-3; HPO; SNOMED CT license terms.
- Kury et al., "Chia, a large annotated corpus of clinical trial eligibility criteria,"
  *Scientific Data* (2020), CC-BY-4.0 (methodology reference).
- Vetted patient-facing bodies for M3 sourcing: NCI, Children's Oncology Group (COG), ESMO, CCLG.

---

## Appendix A — Improvements applied

The following 25 improvements were identified during review and are **already applied** in the
plan above (and reflected in `TASKS.md`).

1. **Split risk tiers explicitly** — data/structuring core = medium; *any* patient-facing surface
   = high — instead of a single project-wide tier. (§7, N1, Non-goals.)
2. **Made oncologist + patient-advocate sign-off an explicit BLOCKING gate** for every
   patient-facing deed, recorded with name/credential/date. (§7 gate 5.)
3. **Added "no eligibility determination" as the first non-goal (N1)** — the data structures
   criteria, never applies them to a person.
4. **Provenance is schema-enforced and 100% (O2)** — no assertion ships without full lineage.
5. **Added source-text SHA-256 + registry version + retrievedAt** for drift detection and
   reproducibility. (Data model, §6, drift risk.)
6. **Open vocabularies first (NCIt/HGNC/RxNorm/LOINC/ICD-O-3/HPO)**, SNOMED CT only as a
   non-redistributed secondary mapping — avoids affiliate-license friction. (§5/§6/§9.)
7. **Per-source License Register with verification-before-ingest**; unclear sources excluded. (§9.)
8. **Handled ClinicalTrials.gov sponsor-copyright caveat** — verbatim spans kept under source
   terms; no non-open protocol PDFs; we license only our annotations. (§9.)
9. **PII redaction of investigator/contact fields** at ingest, enforced in pipeline + CI. (§9/§14.)
10. **Funded-lane LLM extraction is budget-capped and never auto-merged**; deterministic rules
    preferred; all LLM output human-reviewed. (§6/§7/§14.)
11. **Negation + temporality modeled explicitly** (e.g. "no prior therapy within 28 days"). (Data model.)
12. **Ambiguity flag + `human-must-resolve` queue**; ambiguous/under-specified criteria are
    withheld from release, never guessed. (§6/§7 gate 4.)
13. **Age-appropriate performance-status handling** — Lansky (<16) vs Karnofsky/ECOG (≥16),
    scale recorded. (§6, data model.)
14. **Fusion/histology nuance** — EWSR1 rearrangement and histology as first-class eligibility
    variables and a query axis. (§2/§5/§6.)
15. **Cross-registry deduplication** by cross-registration IDs + a coverage ledger. (§6/risks.)
16. **FHIR alignment + OMOP mappability** baked in for downstream adoption. (§5/§6/§12.)
17. **Versioned releases with changelog + Zenodo DOI** for citability and correction tracking. (§6/§15.)
18. **Freshness SLA + staleness flagging (O7)**; recruiting status never asserted, only linked. (§15/N4.)
19. **Correction/appeals path** for clinicians/advocates/families to report errors. (§15.)
20. **Outcome-based success metrics with baselines/targets**, separating inputs (vanity) from
    outcomes; added a **zero-harm metric (O8)**. (§4.)
21. **Inter-annotator agreement (Cohen's κ ≥ 0.75) + double-reviewed gold standard** as an M0
    exit criterion and CI regression gate. (§4/§9/roadmap.)
22. **Conflict-of-interest / veto checklist** so no sponsor/for-profit primarily benefits. (§11/risks.)
23. **Scope boundary with `ewing-trial-finder`** clarified (we structure; finder matches) to avoid
    drifting into recommendations. (N4/open Q6.)
24. **Plain-language/reading-level check** for any patient-facing explainer, sourced to
    NCI/COG/ESMO/CCLG. (§7 gate 5/roadmap M3.)
25. **Secrets discipline** — keys via secret store only, redacted logs, never in dataset/receipts;
    DCO-signed commits + checksummed releases. (§14.)

---

## Review sign-off

**Reviewer:** senior staff engineer + TPM (self-review pass for completeness + correctness).
**Date:** 2026-06-28.

**Completeness check:** all 17 required PLAN sections present and in order; Appendix A (25 applied
improvements) present; TASKS.md authored with milestone tables, acceptance criteria, DoD, backlog,
and a schema-valid example Task JSON.

**Guardrail check (binding):**
- Data/licensing & compliance (§9) and Quality/review (§7) both **lead with the cancer
  guardrails** (open/aggregate/de-identified only; per-source license verification; provenance on
  every assertion; no medical advice). ✔
- Controlled-access (dbGaP/EGA) and identifiable patient data explicitly **out of scope** (N3). ✔
- Patient-facing content is **education only**, sourced to NCI/COG/ESMO/CCLG, labeled
  not-advice + not-eligibility, and **gated behind a credentialed oncologist + patient-advocate
  BLOCKING review** before any deed ships (riskTier high). ✔
- Provenance required on every assertion (O2 = 100%, schema-enforced). ✔

**Corrections applied during review:**
- Tightened N1 so the *non-goal* (no eligibility determination) is unambiguous and mirrored in the
  per-record disclaimer string.
- Made the HIGH-risk gate's sign-off recordable (name/credential/date) so it is auditable, not just
  asserted.
- Added the zero-harm outcome metric (O8) and an explicit incident-review trigger.
- Clarified that M0–M2 deliver standalone value even if no partner is ever secured, so the project
  is not blocked end-to-end by `verifiedNeed=false`.

**Open items requiring a human decision:** secure partner + beneficiary requestor; secure the
oncologist + patient-advocate expert panel; resolve WHO ICTRP / EU CTIS redistribution terms;
confirm the accuracy threshold is high enough for the stakes (Open Questions 1–7).

**Verdict:** Approved as Draft v0.1.0. Patient-facing work (M3) remains blocked until the expert
panel and partner are in place.
