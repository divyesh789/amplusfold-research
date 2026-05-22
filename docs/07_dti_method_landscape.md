# DTI Method Landscape

> A survey of methods to know about. Roughly chronological. We don't need to use all of these — this is a map of the territory so you know what's a baseline, what's SOTA, and what's noise.

---

## Tier 1: Classical baselines (still respectable)

### Random Forest / XGBoost on fingerprints + protein descriptors
- **Inputs**: ECFP4 fingerprint (compound) + amino acid composition or pseudo-AAC (protein)
- **Output**: pKd / pKi / pIC50
- **Pros**: dirt simple, fast, surprisingly strong baseline. Reproducible. Interpretable.
- **Cons**: doesn't scale gracefully. Chemistry features are crude.
- **When to use**: as your first model, before any deep learning. If a deep model can't beat this on cold splits, your deep model is broken.

### Matrix factorization (e.g., NeoDTI, KronRLS)
- **Inputs**: compound-target measurement matrix
- **Output**: filled-in matrix
- **Pros**: handles missingness naturally
- **Cons**: poor cold-split (extrapolates to unseen compounds/targets badly)
- **Used today**: rarely as a primary; sometimes as a side feature

---

## Tier 2: Deep learning baselines (your starting point)

### DeepDTA (Öztürk et al., 2018)
- **Inputs**: SMILES (CNN) + protein sequence (CNN)
- **Output**: continuous pKd
- **Why it matters**: first widely-used deep DTA model; the "MNIST" of the field; reproducible and well-benchmarked
- **Datasets benchmarked on**: DAVIS (CI ≈ 0.88), KIBA (CI ≈ 0.86)
- **Status**: still a respectable baseline. Use via DeepPurpose.
- **Recommended**: train this first, on your own data, on cold splits, before doing anything fancier.

### WideDTA (Öztürk et al., 2019)
- DeepDTA + ligand max common substructure features. Modest improvement.

### GraphDTA (Nguyen et al., 2020)
- **Inputs**: molecular graph (GIN/GAT/GCN) + protein sequence (CNN)
- **Why it matters**: showed graph representations beat SMILES CNNs
- **Variants**: GAT-GCN, GIN are the commonly cited ones
- **Status**: solid; competitive with Tier 3 methods on standard benchmarks

### DeepConv-DTI (Lee et al., 2019)
- DeepDTA with attention. Marginal improvement.

### DeepPurpose (Huang et al., 2020) — library, not a model
- **What it is**: Python library wrapping DeepDTA, GraphDTA, MolTrans, MPNN, and others
- **Why it matters**: lets you train and benchmark a working DTI model in 50 lines
- **Recommended**: this is how you should run your initial baselines. Don't reimplement DeepDTA from scratch.

---

## Tier 3: Transformer-based fusion

### MolTrans (Huang et al., 2020)
- **Inputs**: substructure-tokenized SMILES (transformer) + protein sequence (transformer)
- **Fusion**: cross-attention between compound and protein representations
- **Why it matters**: first strong transformer-based DTA; introduced learned substructure tokens
- **Status**: still a good reference architecture for "transformer+transformer with cross-attention"

### TransformerCPI (Chen et al., 2020)
- Transformer encoder on protein, GNN on compound, decoder for prediction
- Decent performance; less commonly used today

### BACPI (Li et al., 2022)
- Bidirectional attention; combines GNN compound with CNN protein with cross-attention
- Improvement over GraphDTA; used in some recent work

---

## Tier 4: Foundation-model approaches (the modernization path)

### ESM2 + small head
- **Inputs**: protein sequence → ESM2 embedding (frozen)
- **Compound**: any encoder (fingerprint, GNN, ChemBERTa)
- **Why it matters**: ESM2-3B captures protein structure/function much better than CNNs trained from scratch on sequence; consistent ~0.05–0.10 R lift on cold-target splits
- **APPT** (Bindwell, 2024) is one example architecture using this
- **Recommended for our project**: this is our planned protein side

### ChemBERTa / MolFormer / MolBert
- **Inputs**: SMILES → pretrained transformer embedding
- **Why it matters**: pretrained on 10M–1B+ molecules with masked language modeling; transfer learns to downstream
- **ChemBERTa-2** (`seyonec/ChemBERTa-zinc-base-v1`) — pretrained on 77M ZINC compounds; small (~80M params)
- **MolFormer-XL** (`ibm/MoLFormer-XL-both-10pct`) — pretrained on 1.1B molecules; stronger
- **Recommended for our project**: ChemBERTa first (easier), MolFormer if needed

### Uni-Mol (DP Technology, 2022)
- **Inputs**: 3D molecular conformation
- **Why it matters**: 3D-aware foundation model for molecules; better than SMILES models when 3D matters
- **Cons**: requires conformer generation, slower
- **When to consider**: structure-aware refinement phase

### ESM-IF (Hsu et al., 2022)
- **Inputs**: protein structure (graph)
- **Why it matters**: protein language model conditioned on structure; useful when you have structures
- **When to consider**: if you have AlphaFold-predicted structures for your targets

---

## Tier 5: Structure-aware (where SOTA lives in 2024–2025)

### AlphaFold 3 (DeepMind, 2024)
- **Inputs**: protein sequence + ligand SMILES
- **Output**: predicted 3D complex + confidence
- **Why it matters**: first generally-strong predictor of protein-ligand structure; affinity is a downstream
- **Closed weights** but the paper is the playbook for the architecture (diffusion + pairformer)
- **Status**: SOTA for structure prediction; not directly an affinity model

### Boltz-1 (MIT, 2024)
- **Inputs**: protein sequence + ligand
- **Output**: 3D complex structure
- **Why it matters**: first open-weights AF3-class model
- **Use**: drop-in replacement for AF3 for our purposes

### Boltz-2 (MIT, 2025)
- **Inputs**: protein sequence + ligand
- **Output**: 3D complex + binding affinity
- **Why it matters**: adds an affinity prediction head; direct DTA from structure
- **Recommended for our project**: use as a feature extractor or as a direct predictor in Phase 5

### RoseTTAFold-AllAtom (Baker lab, 2024)
- Similar capability to AF3 / Boltz; open-source; strong on protein-ligand

### DiffDock (Corso et al., 2023)
- **Task**: predict ligand binding pose given protein structure (docking)
- **Why it matters**: state-of-the-art geometric deep learning for docking; used as a feature in some affinity pipelines
- **Status**: not a DTA model itself; pose-prediction tool

### EquiBind / TankBind (older)
- Equivariant docking models
- Largely superseded by DiffDock and structure-prediction approaches

---

## Tier 6: Generative / de novo (out of scope for us, but adjacent)

### MoLeR, MolGPT, REINVENT
- Generate new molecules with desired properties
- Used in lead optimization and de novo design
- Out of scope for the active-learning-on-existing-compounds phase, but relevant later

### Structure-conditioned generation (Pocket2Mol, etc.)
- Generate molecules that fit a given protein pocket
- Cutting edge; potential for Phase 5+

---

## Tier 7: Things you'll see cited but probably don't need

- **ChemicalChecker**, **DTI-ML benchmarks**: meta-resources
- **Pose-Net, GNINA**: classical docking + neural rescoring; mostly superseded
- **DeepPocket**: pocket detection (separate problem)
- **Kinase-specific models** (KinML, KinaseNet): kinase-only; useful only for kinase pesticide work

---

## Recommended progression for this project

| Phase | Method | Why |
|---|---|---|
| Phase 2 baseline | DeepDTA via DeepPurpose | Validate training setup against literature numbers |
| Phase 2 v2 | DeepDTA + ESM2 protein encoder | First modernization; cheap win |
| Phase 2 v3 | + ChemBERTa molecule encoder | Symmetric to protein side; strong baseline for foundation-model fusion |
| Phase 2 v4 | + cross-attention fusion (MolTrans-style) | Modest lift; let data win first |
| Phase 4 (with active learning) | Same architecture; iterate on data | The leverage is in the data, not the model |
| Phase 5 | + Boltz-2 features for high-confidence targets | Structure-aware refinement once we have AlphaFold structures of all targets |

Don't try to skip ahead to Boltz-2 in Phase 2. The data pipeline isn't ready, and structural features are worthless if your basic schema is broken.

---

## Foundation model checkpoints to download

When ready to use these, here are the canonical Hugging Face / GitHub locations:

| Model | Source | Notes |
|---|---|---|
| ESM2 (3B) | `facebook/esm2_t36_3B_UR50D` | Production protein encoder |
| ESM2 (35M) | `facebook/esm2_t12_35M_UR50D` | Fast iteration during development |
| ChemBERTa-2 | `seyonec/ChemBERTa-zinc-base-v1` | First-pass molecule encoder |
| MolFormer-XL | `ibm/MoLFormer-XL-both-10pct` | Stronger molecule encoder |
| Uni-Mol | github.com/dptech-corp/Uni-Mol | 3D-aware (later phase) |
| Boltz-1 | github.com/jwohlwend/boltz | Open structure prediction |
| Boltz-2 | github.com/jwohlwend/boltz | Open structure + affinity |

All are MIT or Apache-licensed (verify before use).

---

## Reading list

If you have time for one paper per tier:

| Tier | Paper |
|---|---|
| 2 | DeepDTA: Öztürk et al., 2018, Bioinformatics |
| 3 | MolTrans: Huang et al., 2020, Bioinformatics |
| 4 | ESM-2: Lin et al., 2022, Science (the paper introducing ESM2) |
| 5 | AlphaFold 3: Abramson et al., 2024, Nature |
| 5 | Boltz-1: Wohlwend et al., 2024 |

Plus one general overview:
- Bagherian et al., 2021, "Machine learning approaches and databases for prediction of drug-target interaction: a survey paper" — Briefings in Bioinformatics
