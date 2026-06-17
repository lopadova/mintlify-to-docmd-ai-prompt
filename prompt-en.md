# Task: migrate documentation from Mintlify (MDX) to docmd (+ Cloudflare Pages deploy)

  You are an agent that must **convert** existing documentation from **Mintlify**
  (`.mdx` files + `docs.json`) to **docmd** (https://docs.docmd.io), a static site
  generator based on Markdown. The conversion is NOT a simple `.mdx`→`.md` rename:
  MDX components must be translated into docmd containers and icons remapped. ALWAYS
  verify with a real build; never take anything for granted.

  ## 0. Values to fill in

  - `<PACKAGE_NAME>` / `<PROJECT_TITLE>` / `<ONE_LINE_DESC>`
  - `<REPO_URL>` · `<SITE_URL>` · `<BRAND_HEX>` · `<BANNER_URL>` (if present)
  - `<DOCS_DIR>` = folder currently containing the Mintlify docs (e.g. `docs-site`)
  - `<CF_PROJECT>` = Cloudflare Pages project name
  - `<AUTHOR_NAME>` / `<ORG>` / `<ORG_URL>` / `<LICENSE>`

  ## 1. Pre-migration analysis (do NOT skip)

  Before converting, take an inventory to size the work and discover edge cases:

  ```bash
  cd <DOCS_DIR>
  # list files and config
  find . -name "*.mdx" | sort ; cat docs.json
  # MDX component frequency
  grep -rhoE "<[A-Z][A-Za-z]+" --include=*.mdx . | sort | uniq -c | sort -rn
  # custom import/JSX (high risk if present)
  grep -rnE "^import |^export " --include=*.mdx .
  # "difficult" components to review manually
  grep -rhoE "<(ParamField|ResponseField|Frame|Icon|CodeGroup|Tabs|Accordion|Steps|CardGroup|Columns|Expandable|Snippet)" --include=*.mdx . | sort | uniq -c
  # icons used (FontAwesome → must be remapped to Lucide)
  grep -rhoE 'icon="[a-z-]+"' --include=*.mdx . | sort | uniq -c
  ```

  Classify: if it is nearly-Markdown + standard components → conversion ~lossless. If
  there are `import`/custom JSX or `ParamField`/`ResponseField`/OpenAPI playground →
  those parts must be rewritten manually in explicit Markdown (see §6, final rule).

  ## 2. Target structure (in-place inside <DOCS_DIR>)

  ```
  <DOCS_DIR>/
    docmd.config.json
    package.json
    .node-version                # "20"
    .gitignore
    .docmd-search/config.json    # pinned embedding model (COMMITTED)
    assets/{favicon.svg,custom.css}
    scripts/{migrate-from-mintlify.mjs, check-no-mdx-tags.mjs}
    docs/**/*.md                 # output: converted .mdx files, route = file tree
    MIGRATION_REPORT.md          # per-file conversion log
    _site/                       # build output (git-ignored)
  ```

  Content goes in `docs/` (routes unchanged: `docs/guides/x.md` → `/guides/x`).
  Remove at the end: the old Mintlify `docs.json` and any emptied folders.

  ## 3. Install & versions

  ```bash
  npm install -D @docmd/core            # VERIFY the actual version with `npm view @docmd/core version`
  npm install docmd-search
  npm install -D @huggingface/transformers onnxruntime-node
  ```

  `package.json` scripts: `dev: docmd dev`, `build: docmd build`,
  `check: node scripts/check-no-mdx-tags.mjs`. `.node-version` = `20`. `"type":"module"`.

  ## 4. Mintlify → docmd mapping

  | Mintlify component | docmd | Notes |
  |---|---|---|
  | `<Note>` | `::: callout info` … `:::` | |
  | `<Tip>` | `::: callout tip` | |
  | `<Warning>` | `::: callout warning` | |
  | `<Info>` | `::: callout info` | |
  | `<Check>` | `::: callout success` | |
  | `<Danger>` | `::: callout danger` | |
  | `<Tabs>`/`<Tab title="X">` | `::: tabs` + `== tab "X"` + `:::` | |
  | `<Steps>`/`<Step title="X">` | `::: steps` + numbered list `N. **X**` (body indented **3 spaces**) + `:::` | |
  | `<AccordionGroup>`/`<Accordion title="X">` | `::: collapsible "X"` … `:::` **sequential** | docmd has NO exclusive accordion |
  | `<CardGroup>`/`<Card title icon href>` | `::: grids`›`::: grid`›`::: card "title" icon:lucide` + `[Open →](href)` ›`:::`›`:::`›`:::` | NO `::: button` (see gotchas) |
  | `<code>x</code>` | `` `x` `` | |
  | `&nbsp;` | regular space | |
  | ` ```mermaid ` | unchanged | mermaid plugin |
  | frontmatter `icon:` | removed from page | icon lives in `navigation[]` |

  **Icons**: Mintlify uses **FontAwesome**, docmd uses **Lucide** → must be **remapped**
  (not copied). Examples: `function`→`square-function`, `wave-pulse`→`activity`,
  `layer-group`→`layers`, `shield-halved`→`shield`, `scale-balanced`→`scale`,
  `diagram-project`→`workflow`, `sitemap`→`network`, `gear`→`settings`, `bolt`→`zap`,
  `robot`→`bot`, `seedling`→`sprout`, `code-compare`→`git-compare`,
  `chart-line`→`trending-up`, `book-bookmark`→`book-marked`, `traffic-light`→`traffic-cone`.
  Extend the table for every icon found in §1; unmapped ones pass through unchanged but
  must be flagged in the report.

  ## 5. Conversion script (deterministic) — `scripts/migrate-from-mintlify.mjs`

  Use a structural script (not naive regex) that handles nesting. Run it from the
  `<DOCS_DIR>` root: it reads the `.mdx` files, writes to `docs/**.md`, deletes the
  originals, and produces `MIGRATION_REPORT.md`.

  ```js
  import { readFileSync, writeFileSync, readdirSync, statSync, mkdirSync, rmSync } from 'node:fs';
  import { join, dirname, relative, sep } from 'node:path';

  const ROOT = process.cwd();
  const OUT = join(ROOT, 'docs');

  // Extend with ALL icons found in §1 (FontAwesome -> Lucide, kebab-case).
  const ICON_MAP = {
    'function':'square-function','wave-pulse':'activity','layer-group':'layers','file-code':'file-code',
    'traffic-light':'traffic-cone','shield-halved':'shield','scale-balanced':'scale','diagram-project':'workflow',
    'sitemap':'network','sigma':'sigma','ruler-combined':'ruler','play':'play','gear':'settings','table-list':'table',
    'server':'server','seedling':'sprout','plug':'plug','lock-open':'lock-open','robot':'bot','network-wired':'network',
    'lock':'lock','download':'download','code-compare':'git-compare','cloud':'cloud','chart-line-down':'trending-down',
    'chart-line':'trending-up','bolt':'zap','book-open':'book-open','rocket':'rocket','lightbulb':'lightbulb',
    'book-bookmark':'book-marked','bug':'bug',
  };

  const report = []; const unmappedIcons = new Set();
  const mapIcon = fa => ICON_MAP[fa] ?? (unmappedIcons.add(fa), fa);

  function dedent(text){
    const lines = text.replace(/\t/g,'    ').split('\n');
    let min = Infinity;
    for (const l of lines){ if(l.trim()==='') continue; const m=l.match(/^ */)[0].length; if(m<min) min=m; }
    if(!isFinite(min)) min=0;
    return lines.map(l=>l.slice(min)).join('\n').replace(/^\n+/,'').replace(/\s+$/,'');
  }
  const indent = (text,n)=> text.split('\n').map(l=> l.trim()===''?'':' '.repeat(n)+l).join('\n');

  const CALLOUT_TYPE = { Note:'info', Tip:'tip', Warning:'warning', Info:'info', Check:'success', Danger:'danger' };
  function convertCallouts(text, log){
    let out = text;
    for (const [tag,type] of Object.entries(CALLOUT_TYPE)){
      out = out.replace(new RegExp(`<${tag}>([\\s\\S]*?)<\\/${tag}>`,'g'), (_,inner)=>{
        log && log.push(`OK: <${tag}> -> ::: callout ${type}`);
        return `::: callout ${type}\n${dedent(inner)}\n:::`;
      });
    }
    return out;
  }

  function convert(src, relPath){
    const log = []; let body = src;
    // frontmatter: keep title/description, remove `icon`
    const fm = body.match(/^---\n([\s\S]*?)\n---\n?/);
    let front = '';
    if (fm){ front = `---\n${fm[1].split('\n').filter(l=>!/^icon:\s*/.test(l)).join('\n')}\n---\n`; body = body.slice(fm[0].length); }
    // inline
    body = body.replace(/<code>([\s\S]*?)<\/code>/g,(_,c)=>'`'+c.trim()+'`').replace(/&nbsp;/g,' ');
    // Tabs
    body = body.replace(/<Tabs>([\s\S]*?)<\/Tabs>/g,(_,inner)=>{
      const tabs=[...inner.matchAll(/<Tab\s+title="([^"]*)"\s*>([\s\S]*?)<\/Tab>/g)];
      log.push(`OK: <Tabs>(${tabs.length}) -> ::: tabs`);
      return `::: tabs\n\n`+tabs.map(([,t,c])=>`== tab "${t}"\n${convertCallouts(dedent(c),log)}`).join('\n\n')+`\n\n:::`;
    });
    // Steps (3-space indent + nested callouts)
    body = body.replace(/<Steps>([\s\S]*?)<\/Steps>/g,(_,inner)=>{
      const steps=[...inner.matchAll(/<Step\s+title="([^"]*)"\s*>([\s\S]*?)<\/Step>/g)];
      log.push(`OK: <Steps>(${steps.length}) -> ::: steps`);
      return `::: steps\n\n`+steps.map(([,t,c],i)=>{ const b=convertCallouts(dedent(c),log); return `${i+1}. **${t}**`+(b?'\n'+indent(b,3):'');
  }).join('\n\n')+`\n\n:::`;
    });
    // Accordion -> sequential collapsibles
    body = body.replace(/<AccordionGroup>([\s\S]*?)<\/AccordionGroup>/g,(_,i)=>i);
    body = body.replace(/<Accordion\s+title="([^"]*)"\s*>([\s\S]*?)<\/Accordion>/g,(_,t,c)=>{
      log.push(`OK: <Accordion "${t}"> -> ::: collapsible`); return `::: collapsible "${t}"\n${convertCallouts(dedent(c),log)}\n:::`;
    });
    // Cards -> grids/grid/card (+ markdown link, NOT button)
    const buildCard=(attrs,content)=>{
      const g=(re)=>(attrs.match(re)||[, ''])[1];
      const title=g(/title="([^"]*)"/), iconFa=g(/icon="([^"]*)"/), href=g(/href="([^"]*)"/);
      const icon=iconFa?` icon:${mapIcon(iconFa)}`:''; const desc=convertCallouts(dedent(content),log);
      const btn=href?`\n\n[Open →](${href})`:'';
      return `::: card "${title}"${icon}\n${desc}${btn}\n:::`;
    };
    body = body.replace(/<CardGroup[^>]*>([\s\S]*?)<\/CardGroup>/g,(_,inner)=>{
      const cards=[...inner.matchAll(/<Card\s+([^>]*?)>([\s\S]*?)<\/Card>/g)];
      log.push(`OK: <CardGroup>(${cards.length}) -> ::: grids`);
      return `::: grids\n`+cards.map(([,a,c])=>`::: grid\n${buildCard(a,c)}\n:::`).join('\n')+`\n:::`;
    });
    body = body.replace(/<Card\s+([^>]*?)>([\s\S]*?)<\/Card>/g,(_,a,c)=>{ log.push(`OK: <Card> -> ::: card`); return buildCard(a,c); });
    // top-level callouts + leftover
    body = convertCallouts(body, log);
    for (const t of new Set([...body.matchAll(/<\/?[A-Z][A-Za-z]+/g)].map(m=>m[0]))) log.push(`TODO: unconverted tag ${t}`);
    report.push({file:relPath, log});
    return front+'\n'+body.replace(/\n{3,}/g,'\n\n').trimStart()+'\n';
  }

  function walk(dir){
    for (const name of readdirSync(dir)){
      const p=join(dir,name); const st=statSync(p);
      if (st.isDirectory()){ if(['docs','node_modules','_site'].includes(name)||name.startsWith('.')) continue; walk(p); continue; }
      if (!name.endsWith('.mdx')) continue;
      const rel=relative(ROOT,p);
      // introduction is the home -> docs/index.md (route "/")
      const outRel = rel==='introduction.mdx' ? 'index.md' : rel.replace(/\.mdx$/,'.md');
      const out=join(OUT,outRel); mkdirSync(dirname(out),{recursive:true});
      writeFileSync(out, convert(readFileSync(p,'utf8'), rel.split(sep).join('/')),'utf8');
      rmSync(p);
    }
  }

  mkdirSync(OUT,{recursive:true}); walk(ROOT);
  let md=`# Migration Mintlify -> docmd\n\nConverted ${report.length} files.\n\n`;
  md+= unmappedIcons.size ? `## ⚠️ Unmapped icons (verify on Lucide)\n`+[...unmappedIcons].map(i=>`- \`${i}\``).join('\n')+'\n\n' : `## Icons: all mapped ✅\n\n`;
  const todos=report.filter(r=>r.log.some(l=>l.startsWith('TODO')));
  md+=`## Unconverted tags\n\n`+(todos.length?todos.map(r=>`### ${r.file}\n`+r.log.filter(l=>l.startsWith('TODO')).map(l=>`- ${l}`).join('\n')).join('\n\n')+'\n\n':'None ✅\n\n');
  md+=`## Per-file log\n\n`+report.map(r=>`### \`${r.file}\`\n`+(r.log.length?r.log.map(l=>`- ${l}`).join('\n'):'- (plain markdown)')).join('\n\n')+'\n';
  writeFileSync(join(ROOT,'MIGRATION_REPORT.md'),md,'utf8');
  console.log(`Converted ${report.length}. Unmapped icons: ${unmappedIcons.size}. Files with TODO: ${todos.length}.`);
  ```

  **Run**: `node scripts/migrate-from-mintlify.mjs`. Then read `MIGRATION_REPORT.md`
  and manually fix any `TODO` entries (unconverted components).

  ## 6. Conversion rules (details + difficult cases)

  - **Dedent first, re-indent after**: capture the container body WITHOUT consuming
    the indentation of the first line (no `\s*` adjacent to the capture), then `dedent`,
    then for Steps re-indent to **3 spaces** (so nested code-fences and `::: callout`
    stay inside the numbered list item).
  - **`<AccordionGroup>`** → only wrapper removed; each `<Accordion>` becomes an
    independent `::: collapsible` (docmd has no exclusive accordion).
  - **Card with `href`** → never `::: button` (see gotchas): use `[Open →](href)`.
  - **`introduction.mdx` → `docs/index.md`** (route `/`, mandatory root page).
  - **Non-standard components** (`<ParamField>`, `<ResponseField>`, `<CodeGroup>`,
    `<Frame>`, custom JSX, OpenAPI playground): there is NO faithful automatic mapping.
    Rewrite them in **explicit Markdown** (e.g. a `<ParamField name type required>` →
    `### \`name\`` + list `**Type**`, `**Required**`, `**Description**`). More portable
    and AI-friendly. Flag in the report.
  - **`docs.json` → `docmd.config.json`**: map `navigation.groups[].pages[]` →
    `navigation[]` (see §7), `navbar.links`/`footer.socials` → `footer.columns`/links,
    `colors.primary` → `assets/custom.css`.

  ## 7. docmd.config.json (with navigation mapped from docs.json)

  ```json
  {
    "title": "<PROJECT_TITLE>",
    "description": "<ONE_LINE_DESC>",
    "url": "<SITE_URL>",
    "src": "docs", "out": "_site", "base": "/",
    "favicon": "assets/favicon.svg",
    "theme": { "name": "default", "defaultMode": "dark", "customCss": ["assets/custom.css"] },
    "autoTitleFromH1": true, "copyCode": true, "pageNavigation": true, "minify": true,
    "markdown": { "breaks": true },
    "layout": { "spa": true, "header": { "enabled": true },
      "sidebar": { "collapsible": true, "defaultCollapsed": false },
      "optionsMenu": { "position": "header", "components": { "search": true, "themeSwitch": true } } },
    "footer": { "style": "minimal", "branding": true,
      "content": "© <AUTHOR_NAME> — [<ORG>](<ORG_URL>) · [GitHub](<REPO_URL>) · <LICENSE>",
      "columns": [ { "title": "Project", "links": [ { "title": "GitHub", "path": "<REPO_URL>", "external": true } ] } ] },
    "navigation": [
      { "title": "<GROUP>", "icon": "<lucide>", "children": [
        { "title": "Introduction", "path": "/" },
        { "title": "<Page title>", "path": "/<slug>" }
      ]}
    ],
    "plugins": {
      "search": { "enabled": true, "semantic": true, "placeholder": "Search the docs…" },
      "git": { "repo": "<REPO_URL>", "branch": "main", "editLink": true, "lastUpdated": true, "commitHistory": true, "dateFormat": "relative" },
      "seo": { "defaultDescription": "<ONE_LINE_DESC>", "aiBots": true, "openGraph": {}, "twitter": { "cardType": "summary_large_image" } },
      "sitemap": { "defaultChangefreq": "weekly", "defaultPriority": 0.8 },
      "mermaid": {}, "math": {}, "llms": { "fullContext": true }, "analytics": { "enabled": false }
    }
  }
  ```

  `navigation[]` is the sole source of the sidebar; every new page must be added there
  or it will not appear. Icons = Lucide kebab-case.

  ## 8. Semantic search (docmd-search) — CRITICAL for CI

  `plugins.search.semantic: true`. Embeddings computed at **build-time** with ONNX;
  the browser receives only Int8 vectors (keyword + cosine, client-side, no server/model
  in the browser). **Avoid the interactive wizard** (blocks CI) by committing
  `<DOCS_DIR>/.docmd-search/config.json`:

  ```json
  { "model": "Xenova/all-MiniLM-L6-v2", "chunkSize": 512, "chunkOverlap": 64, "incremental": true, "topK": 10 }
  ```

  (EN ~23MB recommended; multilingual: `Xenova/multilingual-e5-small`.) `.gitignore`:

  ```
  node_modules/
  _site/
  .docmd-search/*
  !.docmd-search/config.json
  ```

  Test: `(sleep 34; echo "paraphrased query") | node node_modules/docmd-search/dist/bin/docmd-search.js docs` → relevant results.

  ## 9. Banner on homepage (if <BANNER_URL>)

  At the top of `docs/index.md` below the frontmatter: `![<PROJECT_TITLE> banner](<BANNER_URL>)`.

  ## 10. Container syntax (reference) and branding

  Containers: `::: callout {info|tip|warning|danger|success}`, `::: tabs`/`== tab "X"`,
  `::: steps` (numbered list, titles in bold), `::: collapsible "X"` (`open` to expand),
  `::: grids`/`::: grid`/`::: card "X" icon:lucide`, mermaid fence, KaTeX `$…$`/`$$…$$`.
  `assets/custom.css`: set `--docmd-color-primary`, `--color-primary`, `--link-color`
  to `<BRAND_HEX>` (and dark variant).

  ## 11. Documentation structure standards

  Sidebar groups: **Get Started, Guides, Concepts & Theory, Architecture, Best Practices,
  Operations, Reference**. Each deep page: **Motivation → Theory (KaTeX formulas,
  academic tone) → Design + Mermaid diagram (flowchart/sequenceDiagram) → Data
  model/contract → ADR (Problem→Decision→Trade-offs, in `::: collapsible`) → Worked-through
  example → Gotchas (`::: callout warning`)**. Create dedicated pages for architecture,
  **pipeline/workflow** (with pipeline diagram), **ADR**, reference (CLI/API).

  ## 12. CI guard — `scripts/check-no-mdx-tags.mjs`

  Fails if any MDX component tag survives in `docs/**.md` (verifies conversion completeness):

  ```js
  import { readdirSync, statSync, readFileSync } from 'node:fs'; import { join } from 'node:path';
  const DOCS=join(process.cwd(),'docs'); const TAG=/<\/?[A-Z][A-Za-z0-9]*(\s|>|\/)/g; const bad=[];
  (function w(d){for(const n of readdirSync(d)){const p=join(d,n); if(statSync(p).isDirectory()){w(p);continue;}
   if(!n.endsWith('.md'))continue; readFileSync(p,'utf8').split('\n').forEach((l,i)=>{const m=l.match(TAG); if(m) bad.push(`${p}:${i+1} ${m.join(' ')}`);});}})(DOCS);
  if(bad.length){console.error('Unconverted MDX tags:\n'+bad.join('\n'));process.exit(1);} console.log('OK: no residual MDX tags.');
  ```

  ## 13. Build & verification (mandatory)

  ```bash
  npm install && npm run check && npm run build
  ```
  - `_site/index.html` exists (otherwise `docs/index.md` is missing).
  - `npm run check` green (0 MDX tags remaining).
  - **0 occurrences of `:::` as visible text** (only inside `class="docmd-container …"`).
    Check: strip `<script>/<pre>/<code>` and tags from the HTML then search for `:::` → must return 0.
  - KaTeX OK (formulas rendered, 0 false positives on `$variables` in code blocks).
  - `_site/.docmd-search/manifest.json`, `llms.txt`, `sitemap.xml` present.

  ## 14. Deploy: Cloudflare Pages (Option A — primary, no API key)

  CF Git integration: build on every push to `main`. Pages build config: Root
  `<DOCS_DIR>`, Build command `npm run build`, Output `_site`, Node via `.node-version` (20).
  **Confirmed working**: the ONNX search build runs on the CF Linux image.
  Use `package-lock.json` **v3** (cross-platform: already includes
  `@img/sharp-linux-x64` and onnxruntime for `linux`). Attach `<SITE_URL>` in Custom domains.

  **Option B (fallback, not needed)**: a GitHub Actions workflow `workflow_dispatch`-only
  that builds and runs `wrangler pages deploy _site --project-name=<CF_PROJECT>` (secrets
  `CLOUDFLARE_API_TOKEN`+`CLOUDFLARE_ACCOUNT_ID`). Use only if the CF build fails on ONNX.

  ## 15. Gotchas (learned in the field)

  1. **Mandatory root page** → `docs/index.md` (route `/`); otherwise "Root index.html not found".
  2. **`::: button` is NOT a paired block**: the closing `:::` renders as text → in cards use `[Open →](href)`.
  3. **Steps**: dedent to column 0, re-indent to **3 spaces** (nested code-fences and callouts stay in the item).
  4. **Math**: KaTeX only outside code blocks; check for spurious `class="katex"` on pages with `$variables`.
  5. **Re-running the converter DELETES the `.mdx` files**: before re-running, restore originals with `git checkout -- '<DOCS_DIR>/*.mdx'`.
  6. **`AccordionGroup`** → sequential collapsibles (no exclusive accordion): only the "one open at a time" behaviour is lost, content is intact.
  7. **Lockfile** v3 cross-platform; do not commit a lock that resolves only the optional-deps for your OS.

  ## 16. DELIVERABLE — create skill + rule

  ### A) Skill `docmd-docs` → `.claude/skills/docmd-docs/SKILL.md`

  Frontmatter (`name`, `description` that triggers when working in `<DOCS_DIR>/`, adding
  pages, touching navigation/plugins, re-running the migration, or keeping docs in sync
  with features). Body: layout, commands, container syntax table,
  **Mintlify→docmd mapping + Lucide icons**, plugin config, **semantic search** (deps,
  pinned model, skip wizard, client-side), **footer/branding**, **CF deploy Option A
  confirmed + B fallback**, doc structure standards (theory→mermaid→ADR→example→gotchas),
  and the **gotchas** from §15 (including restoring `.mdx` files before re-converting).

  ### B) Strict auto-sync rule → `.claude/rules/rule-docmd-docs-sync.md`

  **Binding/blocking** rule: **every time** a user-facing feature is added/modified or
  the README is substantially updated, you MUST update in the same work the docmd page in
  `<DOCS_DIR>/docs/**` (and register it in `navigation[]` if new), following the
  `docmd-docs` skill. Declare when it is NOT needed (internal refactors, tooling fixes,
  cosmetics) by noting it in the PR/changelog. Before closing: `npm run check && npm run build`
  green. Anti-patterns: feature without docs; page not in `navigation[]`; reintroducing
  MDX/JSX syntax in `.md` files.

  ## 17. Acceptance criteria

  - [ ] `npm run check` green (0 MDX tags) and `npm run build` green, `_site/index.html` present.
  - [ ] 0 `:::` as visible text; all icons mapped (clean report) or TODOs resolved manually.
  - [ ] All plugins active: semantic search, git, seo, sitemap, mermaid, math, llms.
  - [ ] Semantic index generated + paraphrased test query returns relevant results.
  - [ ] Home banner (if present), footer with author credit, brand color, nav 1:1 from docs.json.
  - [ ] `MIGRATION_REPORT.md` generated; old `docs.json` and empty folders removed.
  - [ ] CF Pages Git integration configured → live deploy at `<SITE_URL>`.
  - [ ] Skill `docmd-docs` and strict auto-sync rule created.
