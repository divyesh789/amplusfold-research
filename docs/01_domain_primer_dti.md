# DTI / DTA: A Primer

> A self-contained introduction to the Drug-Target Interaction / Drug-Target Affinity field for an engineer new to it.

## What "DTI" and "DTA" mean

**DTI — Drug-Target Interaction**: predict whether a small molecule (drug, pesticide candidate) binds to a protein (target). Often a binary prediction: "binds" vs. "doesn't bind."

**DTA — Drug-Target Affinity**: predict *how strongly* the small molecule binds the protein. Output is a continuous number, usually a binding constant.

We do **DTA** (the regression problem). DTI is sometimes used as an umbrella term for both; the literature mixes them.

## The numbers we actually predict

Five quantities you'll see constantly. They measure related but different things:

| Quantity | Symbol | Definition | Units | When measured |
|---|---|---|---|---|
| Dissociation constant | **Kd** | Compound concentration at which 50% of protein is bound at equilibrium | nM, μM | Pure binding assay (e.g., SPR, ITC) |
| Inhibition constant | **Ki** | Compound concentration that inhibits 50% of enzyme activity at equilibrium | nM, μM | Enzymatic assay; corrects for substrate |
| Half-max inhibitory concentration | **IC50** | Compound concentration that reduces enzyme activity by 50% | nM, μM | Enzymatic assay; depends on substrate concentration |
| Half-max effective concentration | **EC50** | Compound concentration giving 50% of max cellular effect | nM, μM | Cell-based assay |
| Percent inhibition | **% inhibition** | Fraction of activity blocked at a fixed compound concentration | % at e.g. 10μM | High-throughput screening |

**Crucial**: these are NOT interchangeable.

- IC50 depends on the assay's substrate concentration; Ki doesn't.
- EC50 measures cellular response, not direct binding; it bundles cell penetration, metabolism, and efficacy.
- Different assays for the same compound-target pair can give quite different numbers.

For training a model, you typically pick **one** measurement type (most common: IC50 or Kd) and don't mix unless you explicitly model the relationship.

## The log transform

Raw values span 9+ orders of magnitude (1 pM to 1 mM). Models train better on log-transformed values.

**pKd = -log10(Kd in M)**

So Kd = 1 nM (10⁻⁹ M) → pKd = 9. Kd = 1 μM (10⁻⁶ M) → pKd = 6. Kd = 1 mM (10⁻³ M) → pKd = 3.

Higher pKd = stronger binding. Same conversion for pKi, pIC50.

In our schema: `value_nM` is the raw number, `pkd` is the log transform. Train on `pkd`.

## Why predicting binding is hard

1. **Assay noise**: same compound-target pair measured by two labs disagrees by ~0.5–1.0 log units on average. This sets a noise floor your model can't beat.
2. **Negative data scarcity**: published data is overwhelmingly positive (compounds that bind). Compounds that *don't* bind are mostly unpublished. Your model has barely seen non-binders.
3. **Protein conformational flexibility**: the same protein has different binding poses depending on what's bound. Sequence alone can't capture this fully.
4. **Solubility, aggregation, off-target binding**: many "non-binders" are actually false negatives because the compound aggregated, hit an off-target instead, or didn't dissolve.
5. **The cold-split problem**: models that look great on random splits often fail in deployment. See below.

## Cold splits — the most important concept in DTI evaluation

When you split your data for evaluation, **how** you split changes the problem entirely.

| Split type | Setup | Models the question | Typical Pearson R |
|---|---|---|---|
| **Random** | 80/10/10 of all measurements | "Can the model interpolate within seen compounds and targets?" | 0.85–0.92 |
| **Cold-drug** | Test compounds never appear in training | "Can the model handle a new candidate molecule?" | 0.55–0.70 |
| **Cold-target** | Test proteins never appear in training | "Can the model handle a new target?" | 0.45–0.65 |
| **Cold-cold** | Neither test compound nor test protein in training | "Can the model handle a totally new pair?" | 0.30–0.55 |

**The published headlines are usually random-split numbers.** When you actually use the model on a real pesticide candidate against a real target, you're in cold-cold territory. Build cold splits before any model training. Freeze them. Never change them. Always report the cold-cold number as the headline.

## Standard datasets

Listed roughly by usefulness for our purposes:

| Dataset | Records | Strengths | Limitations |
|---|---|---|---|
| **BindingDB** | ~2.7M measurements | Largest curated, well-structured, handles multiple measurement types | Kinase-biased; coverage uneven |
| **ChEMBL** | ~20M bioactivities | Massive, broad coverage of pharma targets | Mixes assay conditions, requires aggressive cleaning |
| **PubChem BioAssay** | ~1B records | Includes screening data → has negatives | Noisy, varied quality |
| **PDBbind** | ~20k complexes | Has 3D structures + measured affinity | Tiny |
| **DAVIS** | ~30k | Curated kinase-inhibitor benchmark | Just kinases |
| **KIBA** | ~120k | Curated kinase benchmark with KIBA score | Just kinases |
| **LIT-PCBA** | ~3M | Curated to avoid easy-negative bias | Niche |
| **Therapeutics Data Commons** | varies | Maintains canonical splits & benchmarks | Reformats existing datasets |

For pesticide work, BindingDB + ChEMBL filtered to pesticide-relevant targets is the starting point. PubChem is useful later for negatives. PDBbind is useful when we move to structure-aware models.

## Standard methods (rough chronology)

| Year | Method | Architecture | Why notable |
|---|---|---|---|
| 2018 | **DeepDTA** | CNN on SMILES + CNN on protein sequence | First deep DTA model, simple, reproducible, our baseline |
| 2020 | **GraphDTA** | GNN on molecular graph + CNN on protein | Showed graph representations help |
| 2020 | **MolTrans** | Transformer fusion over substructures | First strong transformer-based DTA |
| 2021 | **DeepPurpose** (library) | Wraps many methods | Production-ready library; install and run |
| 2023 | **ESM-IF + small-molecule extensions** | Fine-tuned protein LM + structure module | Strong cold-split numbers |
| 2024 | **AlphaFold 3** | Diffusion over all-atom structures including ligands | SOTA for structure prediction with ligand; closed weights |
| 2024 | **Boltz-1** | Open-weights AF3-class model | First open SOTA; structure-aware DTA |
| 2025 | **Boltz-2** | Adds affinity prediction head | Direct DTA from structure prediction |

For our project: start with DeepDTA via DeepPurpose, modernize encoders with ESM2 + ChemBERTa, then later add Boltz-2 features for structure-aware refinement.

## Realistic accuracy ceilings (for planning)

| Setup | Cold-split Pearson R | RMSE on pKd |
|---|---|---|
| Random forest on fingerprints | 0.40–0.55 | 1.2–1.5 |
| DeepDTA baseline | 0.45–0.60 | 1.0–1.3 |
| ESM2 + ChemBERTa fusion | 0.55–0.70 | 0.9–1.1 |
| + Curated pesticide target subset | 0.60–0.72 | 0.8–1.0 |
| + Active learning with 5k–10k proprietary measurements | 0.70–0.80 | 0.6–0.8 |
| + Structure-aware (Boltz-2 / AF3 features) | 0.72–0.82 | 0.6–0.7 |
| Pharma-internal frontier (with millions of proprietary records) | 0.80–0.88 | 0.5–0.6 |

A 0.65 cold-split R is enough to enrich a virtual screen 10–30× over random. A 0.75 is enough for confident lead prioritization. A 0.85+ is currently out of reach without proprietary data scale that solo / small teams can't match — and that's fine for our use case.

## Glossary

- **Ligand** — the small molecule binding to the protein
- **Receptor / target** — the protein being bound
- **Active site** — the pocket on the protein where the ligand fits
- **Selectivity** — preferring one target over another (essential for pesticides)
- **Hit** — a compound that binds at meaningful potency in a primary screen
- **Lead** — an optimized hit that's been improved through medicinal chemistry
- **Mode of action (MoA)** — the mechanism by which a pesticide kills/disables a pest
- **SAR — Structure-Activity Relationship** — how chemical structure changes affect potency
- **High-throughput screening (HTS)** — testing thousands of compounds at once
- **CRO — Contract Research Organization** — commercial service that runs assays for you

## What to read next

- For the field's failure modes: `docs/06_failure_modes.md`
- For specific methods to use: `docs/07_dti_method_landscape.md`
- For pesticide-specific targets: `docs/02_pesticide_targets.md`
