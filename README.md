# AmplusFold — Notes

I'm building AmplusFold, a model that predicts how well small molecules bind to proteins relevant to pesticide discovery — insect, plant, and fungal targets.

This repo has a few notes I've written along the way. The code, data, and results live in a private repo. What you'll find here is the thinking: what the field looks like, what makes wet-lab data trustworthy, how to choose what to test next, and the common ways these projects go wrong.

## Where the project is

Early models score around Pearson R = 0.35 on a held-out set of pesticide-relevant measurements, under a strict split where no protein in the test set was seen during training. That's enough to be useful for narrowing down candidate molecules before sending them to a lab, but not enough to rely on alone. The next step is getting real wet-lab data on the actual targets we care about.

## What's in here

- [`docs/01_domain_primer_dti.md`](docs/01_domain_primer_dti.md) — what drug-target interaction modeling is, and what the standard binding measurements (Kd, Ki, IC50) actually mean
- [`docs/04_assay_quality.md`](docs/04_assay_quality.md) — how wet-lab measurements work, and why their noise sets a hard ceiling on any model trained on them
- [`docs/05_active_learning.md`](docs/05_active_learning.md) — how to pick which molecules to test next when lab time is limited
- [`docs/06_failure_modes.md`](docs/06_failure_modes.md) — ten ways these projects commonly fail, most frequent first
- [`docs/07_dti_method_landscape.md`](docs/07_dti_method_landscape.md) — a survey of how people have built these models, from simple to complex
- [`docs/09_foundation_models_explained.md`](docs/09_foundation_models_explained.md) — what ESM2 and ChemBERTa actually do, in plain language

## Stack

Python, PostgreSQL, RDKit, PyTorch, ESM2, ChemBERTa.
