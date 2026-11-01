---
title: "9 SQL dialects from one grammar: how speaksql transpiles natural language to vendor-specific SQL"
subtitle: "Designing a multi-dialect NL→SQL transpiler that targets Postgres, MySQL, Snowflake, BigQuery, SQL Server, Databricks, DuckDB, SQLite, and SAP HANA — without forking the grammar"
date: 2026-10-15
author: Royce Lam
tags: nlp, sql, transpiler, engineering, language-tooling, llm, applied-ai
target: Medium / dev.to (intended venue - no account live yet)
hero_image: https://raw.githubusercontent.com/rollroyces/speaksql/main/docs/join_graph_example/join_graph.svg
hero_file: assets/speaksql-join-graph.svg
hero_caption: Auto-discovered join graph from a 6-table schema — the foundation speaksql builds its SQL output on.
repo: https://github.com/rollroyces/speaksql
---

# 9 SQL dialects from one grammar

Most NL→SQL systems are built for one dialect. They train a model on Spider (SQLite-flavored) and ship a demo. The first time a user asks *"can I point this at Snowflake?"*, the answer is: not without retraining.

That doesn't match how real teams use SQL. A working data team writes against Postgres, ships ETL to BigQuery, runs analytics on Snowflake, tests on SQLite, and pins a legacy SAP HANA report nobody dares touch. **The query is the same query.** The wire format isn't.

[`speaksql`](https://github.com/rollroyces/speaksql) is a transpiler that targets nine SQL dialects from a single grammar, with no per-dialect retraining. Here's how the design holds up.

## The shape of the problem

At first glance, "NL to vendor-specific SQL" sounds like two problems stacked: natural-language understanding, then SQL generation. In practice the second problem is the harder one for multi-dialect support, because the dialects differ in *weird* ways:

- `LIMIT n` vs. `TOP n` vs. `FETCH FIRST n ROWS ONLY`.
- Identifier quoting: `"name"` (Postgres, BigQuery, SQLite) vs. `` `name` `` (MySQL, Databricks, Snowflake) vs. `[name]` (SQL Server, SAP HANA).
- String concatenation: `||` (most), `CONCAT()` (BigQuery, Snowflake strict), `+` (SQL Server, SAP HANA).
- Date arithmetic: `DATE_ADD(d, INTERVAL 1 DAY)` (MySQL, BigQuery, Databricks) vs. `d + INTERVAL '1 day'` (Postgres, DuckDB, SQLite) vs. `DATEADD(DAY, 1, d)` (SQL Server, SAP HANA).
- NULL semantics: `IS NULL` everywhere, but `NULLS FIRST` / `NULLS LAST` and `COALESCE` semantics differ subtly.
- Reserved keywords: `user` is reserved in some dialects and unreserved in others — affects whether you can use it as a column name unquoted.

If you train one model per dialect, you have a maintenance and quality-control problem. If you train one model and hope, your Postgres output silently misbehaves on Snowflake. The only tractable design is **a single grammar + a per-dialect lowering pipeline**.

## The architecture

`speaksql` has three layers:

1. **Schema understanding.** Given a target database (or an auto-discovered one via the join graph in the hero image), produce a canonical schema graph: tables, columns, types, foreign-key relationships, sample data statistics.

2. **Semantic intent.** Given the natural-language question and the canonical schema, produce a *vendor-neutral* IR — essentially an abstract SQL tree with `Project`, `Filter`, `Join`, `Aggregate`, `Sort`, `Limit` nodes. No dialect-specific tokens appear in this layer.

3. **Dialect lowering.** Take the neutral IR and emit concrete SQL for each target dialect via a small per-dialect template library.

The semantic layer is the interesting one — it's where the model lives. The lowering layer is mostly mechanical: a few hundred templates per dialect, each tagged with which syntactic features it requires. We can validate the templates with property tests that round-trip every generated query through the dialect's actual parser (DuckDB, `pg_query`, `sqlglot`, etc.) — if the parser rejects it, the template is wrong.

## The schema layer is half the work

A surprising amount of effort in NL→SQL goes into *figuring out what tables to ask*. "What's the conversion rate by source?" requires knowing that `orders` has a `source` column, `events` has the same, and they join on `user_id` — none of which is in the natural-language question.

`speaksql` does this with two passes:

- **Static schema analysis.** Read the catalog (or `information_schema`), resolve types, build a column-level embedding from column names + comments + sample values.
- **Join-graph inference.** When foreign keys are missing or ambiguous, infer them from value overlap and naming patterns. The hero image is an actual join graph auto-discovered from a 6-table schema; you can see how it pulls `users`, `events`, `subscriptions`, `invoices`, `plans`, and `coupons` into a coherent shape purely from column-name and value-coverage heuristics.

The semantic layer then queries this graph (literally — it's stored as a small vector index) to find candidate tables and join paths. The model never sees a query like "convert users to events" without already knowing which columns connect them.

## The model layer

The semantic layer is an LLM with a tight prompt: the canonical schema graph, a few worked NL→IR examples, and a strict output schema (JSON with the IR tree). We use a 12-backend pluggable interface so the user can pick the LLM at runtime — same pattern as my [`py-idp`](https://github.com/rollroyces/py-idp) project — and we hit each backend with the same prompt and validate the output.

The output is a small AST, not raw SQL. That choice is what makes the lowering step tractable: instead of "ask the LLM to write Postgres SQL", we ask it to write a normalized intermediate, and we lower that with tested templates. The LLM never has to remember which dialect uses `LIMIT` vs. `TOP`.

A few details that mattered:

- **Strict JSON schema.** The IR has a fixed shape. If the LLM emits something off-schema, we re-prompt with a corrective example. This catches most LLM sloppiness.
- **Two-stage intent.** First pass decides *which tables and which join path*. Second pass decides *which projections, filters, aggregations*. Splitting these means the prompt for each pass is small enough that small models do well.
- **Calibration of confidence.** Each emitted IR carries a per-component confidence (margin in embedding space over candidate columns). The runtime can refuse low-confidence outputs instead of guessing.

## What the dialects cost

The lowering layer is 9 dialects × ~200 templates. It's the part of the codebase that grows linearly with each new dialect we add. A few observations from writing all nine:

- **Postgres and SQLite are easy.** DuckDB is essentially SQLite with better types.
- **MySQL and Databricks are twins** modulo identifier quoting and a few function names.
- **Snowflake and BigQuery are the strict ones** — both reject ambiguous GROUP BY, both require explicit casting in places where other dialects would coerce.
- **SQL Server and SAP HANA are the awkward ones** — `TOP` instead of `LIMIT`, `[brackets]` instead of `"quotes"`, `+` for string concatenation, and a long tail of T-SQL-specific function names (`DATEADD`, `ISNULL`, `IIF`).

The bug surface is the long tail. Every dialect has 2–3 functions that almost-but-not-quite-match the canonical name (e.g., `STR_TO_DATE` in MySQL vs. `TO_DATE` in Postgres vs. `PARSE_DATETIME` in BigQuery vs. `TRY_CAST` in Snowflake). The lowering templates have to handle every cross-mapping or the user hits an opaque syntax error.

## What we ship

The CLI is small:

```
speaksql --schema warehouse.sql --dialect snowflake --question "active users by signup month"
```

It returns the IR, the lowered SQL, and a confidence score. For multi-dialect output, pass `--dialect all` and get a markdown report with one query per dialect, each verified to parse by the dialect's own parser.

The validation harness — the part I'm proudest of — runs every supported dialect's parser over a corpus of generated queries. If `pg_query` rejects a Postgres output, the build fails. This is how we keep the dialect templates honest.

## Limitations

Honest list of what this design *doesn't* solve:

- **Vendor-specific functions.** If you ask for a query that requires `JSON_TABLE_EXTRACT_PATH_TEXT` in Snowflake or `PIVOT` in SQL Server, you need to extend the lowering layer. The semantic layer can express intent, but the lowering has to know the dialect-specific function names.
- **Performance.** A generated Snowflake query can be slow if it doesn't use cluster keys. We don't try to optimize for execution cost — that's a separate layer.
- **Schema drift.** If the database schema changes between NL generation and execution, you'll get a runtime error. Standard SQL tooling problem.
- **Ambiguous questions.** "Top users" without a metric or a time window — we ask. We could guess, but guessing wrong silently is worse than asking.

## What's next

Three things I'm actively working on:

1. **Stored-procedure lowering.** Most NL→SQL systems stop at `SELECT`. We want `INSERT`/`UPDATE`/`CREATE TABLE AS` with the same multi-dialect discipline.
2. **Type-aware templates.** The lowering templates currently use string substitution; a typed IR would let us catch type mismatches before runtime.
3. **Streaming-friendly dialects.** Some dialects have window-function semantics (BigQuery's `QUALIFY`, Postgres's `DISTINCT ON`) that don't fit the IR cleanly. Refactoring the IR to support them properly is on the roadmap.

The repo is [github.com/rollroyces/speaksql](https://github.com/rollroyces/speaksql). If you're shipping NL→SQL in production and want to compare notes on dialect coverage or validation harnesses, open an issue or email me.

— Royce