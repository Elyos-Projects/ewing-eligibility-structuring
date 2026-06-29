# Competitive & Improvement Analysis — `ewing-eligibility-structuring`

> Analyst pass over `PLAN.md` v0.1.0 (2026-06-28) and `TASKS.md`. Web-grounded competitor
> review with cited sources. Scope reminder: the project structures *trial* eligibility
> criteria into machine-readable DATA. It is explicitly **not** a per-patient eligibility
> engine. The strongest risk to design against is downstream misuse of the structured data
> for automated eligibility verdicts.

---

## 1. Correctness & completeness review of PLAN.md

The plan is unusually mature: split risk tiers, schema-enforced provenance (O2 = 100%),
double-reviewed gold standard with Cohen's κ, ambiguity withholding, open-vocabulary-first
coding, and an explicit "no eligibility determination" non-goal (N1). These are the right
bones. The gaps below are real and concrete.

**1.1 The accuracy metric (O1) is under-specified and likely mis-calibrated.**
"Per-assertion F1 (concept + operator + value)" hides the hardest problems:
- **No segmentation/recall metric.** The literature's biggest failure mode is *missing* a
  criterion or splitting a compound criterion wrong, not mis-coding a found one. Rule-based
  systems like EliXR show "high precision but poor recall" (see §2). A single F1 over
  *found* assertions can be 0.90 while the pipeline silently drops 20% of criteria. **Add a
  span-level extraction recall metric and a segmentation/boundary-agreement metric** as
  separate gates, not folded into one F1.
- **No matching definition.** Is a concept "correct" on exact code, code-or-ancestor, or
  display-string? Operator/value/unit/temporal each need partial-credit rules. Without a
  published scoring script, "F1 ≥ 0.85" is not reproducible. **Specify the scoring harness
  (and ship it) — this is also what makes O1 a real CI gate.**
- **Threshold is asserted, not justified.** Open Q7 already flags this. Comparable systems
  report F1 ≈ 0.84–0.90 on *easier* high-volume diseases (Alzheimer's, NASH). Ewing trials
  are rare, prose-dense, and fusion/histology-laden — accuracy will likely be *lower* per
  trial, and a blanket 0.85 may be both unreachable on hard criteria and too low on safety-
  critical ones (age bounds, washout windows). **Stratify the target by criterion class**
  (e.g. ≥0.95 on age/lab thresholds; report-only on free-text "per protocol" prose).

**1.2 Inter-annotator agreement (O3) — Cohen's κ is the wrong statistic for this task.**
κ assumes a fixed set of items each labeled into categories. Eligibility structuring is a
*span-extraction + relation* task where annotators first have to agree on *what the units
are*. You cannot compute a clean κ when annotator A emits 6 assertions and annotator B emits
8 from the same paragraph. The Chia and Leaf corpora measure entity/relation agreement with
**F1-style pairwise agreement (and span/relation-level breakdowns)**, not κ. **Replace or
supplement κ with span-level and relation-level pairwise-F1 agreement**; keep κ only for the
genuinely categorical sub-decisions (type=inclusion/exclusion, negation present, ambiguity
flag). As written, O3 will be gamed or uncomputable.

**1.3 The boundary against per-patient determination is stated but not architecturally
enforced.** N1 and the disclaimer are policy, not mechanism. The schema emits exactly the
artifact (`concept + operator + value + unit + temporal + negation`) that a third party can
evaluate against a patient record in one line of code — that *is* an eligibility engine input
by construction. Mitigations the plan is missing:
- **A license clause / acceptable-use term** forbidding use for automated individual
  eligibility determination (CC-BY allows any use; add a companion AUP or a "responsible use"
  statement in the data card — note it is not legally enforceable but sets norms and supports
  takedown).
- **Deliberately withhold or down-rank the "auto-evaluable" packaging** where it isn't needed
  (e.g. don't ship a ready-made FHIR/CQL *executable* expression of patient-applying logic in
  v1; ship the descriptive structure + provenance, and document the gap on purpose). The plan
  currently lists a FHIR `ResearchStudy`/eligibility export (M2, task 205) and OMOP mapping
  without discussing that an OMOP/CQL expression is *precisely* a runnable cohort query — the
  same thing Criteria2Query/LeafAI produce to select patients. **This is a latent contradiction
  between N1/N4 and tasks 104/205 and the OMOP-mappable goal (G1).** Resolve explicitly: state
  that the dataset is OMOP-*mappable for research cohort analytics over de-identified data*,
  and that producing a patient-facing eligibility verdict from it is out of scope and against
  the AUP.
- **Misuse threat model is thin.** §13 abuse section is one paragraph. Add a concrete
  misuse-cases table (a startup wraps the dataset as a "do I qualify?" chatbot; a scraper
  strips disclaimers; stale "recruiting" inferred from structured fields). For each: the
  design control. This is the project's signature differentiator (§4) and deserves a section.

**1.4 LLM hallucination controls are good but incomplete.** "Never auto-merged + human review
+ gold regression" is correct. Missing:
- **No grounding/verification check on LLM output.** Require every LLM-emitted assertion's
  `sourceText` span to be a *verbatim substring* of the trial's eligibility text (string-match
  gate), and reject assertions whose value/unit/code cannot be traced to that span. This
  catches the most dangerous hallucination — an invented threshold — *before* it reaches a
  human, cheaply.
- **No measurement of reviewer automation bias.** Humans rubber-stamp plausible LLM output.
  Add a planted-error / blind-recheck protocol to keep the review gate honest (e.g. seed known
  errors; track reviewer catch rate).
- **Confidence field (0..1) has no defined source or calibration.** An LLM's self-reported
  confidence is not calibrated and must never gate release. State that `confidence` is
  advisory metadata only and is never used to skip human review.

**1.5 Gold standard is too small to support the headline metric.** M0 = 5 trials double-
reviewed; M1 releases 25 with "F1 ≥ 0.85 vs gold." A 5-trial gold set yields very few
assertions of each rare class (washout, fusion status, measurable disease) — the F1 confidence
interval will be enormous and the "gate" is statistically meaningless. **Define a minimum
gold-set size in assertions per criterion class**, and grow the gold set as a first-class
deliverable (the plan treats gold as a one-time M0 artifact; it should be versioned and
expanded each milestone). Also: gold standard built by 2 annotators + 1 adjudicator with no
named clinical oncologist in M0 — the clin-informatics reviewer is "TO BE SECURED," so the
gold standard's *clinical* correctness is currently unanchored.

**1.6 Temporal/relational logic is the known hard part and is under-modeled.** The data model
has `temporal {window, anchor}` and `negation` (boolean) — but eligibility criteria routinely
need: nested boolean logic (A AND (B OR C)), counts ("≥2 prior lines"), inter-criterion
relations, "unless"/exception clauses, and value ranges (not just a single operator+value).
Chia represents each criterion as a **directed acyclic graph** convertible to Boolean logic
for exactly this reason; a flat assertion row cannot express "no chemotherapy within 28 days
*unless* nitrosourea, then 42 days." **The model needs a logic/grouping layer (criterion
groups, AND/OR/NOT trees, exceptions) or it will mis-flatten compound criteria** — and silent
flattening of an exclusion is a safety bug, not a cosmetic one. This is the single biggest
*technical* correctness gap.

**1.7 Smaller but real gaps.**
- **`registryVersion` from ClinicalTrials.gov is ambiguous.** CTG v2 has `lastUpdatePostDate`
  and an internal version; pin which field and capture the historical-version API so a record
  can be reproduced exactly. Drift detection via SHA-256 is good but won't tell you *what*
  changed semantically — consider storing the prior span for diffing.
- **Freshness SLA (O7, 30-day) has no owner-capacity model.** Who runs the refresh and triages
  drift on a no-partner project? If unstaffed, stale data is the most likely real-world harm.
- **Coverage/denominator is undefined.** O-metrics measure quality of what's structured, but
  there's no "% of in-scope Ewing trials covered" metric, so a high-quality dataset covering
  30% of trials could look "done." Add a coverage-of-known-universe metric with the CTG query
  that defines the denominator.
- **No de-duplication of *criteria* across trials** (boilerplate organ-function blocks recur
  verbatim) — an opportunity (template mining) and a metric-inflation risk (same easy criterion
  counted many times).
- **EU CTIS / ICTRP redistribution terms unresolved (Open Q3)** are a *blocking* dependency for
  M2 task 201 but M2 is scheduled as if they're resolved; sequence the license resolution as a
  hard predecessor.

---

## 2. Competitive landscape

This is a well-trodden research area; the Ewing/pediatric angle and the misuse-resistant,
provenance-first framing are what's novel — not eligibility structuring itself. Direct and
adjacent prior art:

**OHDSI Criteria2Query (1.0 → 2.0 → 3.0).** The reference open tool. Hybrid ML + rule pipeline
that parses free-text inclusion/exclusion criteria into a structured representation and then
into **executable OMOP-CDM cohort queries** runnable in ATLAS; 2.0 added an editable human-in-
the-loop UI; 3.0 (2023) is powered by generative LLMs.
- Strengths: open-source, OMOP-native, end-to-end to an executable query, active OHDSI
  community, interactive correction.
- Weaknesses/gaps: built to *select patients/cohorts* (the very capability this project
  deliberately withholds); general-purpose, not pediatric-oncology tuned; provenance/audit
  trail per-assertion is not its focus; no Ewing/fusion/Lansky handling; produces a runnable
  cohort query (an eligibility engine), which is the misuse surface Elyos guards against.
- https://github.com/OHDSI/Criteria2Query ·
  https://www.ohdsi.org/wp-content/uploads/2023/10/423-Park-BriefReport.pdf ·
  https://www.researchgate.net/publication/331531291_Criteria2Query_A_natural_language_interface_to_clinical_databases_for_cohort_definition

**Chia corpus (Kury et al., *Scientific Data* 2020).** The closest "open dataset" prior art:
12,409 annotated criteria from 1,000 Phase-IV trials, 41,487 entities / 25,017 relations, each
criterion as a **DAG convertible to Boolean logic**. CC-BY-4.0.
- Strengths: large, public, benchmark-grade, models relations/logic explicitly (the §1.6 gap).
- Weaknesses/gaps: Phase-IV general trials (not pediatric oncology, not rare disease, almost no
  Ewing); a static annotation corpus, not a maintained/fresh dataset; no provenance-to-source-
  hash or freshness model; annotation only, no pipeline. The plan correctly cites it as a
  *methodology* reference — it should also be the IAA-metric template (§1.2).
- https://www.nature.com/articles/s41597-020-00620-0 · https://pmc.ncbi.nlm.nih.gov/articles/PMC7452886/

**EliIE / EliXR (Columbia).** EliIE: open-source IE system parsing criteria into OMOP-CDM v5
via entity recognition → negation → relation extraction → concept normalization (F1 ~0.84–0.90
on condition class; trained on 230 Alzheimer's trials). EliXR: earlier UMLS dictionary/rule
system, high precision/poor recall.
- Strengths: open, OMOP-aligned, clear 4-stage architecture this plan can borrow.
- Weaknesses/gaps: older (2017), disease-specific to Alzheimer's, recall limits, no provenance/
  freshness/misuse framing, not maintained.
- https://academic.oup.com/jamia/article/24/6/1062/3098256 · https://pmc.ncbi.nlm.nih.gov/articles/PMC6259668/

**Leaf Clinical Trials Corpus / LeafAI (UW).** LCT: >1,000 criteria with highly granular
structured labels (*Scientific Data* 2022). LeafAI: data-model-agnostic query generator with a
UMLS knowledge base and conditional reasoning; matched 43% of enrolled patients vs 27% for a
human programmer in minutes vs 26 hours.
- Strengths: rich annotation schema, strong reasoning, data-model-agnostic, open corpus.
- Weaknesses/gaps: explicitly a *cohort-discovery / patient-selection* engine (again, the
  withheld capability); general clinical, not pediatric sarcoma; no provenance-first/misuse
  guardrails.
- https://www.nature.com/articles/s41597-022-01521-0 · https://academic.oup.com/jamia/article-abstract/30/12/1954/7238441 · https://arxiv.org/abs/2304.06203

**TrialGPT (NIH/NLM, *Nature Communications* 2024).** Zero-shot LLM patient-to-trial matching:
retrieval → criterion-level eligibility prediction (87.3% accuracy, with explanations + sentence
localization) → trial ranking; cut screening time 42.6%.
- Strengths: state-of-the-art, NIH-backed, criterion-level explanations with source localization
  (a provenance idea worth borrowing), strong matching accuracy.
- Weaknesses/gaps: it *is* the patient-facing eligibility-determination system Elyos refuses to
  build — it outputs per-patient "meets/does not meet/unclear" verdicts; not an open dataset; no
  rare-disease/pediatric tuning; the explanation faithfulness is a research claim, not a safety
  guarantee. Useful as the canonical "what we are NOT" contrast.
- https://www.nature.com/articles/s41467-024-53081-z · https://www.ncbi.nlm.nih.gov/research/trialgpt/about/

**AutoCriteria (JAMIA 2024).** GPT-4-based generalizable extraction of granular criteria
(entities + values + temporality + modifiers, multi-arm logic) across 9 diseases; overall F1
89.42 with no manual annotation.
- Strengths: strong LLM extraction, generalizes across diseases incl. rare, handles multi-arm/
  logic — direct evidence the LLM-assist approach in M1 task 103 is viable.
- Weaknesses/gaps: proprietary GPT-4 dependency; no open dataset/corpus released; provenance-to-
  span and human-review-gate not its emphasis; not pediatric/Ewing; closed prompts.
- https://academic.oup.com/jamia/article/31/2/375/7413158 · https://pmc.ncbi.nlm.nih.gov/articles/PMC10797270/

**n2c2 2018 Track 1 (Cohort Selection).** Benchmark shared task: classify whether *patients* in
records meet 13 trial criteria (47 teams; best micro-F1 0.91, rule-based).
- Strengths: defines evaluation rigor and the criterion taxonomy; gold-standard for *matching*.
- Weaknesses/gaps: patient-record classification, not criteria structuring; again the patient-
  applying task the project avoids. Relevant as evaluation-methodology prior art.
- https://pmc.ncbi.nlm.nih.gov/articles/PMC6798568/ · https://portal.dbmi.hms.harvard.edu/projects/n2c2-2018-t1/

**ClinicalTrials.gov itself (the incumbent baseline).** Eligibility is stored as *semi-
structured free text*; ~49% of values fail to parse to the recommended bulleted format, and
there is no structured eligibility data element. This is exactly the gap the project exploits —
but also means CTG could ship a structured eligibility module upstream and erode novelty.
- https://clinicaltrials.gov/policy/protocol-definitions · https://www.nature.com/articles/s41597-020-00780-z

**Adjacent / recent.** MIMO sequence-labeling parser for cohort-query tuples (2024,
https://pmc.ncbi.nlm.nih.gov/articles/PMC11251129/); LLM→OMOP-CDM query conversion validation
(2025, https://pmc.ncbi.nlm.nih.gov/articles/PMC12530336/); a knowledge base of eligibility
criteria (https://pmc.ncbi.nlm.nih.gov/articles/PMC8407851/); POET / "Protocol Optimization via
Eligibility Tuning" (2026, https://arxiv.org/pdf/2602.00370); and a 2025 scoping review of LLM
patient-trial matching (https://ascopubs.org/doi/10.1200/CCI-25-00071). The field is moving fast
and almost entirely toward *matching*; the structuring-as-an-open-public-good lane is sparse.

**Landscape takeaway:** Extraction itself is solved-enough (multiple F1 0.84–0.90 systems). What
*nobody* offers is an **open, maintained, fully-provenanced, rare-disease-specific, deliberately
misuse-resistant eligibility *dataset*** (vs a corpus or a matching engine). That is the white
space.

---

## 3. Gaps we can fill

1. **Rare-disease depth no general system has.** Every competitor is general-purpose or tuned to
   common diseases (Alzheimer's, NASH). None models Ewing/pediatric-sarcoma specifics:
   *EWSR1–FLI1*/*EWSR1–ETS* fusion status, histology (ICD-O-3), **Lansky vs Karnofsky/ECOG by
   age**, relapsed/refractory states, measurable-disease (RECIST), prior-line counts. The plan's
   §6 already names these — that is the moat.
2. **Provenance to the verbatim span + hash on *every* assertion.** Chia/Leaf are annotation
   corpora without source-version lineage; tools don't persist it. Per-assertion
   `sourceText + sourceTextSha256 + registryVersion + retrievedAt + reviewer` is genuinely
   differentiated and exactly what makes the data *auditable* and *citable* (Zenodo DOI).
3. **Maintained & fresh, not a frozen corpus.** Chia (2020), EliIE corpus (2017), Leaf (2022)
   are static. A refresh job with drift detection and staleness flagging is novel for an open
   eligibility dataset.
4. **Misuse-resistant by design.** Uniquely, this project *withholds* the patient-applying step
   that all competitors race toward. "Structured data, never a verdict," with AUP + ambiguity-
   withholding + no executable patient-logic in v1, is a defensible public-good position.
5. **Open-vocabulary-first coding (NCIt/HGNC/RxNorm/LOINC/ICD-O-3/HPO).** Most prior art leans
   on UMLS/SNOMED (license-encumbered). An open-codes-first dataset is more freely redistributable
   — a real adoption advantage for foundations and small tools.
6. **Ambiguity as a first-class, *published* output.** Competitors optimize to emit *an* answer;
   this project flags and *withholds* under-specified criteria ("per protocol," references to
   non-public docs). A labeled corpus of *which criteria are not safely structurable* is itself a
   research contribution nobody publishes.
7. **An open scoring harness + gold standard for rare-disease eligibility.** No public benchmark
   exists for pediatric-sarcoma eligibility extraction; shipping one (a la Chia, but fresh + rare)
   seeds the field and invites collaborators.

---

## 4. Differentiators to win

- **"Auditable data, not answers."** Every assertion traces to an exact, hashed, versioned
  source span with a named human reviewer — citable via DOI. This is the one-line pitch and the
  thing no competitor delivers.
- **Designed to be hard to misuse.** The deliberate refusal to ship a patient-applying verdict —
  backed by AUP, ambiguity-withholding, and not shipping executable patient logic in v1 — is a
  *trust* differentiator that matters more in pediatric oncology than raw F1.
- **Rare-disease fidelity.** Fusion status, age-appropriate performance scales, RECIST/relapse
  states modeled as first-class — correctness on the criteria that actually gate Ewing trials.
- **Open licensing all the way down** (Apache-2.0 code / CC-BY-4.0 annotations / open vocab
  codes) lowers the adoption barrier for exactly the under-resourced foundations and navigators
  who are the beneficiaries.
- **Maintained + fresh + correctable** (refresh SLA, public correction/appeals path, versioned
  releases) — a living public good, not a paper artifact.
- **Humility as a feature.** Withheld-ambiguous + zero-harm metric (O8) + blocking expert gate
  signals safety culture to clinical partners deciding whether to adopt.

---

## 5. Claude API leverage

Where Claude clearly helps (all as *data-production assistance behind the human-review gate,
funded lane, budget-capped*):

- **Free-text → structured assertion drafting with inline provenance.** Claude is strong at the
  AutoCriteria-style task (entities + operator + value + unit + temporality + negation + multi-
  arm logic) and can emit the *exact verbatim source span* alongside each assertion. Pair with a
  **string-match gate** (§1.4): reject any assertion whose `sourceText` isn't a literal substring
  of the trial text — turning Claude into a high-recall first-drafter whose hallucinations are
  mechanically catchable.
- **Logic/structure parsing for compound criteria (§1.6).** Claude is good at converting "no
  chemo within 28 days unless nitrosourea (42 days)" into a nested AND/OR/NOT + exception tree
  with the windows attached — the part flat rules get wrong. Output remains a *proposal* for human
  adjudication.
- **Terminology-code candidate suggestion + ambiguity triage.** Claude proposes NCIt/HGNC/
  RxNorm/LOINC/ICD-O-3/HPO candidates (human confirms against the named vocab version), and —
  most valuably — **classifies criteria as cleanly-structurable vs ambiguous/withhold**, writing
  the `ambiguityFlag` reason. Flagging "I can't safely structure this" is a higher-value Claude
  behavior here than guessing.
- **Reviewer-assist & QA, not decisions.** Claude can pre-fill the review queue, diff a refreshed
  span against the prior version and summarize *what changed* (drift triage), and draft the M3
  plain-language explainers as *candidates* for the oncologist+advocate gate.

Where Claude must NOT decide (hard lines, mirror these into prompts and policy):

- **Never produce a per-patient eligibility verdict.** No "this child qualifies / does not." The
  output is always a description of the *trial's* criterion, never its application to a person.
- **Structured output is DATA, not a patient verdict or medical advice** — every Claude-assisted
  record keeps the not-advice/not-eligibility disclaimer.
- **Never the gold standard or IAA.** Gold annotation and inter-annotator agreement are
  human-owned; Claude output may be *evaluated against* gold but can never *be* gold (no
  self-grading, no Claude-vs-Claude IAA).
- **No fabricated structure.** If a value/code/span isn't present in the source text, Claude must
  emit nothing for it and flag — enforced by the substring gate, not trust.
- **Confidence is advisory only.** Claude's self-reported confidence never gates release or skips
  human review.
- **Never auto-merge.** Every Claude assertion enters the human-review queue (per M1 task 103).
- **Respect refusal guardrails.** If a task drifts toward individualized advice or patient
  application, Claude stops and surfaces the concern.

(Per Elyos `CLAUDE.md`: Claude usage runs via `packages/runner` on the funded lane behind the
provider adapter, with a hard per-trial budget cap and redacted logs — never secrets in output.)

Concrete Claude API mechanics worth using: **tool-use / structured outputs** to force the schema
shape, **prompt caching** of the schema + few-shot gold exemplars + vocab cheat-sheets across the
many per-criterion calls (big cost win on a budget-capped lane), and **citation-style prompting**
that requires a verbatim span per claim.

---

## 6. Ten concrete optimizations

1. **Ship the scoring harness as code in M0**, with explicit matching rules (concept exact vs
   ancestor; operator/value/unit/temporal partial credit) and **separate segmentation-recall and
   extraction-recall metrics** alongside F1. No metric is a gate until its script is in CI.
2. **Add a criterion-logic layer** (criterion groups + AND/OR/NOT trees + exception/"unless"
   clauses + value ranges + counts) to the data model, à la Chia's DAG. Without it, compound
   exclusions get silently flattened — a safety bug.
3. **Verbatim-substring grounding gate on all extractor output** (rule and LLM): reject any
   assertion whose `sourceText` isn't a literal span of the trial text. Cheap, kills the worst
   hallucinations pre-review.
4. **Switch IAA from κ to span/relation pairwise-F1** (keep κ only for categorical sub-labels).
   Define a minimum gold-set size *in assertions per criterion class*, and make the gold set a
   versioned, growing deliverable, not a one-off M0 artifact.
5. **Stratify accuracy targets by criterion class** — e.g. ≥0.95 on age/lab/washout thresholds;
   report-only on free-text prose — instead of one blanket F1.
6. **Add an explicit misuse threat-model section + AUP/data-card "responsible use" clause**, and
   **decide in writing whether to withhold executable patient-applying logic (FHIR/CQL/OMOP query
   expressions) in v1.** Resolve the latent N1/N4 vs tasks 104/205/G1 contradiction.
7. **Add a coverage-of-known-universe metric** with the canonical CTG query that defines the
   denominator, so "done" can't mean "30% covered, high quality."
8. **Pin CTG versioning precisely** (which field = `registryVersion`; capture the historical-
   version record) and **store the prior span on drift** so refresh reports *semantic* change, not
   just a hash mismatch.
9. **Mine recurring boilerplate criteria into shared templates** (organ-function blocks recur
   verbatim) — speeds structuring, prevents metric inflation from re-counting easy criteria, and
   yields a reusable template library.
10. **Add a reviewer-integrity protocol** (planted errors / blind rechecks) and make
    `confidence` explicitly advisory; track reviewer catch-rate to counter automation bias on
    LLM-assisted output.

---

## 7. Parallel & perpendicular spin-offs

- **`ewing-trial-finder` (already named, HIGH risk).** This project is its data substrate. Define
  a clean, *non-patient-applying* handoff contract: finder owns matching + recruiting status; we
  never assert "open near you." Backlog task `ewing-elig-finder-handoff` already anticipates this.
- **`ewing-family-guide` / patient-facing explainers (M3).** The plain-language "what this
  criterion means" layer — education-only, oncologist+advocate-gated, sourced to NCI/COG/ESMO/CCLG.
- **`systematic-review-assist`.** Structured, provenanced eligibility is gold for evidence
  synthesis (PICO extraction, eligibility comparison across trials, meta-analysis cohorts) — a
  research-facing, lower-risk reuse with clear academic demand (the CTG-metadata-reuse papers cite
  exactly this need).
- **Generalized open eligibility-structuring corpus + benchmark.** Promote the gold standard +
  scoring harness into an open benchmark (backlog `ewing-elig-benchmark`, Chia-aligned) — a
  fresh, rare-disease complement to Chia/Leaf that seeds the field and recruits collaborators.
- **Generalized structuring tool beyond Ewing** (backlog `ewing-elig-adjacent`): once proven,
  extend the misuse-resistant, provenance-first pipeline to other pediatric/rare cancers — the
  pipeline, not the dataset, is the reusable asset.
- **An MCP server** exposing the structured eligibility dataset as read-only, *descriptive* tools
  (`get_trial_criteria`, `search_criteria_by_concept`, `explain_criterion_provenance`) — with a
  hard guarantee it returns *criteria descriptions + citations*, never a patient verdict. This
  makes the public good directly consumable by agents while encoding the misuse boundary in the
  tool surface itself (no `is_patient_eligible` tool exists, by design).
- **`ewing-info-translations` (already named, HIGH risk).** Hand off vetted explainers for
  multilingual reach.

---

## 8. Open questions for the maintainer

1. **Logic layer:** Will you adopt a Chia-style DAG / boolean-group model now (recommended), or
   accept flat assertions and document the compound-criterion limitation? This is the biggest
   technical fork.
2. **Executable-logic boundary:** Do the FHIR/OMOP exports (tasks 205/104, goal G1) stop at
   *descriptive mapping*, or do they produce runnable cohort/CQL expressions? If the latter, how
   do you reconcile that with N1/N4 (it is functionally an eligibility engine input)?
3. **Metric definitions:** Will you replace single F1 with stratified, class-specific targets +
   separate recall/segmentation metrics, and ship the scoring script as the actual gate?
4. **IAA statistic:** Move off Cohen's κ to span/relation pairwise-F1 for the extraction parts?
   And what is the minimum gold-set size (in assertions/class) for the F1 gate to be meaningful?
5. **Clinical anchoring of the gold standard:** Can a pediatric-oncology-literate reviewer be in
   the loop for M0 gold creation, not just later audits? Today the clin-informatics reviewer is
   "TO BE SECURED" while the gold standard is being built.
6. **Freshness staffing:** Who actually runs the 30-day refresh + drift triage on a no-partner
   project? If unstaffed, should the SLA be relaxed and staleness labeling made even louder?
7. **Misuse posture:** Will you add an AUP/responsible-use clause and an explicit misuse threat-
   model section, accepting they're norm-setting (CC-BY can't legally forbid reuse)?
8. **Accuracy threshold (your Open Q7):** Given that comparable systems hit ~0.84–0.90 on
   *easier* diseases and Ewing is harder, is a blanket 0.85 the right *release* bar, or should
   safety-critical classes (age/dose/washout) carry a higher, separate bar?
9. **Scope of "structurable":** How will you publish and license the *withheld/ambiguous* set —
   as a labeled "not safely structurable" contribution, or just excluded silently?
