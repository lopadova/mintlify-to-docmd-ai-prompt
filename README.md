# mintlify-to-docmd-ai-prompt

> **An AI agent prompt that fully migrates a Mintlify (MDX) documentation site to [docmd](https://docs.docmd.io) — including component conversion, icon remapping, semantic search, Cloudflare Pages deploy, and Claude skills/rules.**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Prompt version](https://img.shields.io/badge/prompt-v1.0-green.svg)](#the-prompt-at-a-glance)

---

## Table of Contents

1. [Summary](#summary)
2. [Why](#why)
3. [What the Prompt Does](#what-the-prompt-does)
4. [How to Use](#how-to-use)
5. [Prompt Sections Reference](#prompt-sections-reference)
   - [0 · Values to Fill In](#0--values-to-fill-in)
   - [1 · Pre-migration Analysis](#1--pre-migration-analysis)
   - [2 · Target Structure](#2--target-structure)
   - [3 · Install & Versions](#3--install--versions)
   - [4 · Mintlify → docmd Mapping](#4--mintlify--docmd-mapping)
   - [5 · Conversion Script](#5--conversion-script)
   - [6 · Conversion Rules (edge cases)](#6--conversion-rules-edge-cases)
   - [7 · docmd.config.json](#7--docmdconfigjson)
   - [8 · Semantic Search](#8--semantic-search)
   - [9 · Homepage Banner](#9--homepage-banner)
   - [10 · Container Syntax & Brand](#10--container-syntax--brand)
   - [11 · Documentation Structure Standard](#11--documentation-structure-standard)
   - [12 · CI Guard Script](#12--ci-guard-script)
   - [13 · Build & Verification](#13--build--verification)
   - [14 · Deploy: Cloudflare Pages](#14--deploy-cloudflare-pages)
   - [15 · Gotchas](#15--gotchas)
   - [16 · Deliverables — Skill + Rule](#16--deliverables--skill--rule)
   - [17 · Acceptance Criteria](#17--acceptance-criteria)
6. [Component Mapping Quick Reference](#component-mapping-quick-reference)
7. [Icon Mapping Quick Reference](#icon-mapping-quick-reference)
8. [Contributing](#contributing)
9. [License](#license)

---

## Summary

**mintlify-to-docmd-ai-prompt** is a ready-to-paste AI agent prompt (designed for Claude / Copilot Coding Agent) that turns a full Mintlify documentation site into a [docmd](https://docs.docmd.io) static site — automatically.

It covers the entire migration lifecycle in one shot:

| Phase | What it automates |
|---|---|
| **Audit** | Inventory of `.mdx` files, MDX component frequency, custom imports, FontAwesome icons |
| **Convert** | Deterministic Node.js script translating every Mintlify component to its docmd container equivalent |
| **Configure** | `docmd.config.json` scaffolded from the existing `docs.json` (navigation, footer, plugins, theme) |
| **Search** | Semantic search via `docmd-search` + ONNX embeddings, CI-safe (no interactive wizard) |
| **CI guard** | `check-no-mdx-tags.mjs` fails the build if any MDX tag survives conversion |
| **Deploy** | Cloudflare Pages Git-integration (primary) + GitHub Actions fallback |
| **Sync rules** | Claude skill + hard rule that keeps docs in sync with every future feature change |

---

## Why

### The problem

Mintlify is a fantastic hosted docs platform, but it has real drawbacks:

- **Vendor lock-in** — your docs live in `.mdx` + a proprietary `docs.json`, tied to Mintlify's renderer.
- **MDX complexity** — JSX components (`<Card>`, `<Tabs>`, `<Steps>`, `<Accordion>`, …) are not portable Markdown; they break any other static-site generator.
- **Icon system** — Mintlify uses **FontAwesome** icon names; docmd uses **Lucide**. A naive rename would silently break every icon.
- **Conversion is NOT a rename** — `.mdx → .md` is only the first step. Every component must be translated to the target syntax, indentation matters (especially for nested `Steps`), and one wrong closing `:::` makes the entire page render broken.

### Why docmd?

[docmd](https://docs.docmd.io) is a lightweight, open static-site generator built around plain Markdown with a clean container syntax (`::: callout`, `::: tabs`, `::: steps`, …). It is:

- **Portable** — pure Markdown files, no framework lock-in.
- **Fast to build** — single `npm run build`, deploys anywhere (Cloudflare Pages, Vercel, GitHub Pages, …).
- **AI-friendly** — generates `llms.txt` and a full-context LLM file out of the box.
- **Search-capable** — semantic vector search computed at build time (ONNX + HuggingFace), no server needed.

### Why an AI prompt instead of a plain script?

A plain script can handle the mechanical substitutions. An AI agent can:

1. **Interpret context** — understand that `<ParamField>` has no docmd equivalent and rewrite it as idiomatic Markdown.
2. **Fill in gaps** — map every undocumented FontAwesome icon to the closest Lucide icon.
3. **Keep docs in sync** — with the delivered Claude skill and rule, every future feature change automatically triggers a doc update.
4. **Generate deliverables** — produce `MIGRATION_REPORT.md`, the skill, the rule, and a fully configured `docmd.config.json` in one pass.

---

## What the Prompt Does

Paste the prompt into a Claude (or compatible) coding agent session pointing at a repo that contains a Mintlify `docs-site/` folder. The agent will:

1. **Inventory** every `.mdx` file and extract all MDX component usages.
2. **Scaffold** the target structure inside `<DOCS_DIR>/` (`docmd.config.json`, `package.json`, scripts, `assets/`, `docs/`).
3. **Run** `scripts/migrate-from-mintlify.mjs` — a deterministic Node.js converter that:
   - Translates all standard Mintlify components to docmd containers.
   - Remaps FontAwesome icons to Lucide.
   - Produces `MIGRATION_REPORT.md` listing per-file conversions and any unmapped `TODO` items.
4. **Configure** `docmd.config.json` — navigation, plugins (search, git, SEO, sitemap, mermaid, math, llms), footer, and brand CSS.
5. **Pin** the semantic-search model config (`Xenova/all-MiniLM-L6-v2`) so CI never blocks on an interactive wizard.
6. **Verify** with `npm run check && npm run build` and confirms `_site/index.html` exists.
7. **Deploy** — instructs Cloudflare Pages Git-integration setup (or GitHub Actions fallback).
8. **Generate** `.claude/skills/docmd-docs/SKILL.md` and `.claude/rules/rule-docmd-docs-sync.md`.

---

## How to Use

### 1. Copy the prompt

The full prompt lives in this repository. Copy the entire contents of [PROMPT.md](#the-prompt-at-a-glance) (or the raw text below).

> **Tip:** The prompt is self-contained. You can also inline it directly as the first message to an agent.

### 2. Fill in the placeholder values

Before running, substitute the `<PLACEHOLDER>` tokens at the top of the prompt:

| Placeholder | Example |
|---|---|
| `<PACKAGE_NAME>` | `my-project` |
| `<PROJECT_TITLE>` | `My Project Docs` |
| `<ONE_LINE_DESC>` | `Comprehensive docs for My Project` |
| `<REPO_URL>` | `https://github.com/org/my-project` |
| `<SITE_URL>` | `https://docs.my-project.io` |
| `<BRAND_HEX>` | `#6366f1` |
| `<BANNER_URL>` | `https://cdn.my-project.io/banner.png` |
| `<DOCS_DIR>` | `docs-site` |
| `<CF_PROJECT>` | `my-project-docs` |
| `<AUTHOR_NAME>` | `Lorenzo` |
| `<ORG>` | `MyOrg` |
| `<ORG_URL>` | `https://my-org.io` |
| `<LICENSE>` | `MIT` |

### 3. Start the agent

Paste the filled-in prompt into your AI coding agent (Claude Sonnet recommended) and let it run. The agent will execute shell commands, create files, and verify the build in the same session.

### 4. Review `MIGRATION_REPORT.md`

After the script runs, check `MIGRATION_REPORT.md` for:

- **Unmapped icons** — icons that had no Lucide equivalent (listed as `TODO`).
- **Unconverted MDX tags** — components like `<ParamField>` that need manual rewriting.

Fix any `TODO` items manually, then re-run `npm run check && npm run build`.

### 5. Deploy

Follow the Cloudflare Pages Git-integration instructions in [§14](#14--deploy-cloudflare-pages) or use the GitHub Actions fallback.

---

## Prompt Sections Reference

### 0 · Values to Fill In

Placeholder tokens at the top of the prompt. Fill these before handing the prompt to an agent — every section references them by `<TOKEN>` name.

---

### 1 · Pre-migration Analysis

The agent runs a shell inventory **before** any conversion:

```bash
find . -name "*.mdx" | sort; cat docs.json
grep -rhoE "<[A-Z][A-Za-z]+" --include=*.mdx . | sort | uniq -c | sort -rn
grep -rnE "^import |^export " --include=*.mdx .
grep -rhoE "<(ParamField|ResponseField|Frame|CodeGroup|Tabs|…)" --include=*.mdx .
grep -rhoE 'icon="[a-z-]+"' --include=*.mdx . | sort | uniq -c
```

This classifies the migration complexity:
- **~Lossless** — almost-Markdown + standard components → automated conversion.
- **Manual work needed** — custom JSX imports, `ParamField`/`ResponseField`, OpenAPI playgrounds.

---

### 2 · Target Structure

```
<DOCS_DIR>/
  docmd.config.json
  package.json
  .node-version                # "20"
  .gitignore
  .docmd-search/config.json    # pinned embedding model (committed)
  assets/{favicon.svg,custom.css}
  scripts/{migrate-from-mintlify.mjs,check-no-mdx-tags.mjs}
  docs/**/*.md                 # converted content; route = file tree
  MIGRATION_REPORT.md
  _site/                       # build output (git-ignored)
```

All content goes into `docs/` — the file tree becomes the URL tree. The original `docs.json` and any emptied Mintlify folders are removed at the end.

---

### 3 · Install & Versions

```bash
npm install -D @docmd/core
npm install docmd-search
npm install -D @huggingface/transformers onnxruntime-node
```

`package.json` scripts: `dev`, `build`, `check`. Node version pinned to `20` via `.node-version`. `"type": "module"`.

---

### 4 · Mintlify → docmd Mapping

See [Component Mapping Quick Reference](#component-mapping-quick-reference) below.

---

### 5 · Conversion Script

`scripts/migrate-from-mintlify.mjs` — a structural (non-naive-regex) Node.js script that:

- Handles nested containers correctly via `dedent` / `indent` helpers.
- Converts all standard Mintlify components to docmd containers.
- Remaps FontAwesome → Lucide icons.
- Writes output to `docs/**.md`, deletes source `.mdx` files, and produces `MIGRATION_REPORT.md`.
- Maps `introduction.mdx` → `docs/index.md` (required root page).

**Run:** `node scripts/migrate-from-mintlify.mjs`

> ⚠️ The script **deletes** the source `.mdx` files after converting them. To re-run after edits, restore them first: `git checkout -- '<DOCS_DIR>/*.mdx'`

---

### 6 · Conversion Rules (edge cases)

Key rules baked into the conversion logic:

- **Dedent before, re-indent after** — captures container bodies without consuming leading indentation, then re-indents `Steps` items to exactly **3 spaces** so nested code-fences and callouts stay inside the list item.
- **`<AccordionGroup>`** → wrapper removed; each `<Accordion>` becomes an independent `::: collapsible` (docmd has no exclusive-accordion behavior).
- **`<Card href="…">`** → **never** `::: button` (see gotcha #2); use `[Open →](href)` instead.
- **`introduction.mdx`** → `docs/index.md` (route `/`, mandatory root page).
- **Non-standard components** (`<ParamField>`, `<ResponseField>`, `<CodeGroup>`, `<Frame>`, custom JSX) — no automatic mapping; rewrite as plain Markdown and log as `TODO`.

---

### 7 · docmd.config.json

Full scaffold generated from the existing `docs.json`:

- Navigation groups → `navigation[]` (the only source of truth for the sidebar).
- Navbar links + footer socials → `footer.columns`.
- `colors.primary` → `assets/custom.css` (`--docmd-color-primary`).
- All plugins enabled: `search` (semantic), `git`, `seo`, `sitemap`, `mermaid`, `math`, `llms`.

---

### 8 · Semantic Search

`plugins.search.semantic: true`. Embeddings computed at **build time** using ONNX; the browser receives only Int8 vectors (keyword + cosine, fully client-side — no server or model in the browser).

**CI-safe setup** — commit `.docmd-search/config.json` to skip the interactive wizard:

```json
{
  "model": "Xenova/all-MiniLM-L6-v2",
  "chunkSize": 512,
  "chunkOverlap": 64,
  "incremental": true,
  "topK": 10
}
```

Use `Xenova/multilingual-e5-small` for multilingual docs.

`.gitignore` pattern:
```
.docmd-search/*
!.docmd-search/config.json
```

---

### 9 · Homepage Banner

If `<BANNER_URL>` is provided, the agent prepends this to `docs/index.md` (below the frontmatter):

```markdown
![<PROJECT_TITLE> banner](<BANNER_URL>)
```

---

### 10 · Container Syntax & Brand

Container reference:

| Syntax | Renders as |
|---|---|
| `::: callout {info\|tip\|warning\|danger\|success}` | Colored callout box |
| `::: tabs` + `== tab "X"` | Tabbed content |
| `::: steps` (numbered list, bold titles) | Step-by-step guide |
| `::: collapsible "Title"` (`open` to expand by default) | Expandable section |
| `::: grids` / `::: grid` / `::: card "T" icon:lucide` | Card grid |
| ` ```mermaid ` | Diagram |
| `$…$` / `$$…$$` | KaTeX math |

Brand CSS: `assets/custom.css` sets `--docmd-color-primary`, `--color-primary`, `--link-color` to `<BRAND_HEX>`.

---

### 11 · Documentation Structure Standard

Recommended sidebar groups: **Get Started, Guides, Concepts & Theory, Architecture, Best Practices, Operations, Reference**.

Each deep page follows this template:
> **Motivation → Theory** (KaTeX formulas, academic tone) **→ Design + Mermaid diagram** (flowchart / sequenceDiagram) **→ Data model / contract → ADR** (Problem → Decision → Trade-off, inside `::: collapsible`) **→ Worked example → Gotcha** (`::: callout warning`)

---

### 12 · CI Guard Script

`scripts/check-no-mdx-tags.mjs` fails with exit code 1 if any MDX-style component tag (`<[A-Z][A-Za-z0-9]*`) survives in `docs/**.md`.

Run with `npm run check`.

---

### 13 · Build & Verification

```bash
npm install && npm run check && npm run build
```

Checklist:
- [ ] `_site/index.html` exists (otherwise `docs/index.md` is missing).
- [ ] `npm run check` exits 0 (zero MDX tags remaining).
- [ ] Zero `:::` as visible text (only inside `class="docmd-container …"`).
- [ ] KaTeX renders correctly; no false positives on `$variables` inside code blocks.
- [ ] `_site/.docmd-search/manifest.json`, `llms.txt`, `sitemap.xml` present.

---

### 14 · Deploy: Cloudflare Pages

**Option A (primary — no API key needed):** Cloudflare Pages Git integration.

| Setting | Value |
|---|---|
| Root | `<DOCS_DIR>` |
| Build command | `npm run build` |
| Output directory | `_site` |
| Node version | via `.node-version` (20) |

Connect `<SITE_URL>` as a Custom Domain. The ONNX semantic-search build runs fine on the CF Linux image. Use `package-lock.json` v3 (cross-platform) to avoid optional-dep resolution issues.

**Option B (fallback):** GitHub Actions `workflow_dispatch` + `wrangler pages deploy _site --project-name=<CF_PROJECT>` (requires `CLOUDFLARE_API_TOKEN` + `CLOUDFLARE_ACCOUNT_ID` secrets). Use only if Option A's ONNX build fails.

---

### 15 · Gotchas

| # | Gotcha |
|---|---|
| 1 | **Root page required** — `docs/index.md` must exist (route `/`); missing it gives "Root index.html not found". |
| 2 | **`::: button` is not a paired block** — the closing `:::` renders as visible text. Use `[Open →](href)` in cards instead. |
| 3 | **Steps indentation** — de-indent body to column 0, then re-indent to **3 spaces**; nested code-fences and callouts stay inside the list item. |
| 4 | **KaTeX false positives** — check `class="katex"` on pages with `$variables` in code blocks. |
| 5 | **Re-running the script deletes `.mdx` files** — restore originals first: `git checkout -- '<DOCS_DIR>/*.mdx'`. |
| 6 | **`AccordionGroup`** → sequential collapsibles (no exclusive-open behavior — content intact, UX slightly different). |
| 7 | **Lockfile v3 cross-platform** — don't commit a lock that only resolves optional deps for your OS. |

---

### 16 · Deliverables — Skill + Rule

The agent creates two Claude artifacts:

**A) Skill** — `.claude/skills/docmd-docs/SKILL.md`
Activated whenever you work in `<DOCS_DIR>/`, add pages, touch navigation/plugins, or re-run the migration. Contains: layout, commands, container syntax table, Mintlify→docmd mapping, Lucide icon mapping, plugin config, semantic search setup, footer/branding, CF deploy instructions, documentation structure standard, and all gotchas.

**B) Hard sync rule** — `.claude/rules/rule-docmd-docs-sync.md`
Binding rule: **every** user-facing feature addition or substantial README update **must** update the corresponding docmd page in the same work unit (and register it in `navigation[]` if new). Declares when NOT needed (internal refactors, tooling fixes, cosmetics). Pre-close check: `npm run check && npm run build` must be green.

---

### 17 · Acceptance Criteria

- [ ] `npm run check` green (zero MDX tags) and `npm run build` green; `_site/index.html` present.
- [ ] Zero `:::` as visible text; all icons mapped (clean report) or TODOs resolved manually.
- [ ] All plugins active: semantic search, git, SEO, sitemap, mermaid, math, llms.
- [ ] Semantic index generated + paraphrased query test returns relevant results.
- [ ] Home banner (if provided), footer with author credit, brand color, navigation 1:1 from `docs.json`.
- [ ] `MIGRATION_REPORT.md` generated; old `docs.json` and empty folders removed.
- [ ] CF Pages Git integration configured → live deploy on `<SITE_URL>`.
- [ ] Skill `docmd-docs` and hard auto-sync rule created.

---

## Component Mapping Quick Reference

| Mintlify component | docmd syntax |
|---|---|
| `<Note>…</Note>` | `::: callout info` … `:::` |
| `<Tip>…</Tip>` | `::: callout tip` … `:::` |
| `<Warning>…</Warning>` | `::: callout warning` … `:::` |
| `<Info>…</Info>` | `::: callout info` … `:::` |
| `<Check>…</Check>` | `::: callout success` … `:::` |
| `<Danger>…</Danger>` | `::: callout danger` … `:::` |
| `<Tabs>` / `<Tab title="X">` | `::: tabs` + `== tab "X"` + `:::` |
| `<Steps>` / `<Step title="X">` | `::: steps` + `N. **X**` (body indented 3 spaces) + `:::` |
| `<AccordionGroup>` / `<Accordion title="X">` | wrapper removed; `::: collapsible "X"` … `:::` (sequential) |
| `<CardGroup>` / `<Card title icon href>` | `::: grids` › `::: grid` › `::: card "title" icon:lucide` + `[Open →](href)` |
| `<code>x</code>` | `` `x` `` |
| `&nbsp;` | regular space |
| ` ```mermaid ` | unchanged |
| frontmatter `icon:` | removed from page (icon lives in `navigation[]`) |

---

## Icon Mapping Quick Reference

FontAwesome (Mintlify) → Lucide (docmd):

| FontAwesome | Lucide |
|---|---|
| `function` | `square-function` |
| `wave-pulse` | `activity` |
| `layer-group` | `layers` |
| `traffic-light` | `traffic-cone` |
| `shield-halved` | `shield` |
| `scale-balanced` | `scale` |
| `diagram-project` | `workflow` |
| `sitemap` | `network` |
| `gear` | `settings` |
| `table-list` | `table` |
| `seedling` | `sprout` |
| `robot` | `bot` |
| `network-wired` | `network` |
| `code-compare` | `git-compare` |
| `chart-line` | `trending-up` |
| `chart-line-down` | `trending-down` |
| `bolt` | `zap` |
| `book-bookmark` | `book-marked` |
| `book-open` | `book-open` |
| `lightbulb` | `lightbulb` |
| `rocket` | `rocket` |
| `bug` | `bug` |
| `lock` | `lock` |
| `lock-open` | `lock-open` |
| `cloud` | `cloud` |
| `download` | `download` |
| `plug` | `plug` |
| `server` | `server` |
| `sigma` | `sigma` |
| `ruler-combined` | `ruler` |
| `play` | `play` |
| `file-code` | `file-code` |

> Icons not in this table are passed through unchanged and flagged in `MIGRATION_REPORT.md` for manual review.

---

## Contributing

Contributions, icon mapping additions, and new component translations are welcome. Please open an issue or pull request.

---

## License

[MIT](LICENSE) © 2026 Lorenzo
