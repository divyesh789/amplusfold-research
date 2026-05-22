# Assays and Wet-Lab Data Quality

> What an assay is, how the data your model trains on actually gets generated, and why "assay quality" is the dominant factor on your model's accuracy ceiling.

---

## What an assay is

An **assay is a lab procedure that measures something biological** — specifically, "does compound X do something to protein Y, and how much?"

It's the experimental act of putting a compound and a protein together in a controlled environment and reading out a number. **That number is the row that ends up in your `measurements` table.**

Every row in BindingDB, ChEMBL, and your future proprietary database came from somebody's assay.

## Concrete example: insect AChE inhibition assay

You want to know if compound X inhibits acetylcholinesterase from a housefly. The assay looks like this:

```
1. Buy or express purified housefly AChE protein
2. Prepare 8 wells with different compound X concentrations:
   0 nM, 10 nM, 30 nM, 100 nM, 300 nM, 1 μM, 3 μM, 10 μM
3. Add purified AChE to each well
4. Add a substrate (acetylthiocholine + DTNB) that AChE cleaves to make a yellow product
5. Measure the rate of yellow appearance over 5 minutes (= AChE activity)
6. Plot rate vs compound concentration → sigmoid curve
7. Concentration that halves the rate = the IC50
```

You walk away with: **"Compound X has IC50 = 250 nM on Musca domestica AChE."**

That single number, plus the protein UniProt ID and the SMILES, is one row in your `measurements` table.

## The three tiers of assays

| Tier | What it measures | Throughput | Cost/compound | Reliability |
|---|---|---|---|---|
| **Biochemical** (purified protein in a tube) | Direct binding / enzyme inhibition | 100s–1000s/day | $5–$50 | Highest signal-to-noise |
| **Cell-based** (live insect cells in culture) | Effect on actual cells | 10s–100s/day | $50–$500 | Mid; bundles cell penetration |
| **Whole-organism** (live insects, larvae, plant tissue) | Real insecticidal/herbicidal effect | A few/week | $200–$5,000 | Lowest throughput, most ground truth |

Your DTI model trains on **biochemical assay data** (Tier 1). The downstream tiers are how you confirm the model's predictions matter in the real world. Don't try to train a model on whole-organism assay data — too noisy, too sparse, too many confounds.

---

## Why assay quality dominates everything

**Your model's accuracy ceiling is your assay's noise floor.** No amount of architectural cleverness beats this.

Numerical reality: run the same compound through the same assay twice. You should get the same answer. In practice:

| Assay quality | Typical replicate variance | Model ceiling (cold-split RMSE on pKd) |
|---|---|---|
| **High** (validated SOP, controls, replicates) | ±0.05–0.15 log units | ~0.5–0.7 |
| **Mediocre** (industry-typical) | ±0.3–0.5 log units | ~0.8–1.1 |
| **Poor** (one-off, no controls) | ±0.7–1.0 log units | ~1.2–1.6 |

If you have a poor assay, your model can't be more accurate than ~1.0 log unit RMSE no matter how clever the architecture. Most "active learning didn't work for us" failure stories are actually "we had bad assays" stories.

---

## What assay quality looks like in practice

Things that distinguish a high-quality assay batch from a sloppy one:

### Protein
- Same construct (same species, same isoform, same tag, same expression system) across all measurements
- Same lot when possible; if you must switch lots, validate with positive controls
- QC'd (mass spec, SDS-PAGE) before use
- Active-fraction quantified (not all purified protein is folded/active)

### Buffer and conditions
- Identical buffer composition across batches
- Identical incubation time and temperature
- Identical DMSO concentration cap (most assays tolerate ≤1%)
- Identical readout instrument and gain

### Plate setup
- Plate randomization (avoid systematic edge effects)
- Compound concentrations on a log-spaced grid (typically 8–10 points spanning 4–5 log units)
- Replicates: minimum 3 per compound per concentration
- Negative control wells (DMSO alone) on every plate
- Positive control wells (a known inhibitor with a known IC50) on every plate

### Validation
- The positive control's measured IC50 must agree with literature within 0.3 log units, every plate
- If positive control drifts, throw out the plate and rerun
- Run a calibration curve weekly (signal linearity)

---

## What you'll see in public data — the noise you're ingesting

Every row in BindingDB or ChEMBL is somebody's assay output. Different labs, different protocols, different protein constructs, different days.

Implications:

- The same compound-target pair often appears multiple times with different values
- Disagreement of 0.5–1.0 log units between labs is common
- Some "low binders" are actually inactives that hit the assay limit (`relation = '>'`, value = max tested concentration)
- Some "strong binders" might be aggregators that look potent for non-specific reasons
- ChEMBL's `confidence_score` (1–9) tries to flag this; trust scores ≥7

**Strategy on ingest**:
- Don't deduplicate aggressively (median across labs is usually a reasonable target)
- Carry the source through; flag suspicious values, don't auto-discard
- Use the `relation` column religiously
- Filter ChEMBL records with `confidence_score < 7` for training; keep them flagged for analysis

---

## CRO selection (when you start generating proprietary data)

For Phase 3, when you start running your own assays via a Contract Research Organization:

### What a "good" CRO looks like

- Proven track record on your target class (ask for assay validation reports)
- Published SOPs (or willing to share theirs)
- Provides positive control data with every batch
- Provides raw plate-level data, not just summary IC50s
- Offers replicate testing as a default
- Quick turnaround on a small validation batch (you should run 5–10 known compounds first)

### Pitfalls to avoid

- Cheapest-quote CROs usually have noisy data. Cost is correlated with quality.
- "All-in-one" CROs that don't specialize. Pick one that does AChE assays specifically if you're doing AChE.
- CROs without species-flexibility. You may need to swap insect ortholog mid-project.

### Common CROs for our use cases

- **Eurofins** — large, broad menu, including pesticide-relevant targets
- **Reaction Biology Corporation** — kinase-heavy but also AChE, broad enzyme menu
- **WuXi AppTec** — broad, large-scale
- **Specialty CROs** for specific insect/plant targets — search "[target name] assay CRO"

### What you ship to a CRO and what comes back

You ship:
- A list of compound SMILES (or a physical compound library)
- Concentration range and number of points
- Number of replicates
- Target protein identity (and sometimes the protein itself, if rare)
- SOP or assay-condition spec

They ship back:
- Per-well raw signal (CSV/Excel)
- Per-compound dose-response curve
- IC50 (or Kd, depending on assay) with confidence interval
- Quality control results (positive control value, signal-to-noise, Z' factor)

You ingest the per-compound IC50 + CI into `measurements` with `source = 'internal:CRO_name:batch_id'` and `confidence_score` derived from the CRO's QC metrics.

---

## Negative data — your proprietary advantage

When the CRO sends back results, **enter every measurement, including the misses.**

Public data is overwhelmingly positive-biased: pharma and academia publish hits, keep misses. So the model has barely seen any non-binders. When your CRO tests 200 compounds and 5 hit, **all 200 go into the database**, not just the 5. The 195 misses are uniquely valuable training signal.

Schema-wise: misses look like `relation = '>'`, `value_nM` = max-tested-concentration. They're censored measurements. Either drop them from training, model them with right-censored regression, or use them as binary classification labels (binds/doesn't-bind). Always store them.

---

## Quick glossary

- **Z' factor**: assay quality metric; Z' > 0.5 is considered good for HTS
- **DMSO**: dimethyl sulfoxide; the standard solvent for compound stocks
- **CRO**: Contract Research Organization; the lab that runs assays for you
- **HTS**: high-throughput screening; testing thousands of compounds at once
- **DRC**: dose-response curve
- **Hill slope**: steepness of the DRC; deviations from 1.0 suggest non-canonical binding (cooperative / aggregator / multiple sites)
- **Aggregator**: compound that forms colloids and inhibits enzymes non-specifically; common false positive

For more on field methods and metrics, see `docs/01_domain_primer_dti.md`.
