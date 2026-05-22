# The Active Learning Loop

> The strategy for iteratively improving the DTI model using proprietary wet-lab measurements. This is how you turn a public-data baseline into a deployable screening tool.

---

## Why active learning, not bulk testing

You will never have an unlimited assay budget. If you randomly test 5,000 compounds, the model improves only modestly because most random compounds reinforce what the model already knows.

If you spend the same 5,000 measurements on compounds the **model is uncertain about, or that fill chemical-space gaps, or that test the model's frontier**, you get dramatically more information per dollar.

Realistic scaling (cold-split Pearson R after ingestion of N proprietary measurements on your target class):

| N | Random selection | Active learning |
|---|---|---|
| 0 | 0.50 | 0.50 |
| 500 | 0.52 | 0.58 |
| 2,000 | 0.55 | 0.65 |
| 5,000 | 0.60 | 0.72 |
| 10,000 | 0.65 | 0.78 |

The gap is the entire reason active learning exists.

---

## The loop, formalized

```
   ┌─────────────────────────────────────────┐
   │  current model + measurements database  │
   └────────────────┬────────────────────────┘
                    │
                    ▼
        Score all candidate compounds in library
                    │
                    ▼
   ┌────────────────────────────────────────┐
   │  Sampling strategy (the critical bit):  │
   │   ~50% top-predicted (exploit)          │
   │   ~30% high-uncertainty (explore)       │
   │   ~20% structurally diverse (diverse)   │
   └────────────────┬───────────────────────┘
                    │
                    ▼
        Pick batch (50–500 compounds)
                    │
                    ▼
        Send to CRO for assay
                    │
                    ▼
        Receive measurements (4–6 weeks)
                    │
                    ▼
        Quality-check, ingest into schema
        (BOTH hits AND misses)
                    │
                    ▼
        Retrain model
                    │
                    ▼
        Evaluate on frozen cold splits
                    │
                    ▼
   ┌──────── Goto top ──────────────────────┐
   └────────────────────────────────────────┘
```

A "cycle" = one full pass around this loop. Realistic cadence: 4–8 weeks per cycle (CRO turnaround dominates). 6–10 cycles in year one.

---

## The 50/30/20 sampling strategy (don't deviate from this)

The single most important decision in the loop is **how you pick which compounds to test next**. The naive answer is "test the top predictions." This is wrong, and it's the #1 active-learning failure mode.

### Why pure exploit (top predictions only) fails

```
Cycle 1: model predicts compound class X is potent → test 100 compounds in class X → 5 hits found.
Cycle 2: model retrained, more confident in class X → predict more class X compounds → test → more hits.
Cycle 3: model is now an expert on class X, but has never seen class Y, Z, W.
Cycle N: discover that real pesticide candidates were in class W. You missed them entirely.
```

Pure exploit creates a **scaffold trap**: the model becomes excellent on one chemical scaffold and useless everywhere else.

### Why pure explore (random/uncertain only) fails

You waste the assay budget on compounds that are unlikely to bind. You don't generate hits. Investors and the wet lab lose patience.

### The 50/30/20 hybrid

Every cycle, your batch composition is:

#### Exploit — 50%
Top-predicted binders. Standard greedy selection. **This is what generates hits and proves the model's value.**

```python
# Pseudo-code
exploit_pool = candidate_library
exploit_scores = model.predict(exploit_pool)
exploit_picks = top_k(exploit_pool, exploit_scores, k=batch_size * 0.5)
```

#### Explore — 30%
High-uncertainty compounds. **This fixes model blind spots.**

Estimate uncertainty by:
- **Ensemble** (best): train 5 models with different seeds; uncertainty = stddev across models
- **Monte Carlo dropout**: keep dropout active at inference time; sample 20×; stddev = uncertainty
- **Last-layer Bayesian**: cheap proxy

Then pick compounds where the ensemble disagrees most.

```python
predictions = [model_i.predict(pool) for model_i in ensemble]
uncertainty = np.std(predictions, axis=0)
explore_picks = top_k(pool, uncertainty, k=batch_size * 0.3)
```

#### Diverse — 20%
Structurally distinct from anything tested so far. **This prevents overfitting to one chemical scaffold.**

Use Tanimoto distance on Morgan fingerprints (ECFP4):

```python
existing_fps = compounds_already_tested.morgan_fingerprints
candidate_fps = candidate_library.morgan_fingerprints
min_distances = [min_tanimoto_distance(c, existing_fps) for c in candidate_fps]
diverse_picks = top_k(candidate_library, min_distances, k=batch_size * 0.2)
```

A compound's score in the diverse pool = max distance from anything you've already tested. Pick compounds far from the existing dataset.

### Don't skip any of the three buckets

| If you skip | What happens |
|---|---|
| Exploit | No hits found; assay budget wasted; project loses momentum |
| Explore | Model never improves; cold-split numbers stagnate |
| Diverse | Scaffold trap; model fails outside one chemical class |

---

## Selectivity from cycle 1

For pesticides, every compound assayed against the **on-target** must also be assayed against the **off-target panel** (mammalian ortholog at minimum, ideally pollinator + aquatic too). See `docs/02_pesticide_targets.md`.

Schema implication: each compound generates ≥2 rows per cycle (one per protein). The on-target row has `protein_uniprot_id = <insect AChE>`; the off-target row has `protein_uniprot_id = <human AChE>`.

The actual product metric is the **selectivity ratio**:

```
selectivity = pKd(insect_target) - pKd(mammalian_target)
            = log10(mammalian_Kd / insect_Kd)
```

Higher = more selective for the pest. You want ratios ≥3 (1000× selective) for a viable pesticide.

You can either:
1. Train two separate DTA models (one per protein) and compute selectivity post hoc
2. Train a single multi-output model that predicts both pKds simultaneously
3. Train a model that predicts the selectivity ratio directly

(2) is recommended in practice — shared chemistry features, joint training, easier to add more off-targets.

---

## What "retrain" means

Each cycle, you retrain because the new measurements change the data distribution. Keep these reproducibility guardrails:

- **Frozen cold splits.** New measurements are *added* to the splits, not redistributed. Test set is fixed across cycles.
- **Model versioning.** Tag each retrained model `v0.0.cycle_N`. Save the training data hash, the code commit, and the cold-split metrics.
- **Bake-off, don't overwrite.** Keep all cycle checkpoints; report cycle-over-cycle metric trajectory.
- **Don't tune hyperparameters every cycle.** Lock the architecture and HPs after Phase 2; tune them only when cold-split numbers plateau for 2+ cycles.

---

## Negative data — capture all of it

Public data is positive-biased. Your wet lab generates both. **Enter every measurement**, including non-hits.

When 200 compounds come back from the CRO and 5 are hits at IC50 < 1μM:
- All 5 hits → `relation = '='`, value = measured IC50
- 195 non-hits → `relation = '>'`, value = max tested concentration (usually 10μM or 30μM)

Both go in the schema. Don't drop the 195 — they're the most valuable rows you'll generate this quarter.

How to use them in training:
- **Drop bounds**: simplest, but throws away signal
- **Right-censored regression**: model `y > threshold` properly with a survival-style loss
- **Binary auxiliary task**: classifier head predicts hit/non-hit alongside the regression head
- **Tobit / Heckman models**: classical statistical approach

For Phase 4 cycle 1, the binary auxiliary task is the simplest viable approach.

---

## Cycle cadence and budget

| Stage | Cycle size | Cycle frequency | Budget per cycle |
|---|---|---|---|
| Cycle 1 (validation) | 100–200 compounds | 6–8 weeks | $5k–$30k |
| Cycles 2–3 | 300–500 compounds | 4–6 weeks | $15k–$80k |
| Cycles 4–6 | 500–1,000 compounds | 4–6 weeks | $30k–$150k |
| Cycle 7+ (scaling) | 1,000–2,000 compounds | 4 weeks | $80k–$300k |

Total Phase 4 budget at the upper end: ~$1M for ~10k measurements. Lower end: $300k for ~5k measurements. Adjust based on per-compound assay cost from your CRO.

---

## When to stop iterating

Diminishing returns set in around 5,000–10,000 high-quality measurements per target class. Watch for:

- **Cold-split R plateau**: 2 consecutive cycles with <0.02 R improvement → likely saturating
- **Hit rate plateau**: top-decile prediction accuracy stops improving
- **Diversity exhaustion**: diverse picks stop being far from existing data → you've covered the practical chemical space

When this happens, stop pushing on this target and either:
- Add a new target (start fresh active-learning loop)
- Move to lead optimization (different problem: very high accuracy on a narrow chemical series)
- Move to multi-objective scoring (Phase 5)

---

## Tooling

For Phase 4, you'll need:
- An ensemble training pipeline (5 models, different seeds)
- An uncertainty quantification function
- A diversity scoring function (Tanimoto on ECFP4)
- A candidate library (Enamine REAL or similar — see master plan)
- A CRO contract and SOP (see `docs/04_assay_quality.md`)
- A clean retraining loop with versioning (Weights & Biases recommended)

Build the active learning machinery in Phase 2 alongside the baseline model — don't wait until cycle 1 to write the sampling code.
