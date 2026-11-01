---
title: "Eleven tools and twelve model adapters: building a usable Python layer over the mRNA-therapy AI literature"
subtitle: "mrnavax turns a scattered set of published papers into CLI subcommands and typed Protocol contracts — stdlib-only core, 387 tests, PyPI-published"
date: 2026-11-01
author: Royce Lam
tags: bioinformatics, mrna, codon-optimization, neoantigen, foundation-models, llm, clinical-trials, python
target: Medium / dev.to (intended venue - no account live yet)
hero_image: https://raw.githubusercontent.com/rollroyces/mrnavax/main/docs/assets/pipeline.svg
hero_file: assets/mrnavax-pipeline.svg
hero_caption: The mrnavax pipeline — codon → variant → neoantigen → trial → LNP → manufactured mRNA.
repo: https://github.com/rollroyces/mrnavax
---

# Eleven tools and twelve model adapters

There's a specific kind of frustration that anyone who's tried to build on the mRNA-therapy AI literature will recognize. The papers are good. The methods are real. But every one of them is a separate repository with its own dependency set, its own preprocessing assumptions, and its own idea of what a "result object" looks like. Six months into a project you're maintaining five virtual environments and reimplementing the same sequence-parsing helper for the third time.

[`mrnavax`](https://github.com/rollroyces/mrnavax) started as my attempt to stop doing that. It's a Python package with **eleven small runnable tools** that map 1-to-1 onto the published AI leverage points in mRNA cancer therapy, plus a **typed integration contract for every published foundation model in the field**.

The design constraint I set at the start was strict: **the core is stdlib-only**. No numpy, no torch, no pandas in the core path. Everything that needs a heavy model goes behind a Protocol contract.

## The eleven tools

Each tool is a CLI subcommand and imports cleanly as a Python module. They group into four layers:

**Sequence design**
- `codon` — CAI, GC content, rare-codon analysis
- `construct` — construct assembly with CAI reporting
- `utr-design` — combined UTR scoring
- `LinearDesign` — the O(L) linear-time Pareto-optimal mRNA design DP
- `RiboDecode` — integration with the Li et al. 2025 ribosome-decoding model

**Variant prioritization**
- `variant-regulatory` — variant classification with high-impact calling
- `AlphaMissense` — integration with Cheng et al. 2023
- `BLOSUM62` + Chou-Fasman — substitution and secondary-structure heuristics

**Therapeutic targeting**
- `neoantigen` — candidate generation
- `trial` — TrialGPT-style clinical-trial matching, with the Sim-ICL in-context-learning approach
- `lnp` — lipid-nanoparticle composition targeting
- `scrna` — single-cell expression analysis with clustering
- `spatial` — tissue module discovery
- `manufacture` — manufacturability scoring with pass/fail checks

And `predict`, which upgrades from a mock to a **live AlphaGenome Atlas** call when an API key is configured.

The end-to-end run is the point:

```text
mrnavax end-to-end demo (mock mode)

  codon                CAI=0.720, n_codons=169
  neoantigen           8 candidates
  trial                top-1 = NCT00000003
  lnp                  target=lung, cargo=sarna
  scrna                10 cells × 9 genes, 4 clusters (tumor=3)
  manufacture          score=0.887, 7 pass / 8 total
  spatial              10 tissue modules
  construct            384 nt, CAI=0.887
  utr-design           combined=0.807
  variant-regulatory   6 variants, 6 high-impact
  predict              score=1.000, classification=high
```

All eleven tools, roughly thirty seconds of real runtime, **zero model downloads**. The mock mode isn't a toy — it's the default so that the package is usable in CI, in a container, or on a laptop with no GPU, and so the tests don't depend on network access to five different model providers.

## Why Protocol contracts instead of direct dependencies

The published foundation models in this field — RiboDecode, STModule, ESM2 + the APLM pattern, TrialGPT + Sim-ICL, AlphaGenome Atlas — each live in its own repository with its own heavy dependency tree.

If you depend on them directly, your project inherits all of them, and any two of them might pin incompatible versions of `torch` or `transformers`. That's how you end up with four separate environments and a resolver fight every time one upstream releases.

So mrnavax defines a **Protocol** — a structural type — for each model integration. The core defines the contract; the model itself is an optional extra you install separately:

```python
class NeoantigenPredictor(Protocol):
    def predict(self, variants: Sequence[Variant], context: Context) -> list[Candidate]: ...
```

Any object with that shape satisfies the contract, whether it's the real model, a mock, or your own implementation. The core never imports a model. The heavy dependencies live behind optional extras (`mrnavax[models]`, `mrnavax[ribodecode]`, and so on).

Three things fall out of this:

1. **The package installs fast and stays small.** Core is stdlib-only; `pip install mrnavax` doesn't pull a GPU stack.
2. **Tests run without models.** The mock implementations are the test fixtures. 387 tests, no network.
3. **You can swap models.** If a better neoantigen model appears next year, you implement the Protocol and nothing else changes.

That last point is the real payoff. In a field where the best model changes every few months, the contract is the stable thing.

## The backend integrity checks

Twelve real-model adapters means twelve places where the abstraction could quietly lie — an adapter that returns a slightly different shape, a version drift in an upstream API, a silently-degraded fallback.

mrnavax ships **36 backend integrity checks** that verify each adapter still satisfies its contract. These run in CI, so a breaking upstream change surfaces as a failing check rather than as a wrong result three months later.

It's unglamorous work and it's the difference between "we integrate with these models" and "we integrate with these models *and can prove we still do*."

## Adding a new tool

Because the core is stdlib-only and every model sits behind a Protocol, adding a new tool is a bounded, mechanical job:

1. **Define the data types.** Plain dataclasses or Pydantic models for the inputs and outputs. These are the package's public contract, so they need to be stable and well-documented.
2. **Write the pure implementation.** The stdlib-only path — the heuristic, the DP, the algorithm. This is the part that's fast, dependency-free, and testable in isolation.
3. **Write the Protocol** for the model-backed variant, if the tool has one.
4. **Implement the adapter** in an optional extra, with the model behind it.
5. **Add the integrity check** that verifies the adapter satisfies its contract.
6. **Add the CLI subcommand** and the module-level import path.

The mock implementation falls out of step 2 for free — if you can write the pure heuristic, you can write the mock, and the mock becomes the test fixture. This is why the test suite runs 387 tests without downloading a single model.

The protocol boundary is also what makes the tool *composable*. Because `neoantigen` returns a stable type and `trial` accepts one, you can chain them without either tool knowing the other exists. The end-to-end demo chains eleven of them in sequence.

## Packaging and release engineering

mrnavax is a **published PyPI package** (currently v0.34.6), which sounds like a detail and isn't:

- **Semantic versioning**, with releases that actually mean something — the version number tells you whether an API changed.
- **Python 3.11–3.14** support, tested across the range.
- **AGPL-3.0-or-later** with a **commercial license** option, matching my other bioinformatics work.
- **Three documentation locales.**
- **CI badges** on the README that reflect real passing runs.
- **A 60-second quick start.** The whole point of the package is that you can get from `pip install` to a meaningful result before your coffee gets cold.

## Why the mock mode matters more than it sounds

Every AI-for-science package I look at either requires API keys to run at all, or ships a demo that only works for the author. mrnavax runs the full eleven-tool pipeline in mock mode with zero downloads, and the tests exercise that path.

For research code that has to be reproducible, this is the difference between a pipeline someone can re-run in two years and a pipeline that requires the author's laptop state. Reproducibility isn't a nice-to-have in a field where the models change quarterly — it's the only way anyone can check your work.

## What's next

- **More real-model adapters** — the Protocol contracts are in place; each new integration is a small, isolated job
- **A `predict` path that composes multiple backends** with a documented ensemble strategy
- **Deeper `scRNA` and `spatial` integration** with more published foundation models

## Where to start

```bash
pip install mrnavax
mrnavax codon --sequence dna.fa --report
```

Read the quick start, run the end-to-end demo, then pick the layer you care about. Docs at [mrnavax.github.io](https://github.com/rollroyces/mrnavax).

The repo is [github.com/rollroyces/mrnavax](https://github.com/rollroyces/mrnavax) — package on [PyPI](https://pypi.org/project/mrnavax/). If you're building on the mRNA-therapy literature and want to compare notes on Protocol design or model adapters, open an issue or email me.

— Royce