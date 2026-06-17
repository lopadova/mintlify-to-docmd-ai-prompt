# Task: migra la documentazione da Mintlify (MDX) a docmd (+ deploy Cloudflare Pages)

  Sei un agente che deve **convertire** una documentazione esistente in **Mintlify**
  (file `.mdx` + `docs.json`) verso **docmd** (https://docs.docmd.io), generatore di
  siti statici basato su Markdown. La conversione NON è un semplice rename `.mdx`→`.md`:
  i componenti MDX vanno tradotti nei container docmd e le icone rimappate. Verifica
  SEMPRE con un build reale; non dare nulla per scontato.

  ## 0. Valori da riempire

  - `<PACKAGE_NAME>` / `<PROJECT_TITLE>` / `<ONE_LINE_DESC>`
  - `<REPO_URL>` · `<SITE_URL>` · `<BRAND_HEX>` · `<BANNER_URL>` (se esiste)
  - `<DOCS_DIR>` = cartella che oggi contiene la doc Mintlify (es. `docs-site`)
  - `<CF_PROJECT>` = nome progetto Cloudflare Pages
  - `<AUTHOR_NAME>` / `<ORG>` / `<ORG_URL>` / `<LICENSE>`

  ## 1. Analisi pre-migrazione (NON saltarla)

  Prima di convertire, fai l'inventario per dimensionare il lavoro e scoprire edge case:

  ```bash
  cd <DOCS_DIR>
  # elenco file e config
  find . -name "*.mdx" | sort ; cat docs.json
  # frequenza componenti MDX
  grep -rhoE "<[A-Z][A-Za-z]+" --include=*.mdx . | sort | uniq -c | sort -rn
  # import/JSX custom (rischio alto se presenti)
  grep -rnE "^import |^export " --include=*.mdx .
  # componenti "difficili" da verificare a mano
  grep -rhoE "<(ParamField|ResponseField|Frame|Icon|CodeGroup|Tabs|Accordion|Steps|CardGroup|Columns|Expandable|Snippet)" --include=*.mdx . | sort | uniq -c
  # icone usate (FontAwesome → vanno rimappate a Lucide)
  grep -rhoE 'icon="[a-z-]+"' --include=*.mdx . | sort | uniq -c
  ```

  Classifica: se è quasi-Markdown + componenti standard → conversione ~lossless. Se ci
  sono `import`/JSX custom o `ParamField`/`ResponseField`/playground OpenAPI →
  quei pezzi vanno riscritti a mano in Markdown esplicito (vedi §6, regola finale).

  ## 2. Struttura target (in-place dentro <DOCS_DIR>)

  ```
  <DOCS_DIR>/
    docmd.config.json
    package.json
    .node-version                # "20"
    .gitignore
    .docmd-search/config.json    # modello embedding pinnato (COMMITTATO)
    assets/{favicon.svg,custom.css}
    scripts/{migrate-from-mintlify.mjs, check-no-mdx-tags.mjs}
    docs/**/*.md                 # output: i .mdx convertiti, route = albero file
    MIGRATION_REPORT.md          # log per-file della conversione
    _site/                       # build output (git-ignored)
  ```

  I contenuti vanno in `docs/` (route invariate: `docs/guides/x.md` → `/guides/x`).
  Rimuovi a fine lavoro: il vecchio `docs.json` Mintlify e le cartelle svuotate.

  ## 3. Install & versioni

  ```bash
  npm install -D @docmd/core            # VERIFICA la versione reale con `npm view @docmd/core version`
  npm install docmd-search
  npm install -D @huggingface/transformers onnxruntime-node
  ```

  `package.json` scripts: `dev: docmd dev`, `build: docmd build`,
  `check: node scripts/check-no-mdx-tags.mjs`. `.node-version` = `20`. `"type":"module"`.

  ## 4. Mapping Mintlify → docmd

  | Componente Mintlify | docmd | Note |
  |---|---|---|
  | `<Note>` | `::: callout info` … `:::` | |
  | `<Tip>` | `::: callout tip` | |
  | `<Warning>` | `::: callout warning` | |
  | `<Info>` | `::: callout info` | |
  | `<Check>` | `::: callout success` | |
  | `<Danger>` | `::: callout danger` | |
  | `<Tabs>`/`<Tab title="X">` | `::: tabs` + `== tab "X"` + `:::` | |
  | `<Steps>`/`<Step title="X">` | `::: steps` + lista `N. **X**` (corpo indentato **3 spazi**) + `:::` | |
  | `<AccordionGroup>`/`<Accordion title="X">` | `::: collapsible "X"` … `:::` **sequenziali** | docmd NON ha accordion esclusivo |
  | `<CardGroup>`/`<Card title icon href>` | `::: grids`›`::: grid`›`::: card "title" icon:lucide` + `[Apri →](href)` ›`:::`›`:::`›`:::` | NIENTE `::: button` (vedi
  gotcha) |
  | `<code>x</code>` | `` `x` `` | |
  | `&nbsp;` | spazio normale | |
  | ` ```mermaid ` | invariato | plugin mermaid |
  | frontmatter `icon:` | rimosso dalla pagina | l'icona vive in `navigation[]` |

  **Icone**: Mintlify usa **FontAwesome**, docmd usa **Lucide** → vanno **rimappate**
  (non copiate). Esempi: `function`→`square-function`, `wave-pulse`→`activity`,
  `layer-group`→`layers`, `shield-halved`→`shield`, `scale-balanced`→`scale`,
  `diagram-project`→`workflow`, `sitemap`→`network`, `gear`→`settings`, `bolt`→`zap`,
  `robot`→`bot`, `seedling`→`sprout`, `code-compare`→`git-compare`,
  `chart-line`→`trending-up`, `book-bookmark`→`book-marked`, `traffic-light`→`traffic-cone`.
  Estendi la tabella per ogni icona trovata in §1; le non mappate passano invariate ma
  vanno segnalate nel report.

  ## 5. Script di conversione (deterministico) — `scripts/migrate-from-mintlify.mjs`

  Usa uno script strutturale (non regex naive) che gestisce l'annidamento. Eseguilo dalla
  root `<DOCS_DIR>`: legge i `.mdx`, scrive in `docs/**.md`, cancella gli originali e
  produce `MIGRATION_REPORT.md`.

  ```js
  import { readFileSync, writeFileSync, readdirSync, statSync, mkdirSync, rmSync } from 'node:fs';
  import { join, dirname, relative, sep } from 'node:path';

  const ROOT = process.cwd();
  const OUT = join(ROOT, 'docs');

  // Estendi con TUTTE le icone trovate in §1 (FontAwesome -> Lucide, kebab-case).
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
    // frontmatter: tieni title/description, togli `icon`
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
    // Steps (3-space indent + callout annidati)
    body = body.replace(/<Steps>([\s\S]*?)<\/Steps>/g,(_,inner)=>{
      const steps=[...inner.matchAll(/<Step\s+title="([^"]*)"\s*>([\s\S]*?)<\/Step>/g)];
      log.push(`OK: <Steps>(${steps.length}) -> ::: steps`);
      return `::: steps\n\n`+steps.map(([,t,c],i)=>{ const b=convertCallouts(dedent(c),log); return `${i+1}. **${t}**`+(b?'\n'+indent(b,3):'');
  }).join('\n\n')+`\n\n:::`;
    });
    // Accordion -> collapsible sequenziali
    body = body.replace(/<AccordionGroup>([\s\S]*?)<\/AccordionGroup>/g,(_,i)=>i);
    body = body.replace(/<Accordion\s+title="([^"]*)"\s*>([\s\S]*?)<\/Accordion>/g,(_,t,c)=>{
      log.push(`OK: <Accordion "${t}"> -> ::: collapsible`); return `::: collapsible "${t}"\n${convertCallouts(dedent(c),log)}\n:::`;
    });
    // Cards -> grids/grid/card (+ link markdown, NON button)
    const buildCard=(attrs,content)=>{
      const g=(re)=>(attrs.match(re)||[, ''])[1];
      const title=g(/title="([^"]*)"/), iconFa=g(/icon="([^"]*)"/), href=g(/href="([^"]*)"/);
      const icon=iconFa?` icon:${mapIcon(iconFa)}`:''; const desc=convertCallouts(dedent(content),log);
      const btn=href?`\n\n[Apri →](${href})`:'';
      return `::: card "${title}"${icon}\n${desc}${btn}\n:::`;
    };
    body = body.replace(/<CardGroup[^>]*>([\s\S]*?)<\/CardGroup>/g,(_,inner)=>{
      const cards=[...inner.matchAll(/<Card\s+([^>]*?)>([\s\S]*?)<\/Card>/g)];
      log.push(`OK: <CardGroup>(${cards.length}) -> ::: grids`);
      return `::: grids\n`+cards.map(([,a,c])=>`::: grid\n${buildCard(a,c)}\n:::`).join('\n')+`\n:::`;
    });
    body = body.replace(/<Card\s+([^>]*?)>([\s\S]*?)<\/Card>/g,(_,a,c)=>{ log.push(`OK: <Card> -> ::: card`); return buildCard(a,c); });
    // callout top-level + leftover
    body = convertCallouts(body, log);
    for (const t of new Set([...body.matchAll(/<\/?[A-Z][A-Za-z]+/g)].map(m=>m[0]))) log.push(`TODO: tag non convertito ${t}`);
    report.push({file:relPath, log});
    return front+'\n'+body.replace(/\n{3,}/g,'\n\n').trimStart()+'\n';
  }

  function walk(dir){
    for (const name of readdirSync(dir)){
      const p=join(dir,name); const st=statSync(p);
      if (st.isDirectory()){ if(['docs','node_modules','_site'].includes(name)||name.startsWith('.')) continue; walk(p); continue; }
      if (!name.endsWith('.mdx')) continue;
      const rel=relative(ROOT,p);
      // introduction è la home -> docs/index.md (route "/")
      const outRel = rel==='introduction.mdx' ? 'index.md' : rel.replace(/\.mdx$/,'.md');
      const out=join(OUT,outRel); mkdirSync(dirname(out),{recursive:true});
      writeFileSync(out, convert(readFileSync(p,'utf8'), rel.split(sep).join('/')),'utf8');
      rmSync(p);
    }
  }

  mkdirSync(OUT,{recursive:true}); walk(ROOT);
  let md=`# Migrazione Mintlify -> docmd\n\nConvertiti ${report.length} file.\n\n`;
  md+= unmappedIcons.size ? `## ⚠️ Icone non mappate (verifica su Lucide)\n`+[...unmappedIcons].map(i=>`- \`${i}\``).join('\n')+'\n\n' : `## Icone: tutte mappate
  ✅\n\n`;
  const todos=report.filter(r=>r.log.some(l=>l.startsWith('TODO')));
  md+=`## Tag non convertiti\n\n`+(todos.length?todos.map(r=>`### ${r.file}\n`+r.log.filter(l=>l.startsWith('TODO')).map(l=>`-
  ${l}`).join('\n')).join('\n\n')+'\n\n':'Nessuno ✅\n\n');
  md+=`## Log per file\n\n`+report.map(r=>`### \`${r.file}\`\n`+(r.log.length?r.log.map(l=>`- ${l}`).join('\n'):'- (markdown puro)')).join('\n\n')+'\n';
  writeFileSync(join(ROOT,'MIGRATION_REPORT.md'),md,'utf8');
  console.log(`Convertiti ${report.length}. Icone non mappate: ${unmappedIcons.size}. File con TODO: ${todos.length}.`);
  ```

  **Esegui**: `node scripts/migrate-from-mintlify.mjs`. Poi leggi `MIGRATION_REPORT.md`
  e sistema a mano i `TODO` (componenti non mappati).

  ## 6. Regole di conversione (dettaglio + casi difficili)

  - **Dedent prima, re-indent dopo**: cattura il corpo dei container SENZA mangiare
    l'indentazione della prima riga (niente `\s*` adiacente alla capture), poi `dedent`,
    poi per gli Steps re-indenta a **3 spazi** (così code-fence e `::: callout` annidati
    restano dentro l'item della lista numerata).
  - **`<AccordionGroup>`** → solo wrapper rimosso; ogni `<Accordion>` diventa un
    `::: collapsible` indipendente (docmd non ha accordion esclusivo).
  - **Card con `href`** → mai `::: button` (vedi gotcha): usa `[Apri →](href)`.
  - **`introduction.mdx` → `docs/index.md`** (route `/`, pagina root obbligatoria).
  - **Componenti non standard** (`<ParamField>`, `<ResponseField>`, `<CodeGroup>`,
    `<Frame>`, JSX custom, OpenAPI playground): NON esiste mappatura automatica fedele.
    Riscrivili in **Markdown esplicito** (es. un `<ParamField name type required>` →
    `### \`name\`` + lista `**Type**`, `**Required**`, `**Description**`). Più portabile e
    AI-friendly. Segnali nel report.
  - **`docs.json` → `docmd.config.json`**: mappa `navigation.groups[].pages[]` →
    `navigation[]` (vedi §7), `navbar.links`/`footer.socials` → `footer.columns`/links,
    `colors.primary` → `assets/custom.css`.

  ## 7. docmd.config.json (con navigation mappata da docs.json)

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
      { "title": "<GRUPPO>", "icon": "<lucide>", "children": [
        { "title": "Introduction", "path": "/" },
        { "title": "<Titolo pagina>", "path": "/<slug>" }
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

  `navigation[]` è l'unica fonte della sidebar; ogni pagina nuova va aggiunta o non compare.
  Icone = Lucide kebab-case.

  ## 8. Ricerca semantica (docmd-search) — CRITICO per la CI

  `plugins.search.semantic: true`. Embedding calcolati a **build-time** con ONNX; browser
  riceve solo vettori Int8 (keyword + cosine, client-side, nessun server/modello nel browser).
  **Evita il wizard interattivo** (blocca la CI) committando
  `<DOCS_DIR>/.docmd-search/config.json`:

  ```json
  { "model": "Xenova/all-MiniLM-L6-v2", "chunkSize": 512, "chunkOverlap": 64, "incremental": true, "topK": 10 }
  ```

  (EN ~23MB consigliato; multilingue: `Xenova/multilingual-e5-small`.) `.gitignore`:

  ```
  node_modules/
  _site/
  .docmd-search/*
  !.docmd-search/config.json
  ```

  Test: `(sleep 34; echo "query parafrasata") | node node_modules/docmd-search/dist/bin/docmd-search.js docs` → risultati pertinenti.

  ## 9. Banner in homepage (se <BANNER_URL>)

  In cima a `docs/index.md` sotto il frontmatter: `![<PROJECT_TITLE> banner](<BANNER_URL>)`.

  ## 10. Sintassi container (reference) e brand

  Container: `::: callout {info|tip|warning|danger|success}`, `::: tabs`/`== tab "X"`,
  `::: steps` (lista numerata, titoli in grassetto), `::: collapsible "X"` (`open` per
  aprire), `::: grids`/`::: grid`/`::: card "X" icon:lucide`, mermaid fence, KaTeX `$…$`/`$$…$$`.
  `assets/custom.css`: imposta `--docmd-color-primary`, `--color-primary`, `--link-color`
  a `<BRAND_HEX>` (e variante dark).

  ## 11. Standard di struttura della documentazione

  Gruppi sidebar: **Get Started, Guides, Concetti & Teoria, Architettura, Best Practices,
  Operations, Reference**. Ogni pagina profonda: **Motivazione → Teoria (formule KaTeX,
  tono accademico) → Design + diagramma Mermaid (flowchart/sequenceDiagram) → Modello
  dati/contratto → ADR (Problema→Decisione→Trade-off, in `::: collapsible`) → Esempio
  worked-through → Gotcha (`::: callout warning`)**. Crea pagine dedicate per architettura,
  **pipeline/workflow** (con diagramma della pipeline), **ADR**, reference (CLI/API).

  ## 12. Guard CI — `scripts/check-no-mdx-tags.mjs`

  Fallisce se in `docs/**.md` sopravvive un tag componente MDX (verifica completezza conversione):

  ```js
  import { readdirSync, statSync, readFileSync } from 'node:fs'; import { join } from 'node:path';
  const DOCS=join(process.cwd(),'docs'); const TAG=/<\/?[A-Z][A-Za-z0-9]*(\s|>|\/)/g; const bad=[];
  (function w(d){for(const n of readdirSync(d)){const p=join(d,n); if(statSync(p).isDirectory()){w(p);continue;}
   if(!n.endsWith('.md'))continue; readFileSync(p,'utf8').split('\n').forEach((l,i)=>{const m=l.match(TAG); if(m) bad.push(`${p}:${i+1} ${m.join(' ')}`);});}})(DOCS);
  if(bad.length){console.error('Tag MDX non convertiti:\n'+bad.join('\n'));process.exit(1);} console.log('OK: nessun tag MDX residuo.');
  ```

  ## 13. Build & verifica (obbligatoria)

  ```bash
  npm install && npm run check && npm run build
  ```
  - `_site/index.html` esiste (altrimenti manca `docs/index.md`).
  - `npm run check` verde (0 tag MDX residui).
  - **0 occorrenze di `:::` come testo visibile** (solo dentro `class="docmd-container …"`).
    Controlla: rimuovi `<script>/<pre>/<code>` e i tag dall'HTML e cerca `:::` → deve dare 0.
  - KaTeX ok (formule renderizzate, 0 falsi positivi su `$variabili` nei code-block).
  - `_site/.docmd-search/manifest.json`, `llms.txt`, `sitemap.xml` presenti.

  ## 14. Deploy: Cloudflare Pages (Opzione A — primaria, niente API key)

  Git integration di CF: build ad ogni push su `main`. Build config Pages: Root
  `<DOCS_DIR>`, Build command `npm run build`, Output `_site`, Node via `.node-version` (20).
  **Confermato funzionante**: il build ONNX della search semantica gira sull'immagine
  Linux di CF. Usa `package-lock.json` **v3** (cross-platform: contiene già
  `@img/sharp-linux-x64` e onnxruntime per `linux`). Collega `<SITE_URL>` in Custom domains.

  **Opzione B (fallback, non necessaria)**: workflow GitHub Actions `workflow_dispatch`-only
  che builda e fa `wrangler pages deploy _site --project-name=<CF_PROJECT>` (secret
  `CLOUDFLARE_API_TOKEN`+`CLOUDFLARE_ACCOUNT_ID`). Usalo solo se il build CF rompe su ONNX.

  ## 15. Gotcha (imparati sul campo)

  1. **Pagina root obbligatoria** → `docs/index.md` (route `/`); altrimenti "Root index.html not found".
  2. **`::: button` NON è un blocco appaiato**: il `:::` di chiusura esce come testo → nelle card usa `[Apri →](href)`.
  3. **Steps**: de-indenta a colonna 0, re-indenta a **3 spazi** (code-fence e callout annidati restano nell'item).
  4. **Math**: KaTeX solo fuori dai code-block; controlla `class="katex"` spurie nelle pagine con `$variabili`.
  5. **Re-eseguire il convertitore CANCELLA i `.mdx`**: prima di ri-eseguirlo ripristina gli originali con `git checkout -- '<DOCS_DIR>/*.mdx'`.
  6. **`AccordionGroup`** → collapsible sequenziali (nessun accordion esclusivo): perdita solo del comportamento "uno aperto alla volta", contenuto integro.
  7. **Lockfile** v3 cross-platform; non committare un lock che risolve solo gli optional-deps del tuo OS.

  ## 16. DELIVERABLE — crea skill + rule


  ### A) Skill `docmd-docs` → `.claude/skills/docmd-docs/SKILL.md`

  Frontmatter (`name`, `description` che innesca quando si lavora in `<DOCS_DIR>/`, si
  aggiungono pagine, si tocca navigation/plugin, si ri-esegue la migrazione, o si tiene
  la doc in sync con le feature). Corpo: layout, comandi, tabella sintassi container,
  **mapping Mintlify→docmd + icone Lucide**, config plugin, **ricerca semantica** (deps,
  pin modello, skip wizard, client-side), **footer/branding**, **deploy CF Opzione A
  confermato + B fallback**, standard struttura doc (teoria→mermaid→ADR→esempio→gotcha),
  e i **gotcha** del §15 (incluso il restore dei `.mdx` prima di ri-convertire).

  ### B) Rule rigida di auto-sync → `.claude/rules/rule-docmd-docs-sync.md`

  Rule **vincolante/bloccante**: **ogni volta** che si aggiunge/modifica una feature
  user-facing o si aggiorna il README in modo sostanziale, si DEVE aggiornare nello stesso
  lavoro la pagina docmd in `<DOCS_DIR>/docs/**` (e registrarla in `navigation[]` se nuova),
  seguendo la skill `docmd-docs`. Dichiara quando NON serve (refactor interni, fix tooling,
  cosmetica) scrivendolo nel PR/changelog. Prima di chiudere: `npm run check && npm run build`
  verde. Anti-pattern: feature senza doc; pagina non in `navigation[]`; reintrodurre
  sintassi MDX/JSX nei `.md`.

  ## 17. Criteri di accettazione

  - [ ] `npm run check` verde (0 tag MDX) e `npm run build` verde, `_site/index.html` presente.
  - [ ] 0 `:::` come testo visibile; icone tutte mappate (report pulito) o TODO risolti a mano.
  - [ ] Tutti i plugin attivi: search semantica, git, seo, sitemap, mermaid, math, llms.
  - [ ] Indice semantico generato + test query parafrasata pertinente.
  - [ ] Banner home (se presente), footer con credito autore, brand color, nav 1:1 da docs.json.
  - [ ] `MIGRATION_REPORT.md` generato; vecchio `docs.json` e cartelle vuote rimossi.
  - [ ] CF Pages Git integration configurata → deploy live su `<SITE_URL>`.
  - [ ] Skill `docmd-docs` e rule rigida di auto-sync create.
