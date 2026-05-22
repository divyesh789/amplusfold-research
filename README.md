# AmplusFold — Research Notes

Public research narrative for **AmplusFold**, a Drug-Target Interaction (DTI) / Drug-Target Affinity (DTA) model for **pesticide discovery**.

> **What this repo is.** A curated set of methodology docs and domain primers from the project. Used to share thinking publicly without surfacing the implementation, the data pipeline, or current model state.
>
> **What this repo is *not*.** The source code, schema, ingest pipelines, model architectures, baseline results, and execution plans live in a private repository.

---

## Project in one paragraph

Predicts how strongly small molecules bind to pesticide-relevant proteins (insect acetylcholinesterase, nicotinic receptors, plant ALS / EPSPS, fungal CYP51, etc.). Trained on public pharma data (BindingDB / ChEMBL / PubChem), evaluated on a held-out pesticide vault. Designed as a virtual-screening multiplier on top of wet-lab assays — **not** a wet-lab replacement.

## Where we are

Early baselines on the held-out pesticide vault land at **Pearson R ≈ 0.35** under a strict cold-target split (no protein-cluster overlap between train and held-out). Meaningful signal, useful for virtual-screening triage, far from deployable on its own. Closing the gap to production requires proprietary wet-lab measurements on the actual targets — that's the next phase.

---

## Index

### Domain primers

- [`docs/01_domain_primer_dti.md`](docs/01_domain_primer_dti.md) — DTI vs DTA, binding constants (Kd, Ki, IC50), why pKd is the modeling target
- [`docs/07_dti_method_landscape.md`](docs/07_dti_method_landscape.md) — Survey of DTI methods, from classical baselines to structure-aware models
- [`docs/09_foundation_models_explained.md`](docs/09_foundation_models_explained.md) — Plain-language explainer of what ESM2 (protein) and ChemBERTa (molecule) actually do for a DTI model

### Methodology principles

- [`docs/04_assay_quality.md`](docs/04_assay_quality.md) — What an assay measures, wet-lab noise floor as the model's accuracy ceiling, replicate variance
- [`docs/05_active_learning.md`](docs/05_active_learning.md) — The active-learning loop: balancing exploit / explore / diverse sampling; cycle cadence
- [`docs/06_failure_modes.md`](docs/06_failure_modes.md) — The 10 failure modes that kill DTI projects, ranked by frequency

---

## Stack (for context)

Python · PostgreSQL · RDKit · PyTorch · ESM2 · ChemBERTa
