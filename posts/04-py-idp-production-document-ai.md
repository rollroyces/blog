---
title: "Production document AI: a six-stage pipeline with 23 LLM backends, HITL review, and a built-in eval harness"
subtitle: "What it actually takes to ship AI document processing that someone can run in production — schema-driven extraction, human-in-the-loop review, and measuring per-field accuracy instead of vibes"
date: 2026-11-01
author: Royce Lam
tags: applied-ai, document-ai, idp, llm, ocr, fastapi, pydantic, production, engineering
target: Medium / dev.to (intended venue - no account live yet)
hero_image: https://raw.githubusercontent.com/rollroyces/py-idp/main/docs/assets/demo-hitl.svg
hero_file: assets/py-idp-hitl.svg
hero_caption: The py-idp HITL review console — queued documents, extracted fields, and inline correction.
repo: https://github.com/rollroyces/py-idp
---

# Production document AI

Most "AI document processing" demos are a prompt, a JSON schema, and a regex for the phone number. They work on the one invoice the author kept around while writing the demo. Then someone points it at a real document pile and it falls apart in four specific ways:

1. The documents are bigger than the context window.
2. The schema isn't given — you have to discover it.
3. The OCR is mediocre and nobody can fix a misread number.
4. Nobody can tell you whether the extraction is *correct*, only whether it ran.

[`py-idp`](https://github.com/rollroyces/py-idp) is my attempt at the version that survives contact with production. It's a general-purpose, AI-enabled Intelligent Document Processing library for Python, and this post is about the design decisions that matter.

## The six-stage pipeline

Everything in py-idp is organized around a fixed pipeline:

```
parse → classify → extract → assess → validate → HITL
```

Each stage has a narrow job, and each stage's output is inspectable. That last part is the important one: when a document comes out wrong, you can point at *which* stage made it wrong.

**parse** turns bytes into structured text. This is where multi-page PDFs, scanned documents, and mixed-format uploads get turned into text + page metadata + embedded images. The stage has to survive documents where text and image are interleaved.

**classify** assigns the document to a type. Given the type, the extraction stage knows which schema to use and what "valid" means for this document class. Classification is also where you decide *which* schema applies when the caller didn't specify one.

**extract** pulls fields. This is where the LLM work happens, and it's where the schema comes from: **Pydantic-schema-driven**. Fields are defined as typed Python objects, so the contract between "what I want" and "what the model produces" is checked by the type system, not by a hope and a `try/except`.

**assess** scores the extraction — confidence, field-level flags, "this value looks wrong for this document type." It's the stage that decides what needs human attention.

**validate** runs business rules over the extracted values. Not "is this a valid date" (the type system handled that) but "is this invoice total equal to the sum of its line items" or "does this employee's start date precede their first payroll record."

**HITL** is the human-in-the-loop review console. Documents the earlier stages flagged get queued for a person, who can correct fields inline and submit. Corrections flow back into the eval harness as labeled examples.

## 23 LLM backends, and why that's a design choice

py-idp supports **23 LLM backends** behind one interface — OpenAI, Anthropic, DeepSeek, Qwen, Gemini, Mistral, Groq, Ollama, llama.cpp, vLLM, and a dozen more, plus local/self-hosted options.

The reason to support that many isn't flexappeal. It's two concrete engineering reasons:

**Routing.** Different backends are good at different things. A large cloud model handles hard extraction on complex layouts; a small local model handles the 80% of documents that are simple, at a fraction of the cost and with no data leaving your network. Once you have a router, you need the backends to be swappable.

**Testing.** When you have one backend, your test suite tests one vendor's API behavior. When you have twenty-three, your test suite tests *your* abstraction — which is the thing that actually needs testing. A regression in the adapter layer shows up immediately instead of as a production surprise.

The interface is deliberately small: given a prompt and a schema, return typed values. Every adapter implements the same protocol, and a small integrity-check suite verifies each adapter behaves the way the abstraction claims. This is also why the project ships with a commercial license option — the backends themselves are third-party services and someone using this in a regulated environment will want a support contract.

## Auto-chunking for oversized documents

Documents that exceed the context window are the most common failure in real deployments. The naive approaches are both bad:

- **Truncate.** You silently lose the back half of a 200-page contract.
- **Naive fixed-size chunking.** You cut through tables, rows split across chunk boundaries, and you get fragments that are meaningless to a schema.

py-idp's auto-chunking is **schema-aware**: it uses the target field structure to decide what's a sensible boundary, and it tracks which chunk produced which field so that conflicting values across chunks can be detected rather than silently overwritten. If two chunks disagree about `invoice_total`, that's a signal for the assess stage, not a coin flip.

## Self-hosted OCR

The OCR story is: **Nanonets-OCR2-3B**, self-hosted. You can point it at a local model server and keep document images inside your own infrastructure.

This matters for a category of users that cloud API pricing quietly excludes — anyone handling documents they can't send to a third party. Self-hosted OCR also means the OCR model can be tuned for a document class (a specific font, a specific scan quality, a specific layout) without waiting on a vendor.

## The eval harness, and per-field eval reports

This is the part I'd argue most differentiates a real production system from a demo: **py-idp ships a built-in eval harness, and v0.5 added async pipeline execution with per-field eval reports.**

The problem this solves is that "how good is your document AI" is not a single number. Overall accuracy can look fine while one critical field — `total_due`, `policy_number` — is wrong 8% of the time. That field is what causes the business problem. Aggregating it away is how you end up with a system that scores 94% and still isn't usable.

Per-field eval reports give you the breakdown: accuracy, confusion, and failure examples *per field*, so you can see which field to fix first and whether your last change helped the one you cared about and hurt others.

```
eva eval --dataset labeled/ --per-field --report eval-report.json
```

The eval harness and the HITL console are the same loop: HITL corrections become labeled data, labeled data becomes eval sets, eval sets tell you what to improve next. That loop is the actual product.

## What's in the box

- **Six-stage pipeline** with each stage independently inspectable
- **23 LLM backends** behind one typed interface, with adapter integrity checks
- **Pydantic-schema-driven** extraction, plus **AI-driven schema discovery** when you don't have a schema yet
- **Self-hosted OCR** via Nanonets-OCR2-3B
- **Auto-chunking** for oversized documents
- **Async pipeline** + per-field eval reports (v0.5)
- **Built-in eval harness** with a labeling loop fed by HITL corrections
- **Production FastAPI** service + a public `idp.console` for HITL review
- **Trilingual docs** — English, 繁體中文, 简体中文
- **92% test coverage**, 1000-example fuzzers, semantic-versioned releases
- **AGPL-3.0-or-later**, with a commercial license available

## Honest limitations

- **Schema discovery is a starting point, not a finished schema.** The AI-driven discovery gets you 80% there; the last 20% is domain knowledge you have to supply.
- **Per-field eval reports need labeled data.** The harness tells you what's broken; you have to tell it what right looks like.
- **Self-hosted OCR still needs GPU budget.** Nanonets-OCR2-3B on your own hardware isn't free, it's just self-hosted.
- **The commercial license is a real consideration** if you're building a product on top of this — AGPL has obligations, and the commercial route exists for exactly that reason.

## Where to start

```bash
pip install py-idp
```

Define your fields as a Pydantic model, point it at a directory of documents, and let the HITL console catch what the model gets wrong. If you want the shape of the thing first, the docs site has the stage-by-stage walkthrough.

The repo is [github.com/rollroyces/py-idp](https://github.com/rollroyces/py-idp) — docs at [rollroyces.github.io/py-idp](https://github.com/rollroyces/py-idp), package on [PyPI](https://pypi.org/project/py-idp/). If you're running document AI in production and want to compare notes on eval methodology or HITL UX, open an issue or email me.

— Royce