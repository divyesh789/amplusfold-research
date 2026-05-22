# Failure Modes: How DTI Projects Die

> A pre-mortem. The 10 failure modes that have killed similar projects, ranked by frequency. Read before each major milestone and ask "are we doing any of these?"

---

## 1. Bad assay quality (the silent killer)

**Symptom**: Model cold-split numbers stop improving despite more data. Predictions feel arbitrary in production.

**Root cause**: The wet-lab assay has 0.5+ log units of replicate noise. Your model can't be more accurate than the noise floor. Adding more data only adds more noise.

**How to detect**:
- Run 10–20 known compounds through the CRO twice. Plot replicate IC50 against itself. Variance should be ≤0.2 log units.
- Check whether the positive control IC50 agrees with literature within 0.3 log units on every plate.
- Look at the spread of duplicate measurements *within* your dataset — if compound X has been measured 5 times with values ranging over 1 log unit, you've got noise problems.

**Mitigation**:
- Lock the SOP before running compounds at scale
- Validate the CRO with a small known-control batch before committing
- Demand replicate measurements (≥3 per compound) and per-plate positive controls
- Track Z' factor across plates; reject plates with Z' < 0.5

---

## 2. Pure exploit sampling

**Symptom**: Cycle 1 generates great hits in chemical class X. Cycle 5 has the same chemistry. Cold-split R on out-of-class compounds is 0.30.

**Root cause**: You only sampled top-predicted binders. The model became an expert on one chemical scaffold and never saw anything else.

**How to detect**:
- Plot tested compounds in chemical space (UMAP of fingerprints). Are they clustered tight or spread?
- Slice cold-split R by chemical class. Big drops outside the over-tested class = scaffold trap.

**Mitigation**:
- Always 50/30/20 sampling: exploit / uncertainty / diversity
- Track the diversity score (mean Tanimoto distance to existing dataset) of each batch over time
- Fix the diverse fraction at 20%, not "as time allows"

---

## 3. Throwing away non-hits

**Symptom**: Despite many tested compounds, model never learns to identify *non*-binders. Every prediction looks like "weakly binds."

**Root cause**: The 195 non-hits from each batch of 200 weren't entered into the database. Public data lacks negatives, and you discarded yours.

**How to detect**:
- Query: `SELECT relation, COUNT(*) FROM measurements GROUP BY relation`
- If `relation = '>'` is < 30% of internal records, you're undersampling negatives.

**Mitigation**:
- Pipeline: every CRO output enters the schema, no exceptions
- Treat `relation = '>'` records as first-class citizens (right-censored regression or binary aux task)
- Audit: at end of every cycle, count the non-hits ingested; should be ≥80% of compounds tested in a typical screen

---

## 4. Schema thrash

**Symptom**: Three months in, you discover the schema doesn't handle bounds (`>10μM`), or organism collapse, or measurement_type mixing. You re-ingest 2.7M BindingDB rows. Lose two weeks.

**Root cause**: Started ingestion before schema design.

**Mitigation**:
- Spend 2 weeks on schema design and a 100-row toy pipeline (Phase 0)
- Get the schema reviewed against `docs/03_schema_rationale.md` before scale ingestion
- Don't add a column without thinking through migration of existing rows

---

## 5. Unit and measurement-type mixing

**Symptom**: Model predictions are bimodal or have weird gaps. Loss curves are jagged.

**Root cause**: IC50 values from one assay condition and Kd values from another were collapsed into a single "affinity" column. They're related but not identical, so the model is trying to fit two different distributions.

**How to detect**:
- Plot histograms of `pkd` faceted by `measurement_type`. They should overlap loosely but not be identical.
- If you can't tell the measurement type from the value distribution, you've mixed.

**Mitigation**:
- Schema: `measurement_type` is a required column; never collapse
- Training: pick one measurement type (start with IC50 since it has the most data) and filter
- Modeling enhancement (later): a single model with measurement_type as an input feature, learning the conversion

---

## 6. Organism collapse

**Symptom**: Model can't distinguish insect AChE from human AChE binding. Selectivity predictions are useless.

**Root cause**: At ingestion time, all "AChE" measurements were collapsed under a generic protein label, losing organism specificity.

**How to detect**:
- Query: distinct protein_uniprot_id and protein_organism counts. If they don't match (one is much smaller), collapse happened.
- Selectivity sanity check: predict pKd for the same compound on insect AChE vs human AChE. Should give different numbers; if they're always identical, the model can't distinguish.

**Mitigation**:
- Schema: `protein_uniprot_id` is the join key, never the protein name
- Ingest: resolve the source's "protein name" + "organism" to an exact UniProt ID before insertion. Drop ambiguous ones.
- Auditing: spot-check 50 random rows across multiple organisms; verify the IDs and sequences match UniProt

---

## 7. Benchmark-set leakage (you're tuning on the test set)

**Symptom**: Cold-split R climbs steadily, but hit rate in real prospective testing doesn't match.

**Root cause**: You evaluated on the same test set after every architecture tweak. Implicit hyperparameter search on the test set memorized it.

**Mitigation**:
- Freeze cold splits before any model training. Never reshuffle.
- Hold a "vault" set: 10–20% of test data that you only evaluate on quarterly, never during iteration
- All hyperparameter sweeping goes through `split_random` validation, not cold splits
- When cold-split R seems implausibly high, that's the signal

---

## 8. Selectivity blindness

**Symptom**: Strong predictions on insect target. Promising compound goes to wet lab. Hits insect target *and* mammalian target equally. Useless as a pesticide.

**Root cause**: Trained only on the on-target. Off-targets weren't part of the loop.

**Mitigation**:
- From cycle 1: assay every compound against on-target AND off-target panel
- Multi-output model from day 1, or multi-model + selectivity post-processing
- Selectivity ratio is the actual product metric; bake it into the candidate ranking function

---

## 9. CRO selection failure

**Symptom**: First batch of internal data has noisy, weird, or inconsistent values. Doesn't match published numbers for known compounds.

**Root cause**: Cheapest-quote CRO. Or a CRO that doesn't specialize in your target.

**Mitigation**:
- Validate any new CRO with a 5–10 known-compound batch first
- Compare to literature; demand explanations for any deviation >0.3 log units
- Get raw plate-level data (per-well signal), not just summary IC50s
- Specialty > generalist for unusual targets

---

## 10. Model architecture obsession

**Symptom**: Three months in, you've tried 6 transformer variants, attention mechanisms, fusion modules. Cold-split R hasn't moved much from the DeepDTA baseline. Data pipeline is half-finished.

**Root cause**: Architecture optimization is intellectually fun. Data work is not. So engineers gravitate to architecture.

**Mitigation**:
- Self-imposed ban: no architecture work until Phase 2 ends
- Use DeepPurpose / DeepDTA as the baseline. Don't reimplement.
- The leverage in DTI is overwhelmingly in the data: cleanliness, splits, active learning, negatives
- Fancy architectures contribute 0.02–0.05 R; clean data contributes 0.10–0.20 R

---

## Summary: how to avoid these

Run this checklist before each major milestone:

| Failure mode | Check |
|---|---|
| 1 | Has the CRO been validated with positive controls? Are replicate variances ≤0.2 log units? |
| 2 | Last batch composition: is it 50/30/20? |
| 3 | Last batch ingestion: did all non-hits enter the schema? |
| 4 | Schema unchanged in last 30 days? |
| 5 | Are model inputs filtered to one `measurement_type`? |
| 6 | Are protein IDs UniProt-resolved with organism? |
| 7 | Cold-split test set untouched in last week? |
| 8 | Off-target measurements present for all on-target measurements? |
| 9 | CRO consistency: agreement with literature within 0.3 log units? |
| 10 | Time spent on data work vs architecture in last month: ≥3:1? |

Audit honestly. The cost of catching these early is small; the cost of catching them late is the project.
