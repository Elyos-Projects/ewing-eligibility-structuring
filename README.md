# ewing-eligibility-structuring

> Ewing sarcoma is a rare bone/soft-tissue cancer that strikes mostly children, adolescents, and young adults (AYA), driven in ~85% of cases by the *EWSR1–FLI1* fusion (and related *EWSR1–ETS* fusions).  ·  **Risk tier:** med  ·  **Status:** planning

Ewing sarcoma is a rare bone/soft-tissue cancer that strikes mostly children, adolescents, and young adults (AYA), driven in ~85% of cases by the *EWSR1–FLI1* fusion (and related *EWSR1–ETS* fusions). For relapsed or refractory disease, clinical trials are often the most meaningful path — but the trials are scattered across registries (ClinicalTrials.gov, EU CTIS, WHO ICTRP, ISRCTN), and their *eligibility criteria* are buried in dense, free-text inclusion/exclusion blocks written for clinicians, not families. The result: families and even general clinicians struggle to tell which trials a child could plausibly be a candidate for, and trial-matching tools have nothing machine-readable to work with for this rare disease.

**Definition of shipped:** gold-standard regression) **+** provenance 100% **+** accuracy gate passed **+** (for patient-facing) oncologist + advocate sign-off recorded **+** the artifact actually handed to / adopted by the partner or beneficiary workflow. *Delivered, not merged.*

This is an **Elyos** good-deed project. Contributors pull a task, do it with their own coding agent, and open a PR. Platform: https://github.com/jdev1977/elyos

## Plan
- [PLAN.md](./PLAN.md) — robust enterprise plan (vision, architecture, roadmap, risks; includes an applied-improvements appendix + review sign-off)
- [TASKS.md](./TASKS.md) — schema-mapped task backlog
- [tasks/](./tasks/) — ready-to-pull task JSON(s)

## Contribute
```bash
elyos browse
elyos next --repo Elyos-Projects/ewing-eligibility-structuring --no-fork
```

## Licensing & review
- Open license (see PLAN.md).
- Risk tier **med** — deeds are *delivered, not merged*; a domain reviewer (and expert sign-off for any high-stakes content) must approve before merge.

> Planning stage; no adopting partner secured yet (`verifiedNeed: false` on delivery-dependent tasks).
