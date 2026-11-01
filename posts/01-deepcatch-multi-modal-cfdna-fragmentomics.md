---
title: "Multi-modal cfDNA fragmentomics for ultra-early cancer detection"
subtitle: "How performance-weighted fusion of fragment-size, methylation, and sequence features pushed our multi-cancer screen below 0.001% VAF"
date: 2026-10-15
author: Royce Lam
tags: bioinformatics, cancer-screening, cfDNA, fragmentomics, methylation, machine-learning, multi-modal
target: Medium / dev.to (intended venue - no account live yet)
hero_image: https://raw.githubusercontent.com/rollroyces/deepcatch/main/docs/figures/fig1_pooled_roc.png
hero_file: assets/deepcatch-roc.png
hero_caption: Pooled ROC across cancer types on the deepcatch validation cohort.
repo: https://github.com/rollroyces/deepcatch
---

# Multi-modal cfDNA fragmentomics for ultra-early cancer detection

Liquid-biopsy cancer screening is one of those problems that *feels* like it should be solved by 2026 and, in practice, is held back by signal-to-noise. A teaspoon of blood contains a few nanograms of cell-free DNA. Most of it is healthy. The tumor fraction you are hunting for can be one part in a hundred thousand. To detect that, every feature you extract has to pull its weight.

In this post I'll walk through how [deepcatch](https://github.com/rollroyces/deepcatch) attacks that problem with **performance-weighted multi-modal fusion** across three cfDNA feature families — fragment size, methylation, and sequence — and why pooling the modalities at the score level (not the feature level) made the difference for our 0.001% VAF target.

## The signal budget

For stage-I solid tumors, circulating tumor DNA (ctDNA) fractions in plasma typically sit between 0.01% and 0.1% variant allele frequency (VAF). For the earliest, smallest tumors — the cases a screening test is actually built to catch — the realistic operating point is **0.001% VAF** and below.

At that dilution, single-feature classifiers break. A fragment-size distribution model can tell you that *something* is off about a sample; a methylation model can localize that *something* to a tissue of origin; a sequence-based mutation caller can sometimes find a specific driver. None of them, individually, gets you a confident call at 0.001%. The signal is there, but it's been split across three noisy channels.

The intuition behind deepcatch is straightforward: **the modalities fail differently**. A fragment-size signature degrades gracefully as VAF drops. A methylation signal saturates once you have enough fragments covering the right CpGs. A mutation caller needs the specific mutant fragment to be sequenced — at 0.001% VAF that's a coin flip on a typical panel. If you can combine their outputs at the *score* level with weights learned from validation data, you can recover signal that any single channel misses.

## What the features actually look like

Before talking about fusion, a quick tour of what each modality contributes:

**Fragmentomics.** Size distribution, end motifs, 4-mer sequence motifs around fragment ends, and per-bin GC correction. Tumor-derived cfDNA tends to be shorter (~120–160 bp vs. the healthy ~166 bp mode), has different 4-mer preferences, and shows a characteristic "DELFI-like" oscillation pattern. We compute these features per genomic bin (5 Mb windows), then aggregate across bins to a sample-level vector.

**Methylation.** CpG-level beta values from a targeted panel (FinaleMe / Illumina methylation EPIC subset). Tumors carry tissue-specific methylation signatures, so even at low tumor fractions the *shape* of the methylation profile is informative — the question is just whether you can detect the shape through the noise.

**Sequence.** Standard coverage tracks plus a small targeted panel of driver mutations. At 0.001% VAF, most samples have *zero* mutant fragments in any given panel — but the position-specific error profile still contributes weak evidence.

Each of these comes out of a separate pipeline. The fragmentomics path is the same `cfdna-fragmentomics-pipeline` I open-sourced earlier; the methylation path is the sibling [`deepcatch-methylation`](https://github.com/rollroyces/deepcatch-methylation) repo; the sequence path is a lightweight coverage + error-rate feature extractor built on top of `pysam`.

## Where to fuse: features, embeddings, or scores?

The first design choice for any multi-modal system is *where* to combine the modalities. We considered:**

- **Early fusion (feature level)**. Concatenate the per-modality feature vectors and feed them into one model. Pros: lets the model learn cross-modal interactions directly. Cons: the per-modality preprocessing has to be in lockstep, and the model has to re-learn what each modality means from scratch. With three modalities at different dimensionalities (fragmentomics ~2K, methylation ~10K after CpG filtering, sequence ~1K), early fusion felt fragile.
- **Mid fusion (embedding level)**. Train a small per-modality encoder, concatenate embeddings, then a head. This is the fashionable answer. It works well when each modality is rich and the encoders are large. For our modalities — most of which are already low-dimensional handcrafted features — the embeddings didn't add much.
- **Late fusion (score level)**. Train a per-modality classifier, combine at the score level with learned weights. Pros: each modality can be optimized, validated, and debugged independently. Cons: you lose cross-modal interactions at the feature level.

We chose **late fusion** because of the validation discipline it enables. Each modality gets its own train/val split, its own hyperparameter sweep, its own ROC. The fusion layer is a small set of learnable per-class weights, fit on a held-out validation cohort.

## Performance-weighted fusion

Once you have per-modality scores, you need to combine them. A weighted average is the naive answer. We found two refinements matter:

1. **Per-class weights.** Different cancers signal in different modalities. Liver cancer lights up in methylation; lymphoma barely shows there. A single weight vector flattens that. We use per-class weight matrices fit on the validation cohort.

2. **Operating-point-aware weights.** At different sensitivity targets, the optimal weight shifts. A weight vector tuned for 95% sensitivity underweights fragmentomics; a vector tuned for 99% specificity overweights it. We compute a small grid of weights, one per operating point, and pick at inference time.

The result is the multi-modal ROC you see in the hero figure. Per-modality ROCs sit noticeably below the fused curve across the full sensitivity band — exactly what you want from a fusion that's pulling real signal, not just averaging noise.

## What about cross-modal interactions at the score level?

The standard worry with late fusion is that you're throwing away interaction information. We partially address this with two tricks:

- **Concatenation of *confidence* signals.** Each modality exposes not just a score but a confidence proxy (margin to decision boundary, ensemble disagreement, calibration residual). These confidence signals *do* interact across modalities in useful ways — e.g., a high-confidence methylation call can down-weight a low-confidence mutation call — and we concatenate them into a small "meta" feature that the fusion layer uses.

- **Soft gating.** A learned per-sample gate re-weights the modalities based on a few observable features (input coverage, fragment size distribution shape). At low coverage, the gate trusts methylation more. At high coverage, it trusts fragmentomics. This isn't a deep interaction model but it captures the most important one.

These are not a substitute for true early fusion. They are, however, enough to recover most of the signal we'd lose otherwise — at a fraction of the engineering complexity.

## Engineering notes

A few practical things that mattered during the build:

- **Per-modality calibration matters more than you'd think.** Methylation classifiers tend to be over-confident at low VAF; mutation callers tend to be under-confident. A simple isotonic-regression calibration per modality was a 5-minute change that improved fusion AUC by a measurable amount.
- **Validation cohort curation is the bottleneck.** Our biggest cost was assembling a validation cohort with paired methylation + fragmentomics + sequencing at the same draws. The modeling was the easy part.
- **Reproducibility tooling is non-optional.** Every figure in the paper (and on the repo) is regenerated from a single `make` invocation. The figure you've seen throughout this post is the artifact of `make figures/fig1`. If a reviewer asks how the ROC was generated, we point them at a single command.

## What's next

Three things I'd like to push next:

1. **Add a fourth modality.** Cell-free RNA (cfRNA) is starting to be cost-effective for some indications. A cfRNA channel would round out the fNMR + cfDNA picture.
2. **Calibration at deployment.** Move from cohort-level isotonic calibration to a per-batch QC-aware calibration that adjusts when input coverage drops.
3. **Open the validation cohort tooling.** The biggest contribution we could make to the field is probably not another model but a clean, open validation cohort builder.

The repo is at [github.com/rollroyces/deepcatch](https://github.com/rollroyces/deepcatch). If you're working on liquid-biopsy ML and want to compare notes, open an issue or email me — I'm always interested in fusion strategies I haven't tried.

— Royce