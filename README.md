# Writing

Technical writing on **ML for biology**, **applied AI tooling**, and **formal methods** — by [Royce Lam](https://github.com/rollroyces).

Every post is drafted here first — this repo is where they are written, revised, and version-controlled. Published versions go out to Medium and dev.to once those accounts are live; until then this repo is the only canonical home.

## Posts

| # | Title | Lane | Words | Repo |
| --- | --- | --- | --- | --- |
| 01 | [Multi-modal cfDNA fragmentomics for ultra-early cancer detection](posts/01-deepcatch-multi-modal-cfdna-fragmentomics.md) | Bioinformatics | 1,305 | [deepcatch](https://github.com/rollroyces/deepcatch) |
| 02 | [9 SQL dialects from one grammar](posts/02-speaksql-9-sql-dialects-one-grammar.md) | Applied AI / NL→SQL | 1,371 | [speaksql](https://github.com/rollroyces/speaksql) |
| 03 | [100 theorems toward Navier–Stokes regularity in Lean 4](posts/03-ns-blowup-100-lemmas-navier-stokes-lean-4.md) | Formal methods | 1,300 | [ns-blowup](https://github.com/rollroyces/ns-blowup) |
| 04 | [Production document AI](posts/04-py-idp-production-document-ai.md) | Applied AI | 1,303 | [py-idp](https://github.com/rollroyces/py-idp) |
| 05 | [Agentic Power BI generation](posts/05-nl2pbip-agentic-power-bi-generation.md) | Applied AI / BI | 1,374 | [nl2pbip](https://github.com/rollroyces/nl2pbip) |
| 06 | [Eleven tools and twelve model adapters](posts/06-mrnavax-mrna-cancer-therapy-toolkit.md) | Bioinformatics / foundation models | 1,315 | [mrnavax](https://github.com/rollroyces/mrnavax) |
| 07 | [Turning exit interviews into attrition signals](posts/07-employee-voice-analytics-hr-nlp.md) | NLP / HR analytics | 1,328 | [employee-voice-analytics](https://github.com/rollroyces/employee-voice-analytics) |

**Total: 9,296 words across 7 posts.**

### Lane coverage

| Lane | Posts |
| --- | --- |
| Bioinformatics / ML-for-biology | 01, 06 |
| Applied AI & production systems | 02, 04, 05 |
| Formal methods | 03 |
| NLP & analytics | 07 |

## Structure

```
posts/          # post drafts (YAML front matter + markdown body)
assets/         # hero images, byte-identical to the live URLs in each post's front matter
```

Each post's front matter carries:

| Field | Purpose |
| --- | --- |
| `title` / `subtitle` | Post headline and deck |
| `date` | Publication date |
| `author` | Byline |
| `tags` | Topic tags — reuse for Medium and dev.to tags |
| `target` | Where the post is intended to run |
| `hero_image` | Permanent hosted URL for the hero image |
| `hero_file` | Matching local copy in `assets/` |
| `hero_caption` | Caption text and alt text |
| `repo` | The GitHub repo the post is about |

## Publishing

These posts are not published yet — no Medium or dev.to account is currently active, so nothing here links to a platform that doesn't exist. When an account is set up, publish as follows.

Both Medium and dev.to accept the markdown body directly.

1. Copy the body of the post file (everything after the front matter).
2. Upload the hero from `assets/` — or use the `hero_image` URL, which is already permanently hosted.
3. Use `tags` for the platform's tag field.

dev.to supports `dev.to` front matter natively. For Medium, paste the body — the structure is plain markdown with no platform-specific syntax.

## Hero images

Five heroes are real artifacts from the projects themselves. Two were generated for this series:

- `assets/ns-blowup-hero.jpg` — generated; flow streamlines and a proof dependency graph
- `assets/eva-hero.jpg` — generated; feedback streams resolving into a risk gauge and topic clusters

Both generated images are also committed to their source repos (`ns-blowup/docs/assets/hero.jpg`, `employee-voice-analytics/docs/assets/hero.jpg`) so the `hero_image` URLs stay stable.

---

[Royce Lam](https://github.com/rollroyces) · [orcid.org/0009-0008-9113-769X](https://orcid.org/0009-0008-9113-769X) · [roycelam@umich.edu](mailto:roycelam@umich.edu)
