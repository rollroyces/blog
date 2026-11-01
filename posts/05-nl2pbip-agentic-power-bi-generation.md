---
title: "Agentic Power BI generation: from one natural-language prompt to a Git-friendly .pbip workspace"
subtitle: "How nl2pbip builds a TMDL semantic model, a PBIR report surface, and an OPC-compliant .pbit package — with an agentic self-reflection loop and 1,082 tests"
date: 2026-11-01
author: Royce Lam
tags: applied-ai, powerbi, power-bi, llm, agentic-ai, data-engineering, business-intelligence, generative-ai
target: Medium / dev.to (intended venue - no account live yet)
hero_image: https://raw.githubusercontent.com/rollroyces/nl2pbip/main/assets/architecture.png
hero_file: assets/nl2pbip-architecture.png
hero_caption: The nl2pbip architecture: LLM planner, TMDL writer, PBIR layout engine, and the .pbit exporter.
repo: https://github.com/rollroyces/nl2pbip
---

# Agentic Power BI generation

Every BI team I've worked with has the same recurring moment: someone knows exactly what analysis they want, and it takes three sprints to get from "the ask" to a dashboard that opens in Power BI Desktop. The gap isn't analysis — it's the mechanical work of writing a semantic model, laying out report pages, and wiring visual bindings.

[`nl2pbip`](https://github.com/rollroyces/nl2pbip) is an engine that closes that gap. One natural-language prompt produces a complete **Power BI Project (`.pbip`)** directory: a semantic model in **TMDL**, a report surface in **PBIR JSON**, and a packaged workspace that opens in Power BI Desktop, exports to `.pbix`/`.pbit`, and commits cleanly to a **Microsoft Fabric Git repo**.

This post is about the architecture — specifically why the AI part is smaller than you might expect, and why the packaging part is harder than it looks.

## Why `.pbip` and not `.pbix`

The `.pbix` format is a binary blob. It's effectively opaque to version control: you can't diff it, you can't review a change to it, you can't merge two of them. That makes it hostile to the way BI work actually happens now — in Git, with review, with CI.

The `.pbip` format is Microsoft's answer to that: a **project** directory with human-readable files, diffable and mergeable. It's what you'd want to `git commit`. So `nl2pbip` targets `.pbip` as the native output, and `.pbix`/`.pbit` become *export* artifacts rather than the source of truth.

That decision drives everything else about the design.

## The pipeline

There are six distinct components, and the important architectural choice is that **only two of them are LLM-driven**:

```
LLM planner  →  TMDL writer  →  PBIR layout engine
                                    ↓
                    deterministic data inspection
                                    ↓
              curated schema.org ontology
                                    ↓
                 OPC-compliant .pbit exporter
```

**The LLM planner** turns the requirement into a structured plan: which measures, which dimensions, which breakdowns, what the visual structure should be. It doesn't write raw files.

**The TMDL writer** takes that plan and emits a valid Tabular Model Definition Language model — tables, relationships, measures with DAX, calculated columns, hierarchies, roles, and object-level security.

**The PBIR layout engine** is **not** LLM-driven. It's deterministic code that takes the plan and the model and produces the report surface: pages, visual positions, visual types, and data bindings, in PBIR JSON.

**Deterministic data inspection** reads the actual data source (through whatever connector the caller provides) to learn real column names, types, cardinalities, and sample values. This is how the engine avoids hallucinating a column that doesn't exist — it reads the schema instead of guessing.

**The curated `schema.org` ontology** provides the semantic layer — a mapping from business concepts to concrete measure and dimension choices, so "revenue" means *the* revenue measure in this workspace, not whatever the LLM thought revenue meant.

**The OPC-compliant `.pbit` exporter** packages the output. `.pbit` is a template file, and its format is OPC (Open Packaging Conventions) — a ZIP with a specific internal layout and relationship parts. Getting this right is fiddly, and getting it wrong produces a file that Power BI refuses to open.

The point of the split: **LLMs do the part that's genuinely open-ended, deterministic code does the part where correctness is checkable.** You can write a test that asserts the TMDL parses. You cannot write a test that asserts an LLM wrote "nice" TMDL.

## A quick TMDL and PBIR primer

If you haven't worked with the modern Power BI file formats, the two acronyms deserve a minute.

**TMDL** (Tabular Model Definition Language) is the declarative language for the **semantic model** — the thing that describes your data: tables and columns with types, relationships between them, DAX measures, hierarchies, roles, and object-level security rules. It's a textual format, human-readable, diffable, and designed for exactly the version-control workflow that `.pbip` enables.

**PBIR** (Power BI Enhanced Report Format) is the JSON representation of the **report surface** — the pages, the visuals placed on them, their positions and sizes, their types, and their data bindings. Where TMDL describes what the data *means*, PBIR describes what the user *sees*.

The split matters because the two are edited by different kinds of people and change at different rates. A semantic model is reviewed carefully and rarely; a report layout is tweaked constantly. Keeping them in separate, text-based files means a layout tweak produces a tiny diff, and a semantic-model change produces a diff a data engineer can actually review.

`nl2pbip` writes both, and the boundary between them is where most of the interesting failures live — a measure that exists in the TMDL but isn't bound to any visual, or a visual bound to a field the TMDL never defined.

## The agentic self-reflection loop

The first version was a single pass: prompt → plan → output. It produced structurally valid but frequently *analytically wrong* reports — a revenue-by-month breakdown that used the wrong date column, a "top customers" chart ranking by row count instead of by revenue.

The fix was a **self-reflection loop**: after the deterministic data inspection stage, the engine compares the generated plan against the *actual* schema and available measures, and feeds the discrepancies back into the planner for a revision pass. If the plan says "group by `order_date`" and the data has `created_at`, `ship_date`, and `invoice_date`, the loop surfaces that ambiguity rather than silently picking one.

It converges in a small bounded number of passes. This is not an agent that runs indefinitely — it's a fixed-point loop with a revision budget and an explicit stopping condition, because an unbounded agent loop in a CI pipeline is a liability.

## MCP server, CLI, and Python API

Three entry points, same engine:

- **CLI** — `nl2pbip` for local use and CI
- **Python API** — for programmatic use
- **MCP stdio server** — so an agent like Claude Code or Cursor can drive report generation directly

The MCP server matters more than it sounds. Once the engine is an MCP server, generating a BI report becomes a tool call an LLM agent can make — which means the request arrives already understood in agent context, and the output can be iterated on in the same conversation. This is where the "agentic" in "agentic Power BI generation" really comes from: not that the engine is an agent, but that it can *be* used by one.

## Engineering discipline

This is a project where I went conservative on the parts that matter:

- **1,082 tests collected, 1,065 passing**, with a CI badge
- **`mypy --strict`** as a separate enforced workflow — the TMDL/PBIR layers are heavily typed because schema errors surface as confusing runtime failures in Power BI otherwise
- **Bandit** security scanning in CI
- **Deterministic output for deterministic inputs** — the same prompt and the same data produce byte-identical output, which is what makes the thing reviewable in a PR

That last point is the one I'd underline. Generative output that isn't reproducible isn't reviewable, and an unreviewable generator in a BI pipeline is a liability you'll discover during an audit.

## Licensing

`nl2pbip` is **commercially licensed** — a deliberate departure from the AGPL default I use elsewhere (py-idp, mrnavax, ns-blowup are all AGPL-3.0-or-later with a commercial option). BI tooling is usually embedded in a product, and AGPL's copyleft obligations would be a blocker for most vendors embedding it. The commercial license is the path for that use case.

## What's next

- **Dax measure validation against a live semantic model** — catching broken DAX at generation time rather than at open time
- **Incremental regeneration** — updating an existing `.pbip` in place instead of regenerating from scratch, so a BI team can hand-edit output and still get the benefit of generation
- **More connectors** — the deterministic inspection stage is the piece that most benefits from breadth (warehouse, files, APIs)

## Where to start

```bash
pip install nl2pbip
nl2pbip --prompt "revenue by month with YoY comparison and top-10 customers" --source ./warehouse
```

You get a directory. Open the `.pbip` in Power BI Desktop, or commit it to Fabric Git.

The repo is [github.com/rollroyces/nl2pbip](https://github.com/rollroyces/nl2pbip) — package on [PyPI](https://pypi.org/project/nl2pbip/). If you run BI in a Git workflow and want to compare notes on TMDL/PBIR generation or the self-reflection loop, open an issue or email me.

— Royce