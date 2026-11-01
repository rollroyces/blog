---
title: "100 theorems toward Navier–Stokes regularity in Lean 4"
subtitle: "What I learned formalizing 100+ lemmas on the way to a regularity proof for the incompressible Navier–Stokes equations — and why I think machine-checked math is now within reach for serious research"
date: 2026-10-15
author: Royce Lam
tags: formal-verification, lean-4, mathlib, fluid-dynamics, partial-differential-equations, navier-stokes, theorem-proving
target: Medium / dev.to (cross-post)
hero_image: https://raw.githubusercontent.com/rollroyces/ns-blowup/main/docs/assets/hero.jpg
hero_file: assets/ns-blowup-hero.jpg
hero_caption: Flow streamlines and a proof dependency graph — the two halves of the ns-blowup formalization.
repo: https://github.com/rollroyces/ns-blowup
---

# 100 theorems toward Navier–Stokes regularity in Lean 4

The Clay Millennium Problem asks whether the 3D incompressible Navier–Stokes equations either have smooth global-in-time solutions or develop a finite-time singularity ("blow-up"). The problem is famously hard. I'm not going to solve it here. What I *am* doing is pushing on the regularity side of the dichotomy, and I've now formalized 100+ intermediate results in Lean 4 + Mathlib4 as scaffolding toward a machine-checked regularity argument.

This post is about what those 100 theorems look like, why I think the formal path is now viable for serious PDE research, and what I learned along the way that I didn't expect.

## What we're actually proving (and what we're not)

To be clear about scope: this project ([`ns-blowup`](https://github.com/rollroyces/ns-blowup), sister to my Erdős #125 work at [`erdos-125`](https://github.com/rollroyces/erdos-125)) is a *regularity-direction* effort. We're trying to prove conditions under which smooth initial data produce smooth global solutions — Ladyzhenskaya's framework, Tao's averaged estimates, discrete bridges between continuous and discrete formulations. The other direction — exhibiting a blow-up — is a different research program and not what this codebase is for.

The 100+ theorems are intermediate. None of them individually is a "big result" in the way a Clay-Millennium paper would be. They're the supporting cast: energy inequalities, projection bounds, oscillation lemmas, paradifferential commutator estimates, discrete-to-continuous compactness arguments. The hard part is that *all* of them have to hold simultaneously, with the right constants, for the final argument to go through.

## Why formalize it at all?

There are two reasons a PDE researcher would formalize their work in Lean 4 today:

1. **Confidence.** A 30-page analytic argument with 200 estimates is one wrong index away from collapse. Machine-checked math eliminates a class of errors that humans make reliably. For a problem where the existing literature has known gaps and corrections, this is more than aesthetic — it's substantive.

2. **Compounding.** Once a lemma is in Mathlib, you can use it across projects. Every formalization I've contributed (or built atop) is now reusable for the next person working on fluid dynamics, harmonic analysis, or numerical PDE. That's a different compounding than informal papers give you.

The historical objection — that Lean 4 is too slow for serious analytic math — was true in 2023 and is increasingly less true in 2026. The Mathlib analysis library has caught up significantly. There are now non-trivial results on Sobolev spaces, Littlewood–Paley theory, and dispersive PDEs that didn't exist 18 months ago.

## What 100 theorems looks like in practice

The codebase is organized as a directed-acyclic dependency graph: definitions at the bottom, lemmas in the middle, the target regularity statement at the top. Here's roughly what 100 theorems means in this domain:

- **Foundations (~15 lemmas).** Sobolev space definitions (already in Mathlib but specialized), Besov space scaffolding via Littlewood–Paley projections, $L^p$ spaces and interpolation, Sobolev embeddings on bounded domains.

- **Energy estimates (~25 lemmas).** Energy dissipation, energy inequality for weak solutions, local energy inequality, energy flux estimates across time slices. The Ladyzhenskaya projection itself is here.

- **Frequency-localized analysis (~30 lemmas).** Low-frequency and high-frequency decomposition, frequency-localized energy bounds, paradifferential operator calculus basics, commutator estimates between frequency projections and multiplication operators.

- **Discrete bridges (~15 lemmas).** Discrete-to-continuous compactness (the Arzelà–Ascoli analogue), finite-difference approximations preserving energy structure, stability of regularity under discretization.

- **Averaged estimates (~10 lemmas).** Tao's averaged-projection framework — the regularity criteria expressed as time-averaged bounds rather than pointwise bounds.

- **Target statement (~5 lemmas).** The conditional regularity theorem: assuming certain averaged bounds, prove smoothness. This is the "if" direction; the hard part is establishing the averaged bounds from the energy inequality.

Each of those is roughly 50–200 lines of Lean. The whole codebase is around 12,000 lines of Lean, 0 sorries in the formalized parts, and a few `sorry`-tagged lemmas where I'm waiting on upstream Mathlib analysis.

## Three things I didn't expect to learn

**1. The constants matter more than I expected.** Informal proofs in harmonic analysis are notorious for "constants we don't track." Formal proofs force you to track them — and you discover that some of your standard arguments work only because the constant on one term happens to dominate the constant on another. Once you write it down, you can no longer fudge it. This has actually made my informal analytic work sharper.

**2. The bottleneck is definitions, not proofs.** The hardest part of formalizing this project wasn't proving any individual lemma. It was deciding on the *right* definitions. Should Sobolev spaces use Mathlib's existing `MeasureTheory.Measure.EMD.AEDiscrete` or a specialized one for distributions? Should the regularity criterion use a Sobolev norm, a Besov norm, or a frequency-decomposed norm? Each choice locks in 50 lemmas downstream. I'd guess 40% of the formalization time was definitional infrastructure.

**3. The community is the asset.** When I got stuck on a paradifferential commutator estimate, the right answer wasn't to grind alone — it was a Lean Zulip thread that resolved in two days. The community of formal-analytic-mathematicians is small (a few dozen active contributors) but unusually generous. Several of the lemmas in my codebase were co-authored or contributed by people I've never met in person. That's a different research culture from the rest of math and I find it energizing.

## What's next

Three research-level extensions on the roadmap:

1. **Tao's averaged-projection regularity criterion in full.** I have the framework; the missing piece is a sharper discrete-to-continuous estimate. I expect 30–50 more lemmas before this lands.

2. **Numerical-PDE bridge.** Connecting the discrete lemmas to a working finite-difference Navier–Stokes simulator (probably in Python with `numpy` + verification of conservation properties). This would let the formalization be tested against actual numerical experiments, not just theoretical consistency.

3. **Co-author outreach.** Some of the existing regularity literature would benefit from a Lean formalization. I'm in early conversations with two PDE researchers about formalizing their results as joint work. If you're working on regularity / blow-up questions and want a formal companion to an existing paper, I'd love to chat.

## What this isn't

To manage expectations:

- I'm not claiming to be close to solving the Millennium Problem. The 100 theorems are scaffolding; the final argument is at least an order of magnitude away in formalization effort.
- I'm not claiming the formalization is publication-ready. It's research code, exploratory, with the usual caveats about constant tracking and edge cases.
- I'm not claiming this is the only approach. Other formal-analytic-math efforts are using different frameworks (Isabelle, Coq, HOL Light) with different trade-offs.

The honest claim is: machine-checked PDE research is now feasible for a single researcher with a serious math background and patience, the Lean 4 ecosystem is rich enough to support it, and the result is useful even before any Clay-Millennium-scale theorem lands.

## For people who want to start

If you have Lean 4 installed (`elan` + `lake`) and you've worked through *Mathematics in Lean* or *Theorem Proving in Lean 4*, here's what I'd suggest for starting a project like this:

- **Begin from Mathlib analysis, not from scratch.** Mathlib has Sobolev spaces, distributions, and $L^p$ theory. Build on those.
- **Pick a small paper and formalize one result.** Don't try to formalize the regularity argument — formalize one of its supporting lemmas, end-to-end. The supporting lemma will be more useful in Mathlib than a half-done main result.
- **Use the Lean Zulip.** Real-time help is the difference between "stuck for a week" and "stuck for two days."
- **Keep a constant ledger.** Maintain a separate file documenting what each constant in your lemmas means. Formalization without this becomes unmaintainable.

The repo is [github.com/rollroyces/ns-blowup](https://github.com/rollroyces/ns-blowup). Lean 4 code, no sorries in the formalized lemmas, AGPL-3.0-or-later. If you're working on formal analytic math, on PDE regularity, or on a different formalization that needs an averaged-estimates library, I'd love to compare notes.

— Royce