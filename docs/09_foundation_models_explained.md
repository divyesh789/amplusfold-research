# Foundation Models: What ESM2 and ChemBERTa Actually Do For Us

> Plain-language explanation of the two frozen pretrained models at the core of our architecture, written so a non-ML reader can follow. If you're an agent reading this cold, this is the conceptual bridge between "we have raw SMILES and protein sequences" and "we have a model that predicts binding."

---

## TL;DR

ESM2 and ChemBERTa **turn raw chemistry/biology into numbers that a computer can actually learn from.**

Without them, a model would have to learn what a protein and a molecule even *are* before learning anything about binding — which would take Meta-scale data and millions of dollars. We get all that knowledge for free by using them.

---

## The two models we use

| Model | What it encodes | Output |
|---|---|---|
| **`facebook/esm2_t33_650M_UR50D`** (ESM2-650M) | Proteins | 1280-d vector per protein |
| **`seyonec/ChemBERTa-zinc-base-v1`** (ChemBERTa-2) | Small molecules | 768-d vector per molecule |

> **Why 650M and not 3B**: A 2025 transfer-learning benchmark (Nature Sci Reports) found medium-sized PLMs (650M, 600M) consistently match or beat their bigger siblings on downstream tasks when training data is limited (≤a few million rows). We have 1.25M rows. ESM2-650M is 5× faster to embed with, half the storage per vector, MIT-licensed, and roughly equivalent in accuracy at our data scale. ESM2-3B remains an option for Phase 2 v2b if cold-target numbers plateau.

Both are pretrained on huge unlabeled datasets, both are frozen (we never train them), and both run as feature extractors only.

For full citations, sizes, and licenses see `docs/07_dti_method_landscape.md` and the "Stack decisions" section of `CLAUDE.md`.

---

## Concrete example: aspirin + human AChE

Say you want to predict whether **aspirin** binds to **human acetylcholinesterase (AChE)**.

You start with two raw strings:

```
Aspirin (compound):
   CC(=O)Oc1ccccc1C(=O)O
   ^ a SMILES — meaningless to a neural network as-is

Human AChE (protein):
   MTPPQCLLHTPSLASPLLLLLLWLLGGGVGAEGREDAELLVTV...
   (583 amino acid letters)
   ^ a sequence — also meaningless to a neural network as-is
```

These are strings. A computer can't do math on strings. The frozen models **convert** them into numbers.

---

## What ChemBERTa does

You feed `CC(=O)Oc1ccccc1C(=O)O` into ChemBERTa:

```
"CC(=O)Oc1ccccc1C(=O)O"
        │
        ▼
   ┌──────────────┐
   │  ChemBERTa   │
   │  (frozen)    │
   └──────┬───────┘
          │
          ▼
[0.23, -1.4, 0.78, 0.04, ..., -0.66]   ← 768 numbers, summarizing aspirin's
                                          structure, functional groups, ring
                                          systems, hydrogen-bond patterns, etc.
```

That 768-dimensional vector is **a numerical fingerprint of aspirin's chemistry** — encoded in a way that captures which structural features matter. Two similar molecules (e.g., aspirin and salicylic acid) will produce **similar vectors**. Two completely different molecules will produce **different vectors**.

---

## What ESM2 does

You feed the AChE sequence into ESM2:

```
"MTPPQCLLHTPSL...AELLVTV"   (583 letters)
        │
        ▼
   ┌──────────────┐
   │   ESM2-650M  │
   │   (frozen)   │
   └──────┬───────┘
          │
          ▼
[0.91, 0.12, -2.3, 0.55, ..., 0.04]   ← 1280 numbers, summarizing the
                                         protein's fold, active site
                                         features, family, function, etc.
```

That 2560-dimensional vector is **a numerical fingerprint of human AChE's biology** — it captures which residues are in the active site, what enzyme family it belongs to, how it likely folds, etc.

---

## What "knowledge" is encoded in these vectors

This is the magical part. During pretraining, these models learned to "fill in the blanks":

- **ESM2** was given protein sequences with random amino acids hidden, and trained to predict the missing ones. Doing this 65 million times taught it deep stuff about proteins:
  - Which amino acids are interchangeable (lysine ↔ arginine — both positively charged)
  - Which positions tend to be conserved (active sites, binding pockets)
  - How sequences relate to fold (it can predict structure surprisingly well)
  - Which mutations are tolerated vs catastrophic

- **ChemBERTa** did the same thing with SMILES — hide pieces of molecules, predict them. After 77M molecules, it learned:
  - What functional groups exist and how they combine
  - Which substructures co-occur
  - How aromatic systems behave
  - Equivalent SMILES of the same molecule

**None of this knowledge is about binding.** ESM2 doesn't know what AChE binding is. ChemBERTa doesn't know what binding is at all. They just know proteins and molecules deeply.

---

## What our small trainable model does

Now we have two vectors:
- Aspirin → 768 numbers (chemistry summary)
- AChE → 2560 numbers (biology summary)

Our small trainable model learns the missing piece: **given a chemistry summary and a biology summary, predict the binding strength.**

```
Aspirin's 768 numbers ───┐
                         │
                         ▼
                   ┌─────────────┐
                   │ Small model │   ← THIS is what we train
                   │  (~3M params)│      from labeled data
                   └─────┬───────┘
                         ▲
                         │
Human AChE's 2560 nums ──┘

Output: predicted pKd = 4.2  (i.e., aspirin is a weak binder of human AChE)

Truth from BindingDB: pKd = 4.0  ✓ close!
```

The model learns this by seeing thousands of examples like:

```
ChemBERTa(aspirin)    + ESM2(human AChE)   → 4.0   ← ground truth
ChemBERTa(donepezil)  + ESM2(human AChE)   → 8.7   ← ground truth
ChemBERTa(donepezil)  + ESM2(insect AChE)  → 6.5   ← ground truth
... thousands more ...
```

After enough examples, it learns patterns: "molecules that look like X tend to bind to proteins with active sites like Y at strength Z."

---

## Why this division of labor matters

| Task | Who learns it | Cost |
|---|---|---|
| What proteins are, how they fold, what active sites are | ESM2 (already done by Meta) | $millions, billions of GPU-hours |
| What molecules are, what functional groups do | ChemBERTa (already done by community) | hundreds of GPU-days |
| **How specific molecules bind to specific proteins** | **Our small model** | **A few hours on one GPU** |

We get the first two for free. We only have to do the third one — which is the only part that's specific to our problem (pesticide discovery on insect targets).

This is why the pattern is "frozen base + small trainable head." The base does the unsupervised heavy lifting (general protein/molecule knowledge). The head does the supervised application-specific lifting (predicting binding).

---

## In one sentence

**ESM2 and ChemBERTa convert raw biological/chemical inputs into rich numerical features that capture general knowledge about proteins and molecules — and we train a small model on top that uses those features to predict binding strength specifically.**

Without them, our model would have to learn what amino acids are, what functional groups are, what proteins are, etc. — from scratch — using only the ~3M binding measurements we have. That's not enough data. With them, we only need to learn the binding part, which IS something 3M examples is enough for.

---

## Where this fits in the pipeline

The output vectors from ESM2 and ChemBERTa are **cached in the database** to avoid recomputing them every training run:

- `proteins.esm2_650m_embedding` — populated once per unique protein in Phase 1.5 (PRIMARY column we'll fill)
- `proteins.esm2_3b_embedding` — reserved for optional later upgrade
- `compounds.chemberta_embedding` — populated once per unique compound in Phase 1.5

See `docs/03_schema_rationale.md` and `schema/schema.sql` for the storage layout, and `MASTER_PLAN.MD` Phase 1.5 for when these get computed.

---

## See also

- `docs/00_context_for_claude.md` — why we picked ESM2 + ChemBERTa over alternatives
- `docs/01_domain_primer_dti.md` — the broader DTI/DTA field
- `docs/07_dti_method_landscape.md` — survey of methods including alternative foundation models
- `src/amplusfold/models/README.md` — Phase 2 architecture progression (v1 → v4)
