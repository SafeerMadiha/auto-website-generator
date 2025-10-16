# Prompt → Automatische website → Meteen live

Dit is een **starter‑repo** waarmee je met één prompt een statische website genereert, automatisch commit naar GitHub en **direct live** zet op **Vercel** of **Netlify**.

---

## 🔧 Wat zit erin
```
.
├─ .github/workflows/generate-site.yml   # GitHub Action: prompt → generate → commit → deploy
├─ generator/
│  └─ generate-site.js                   # Simpele generator (HTML uit JSON‑prompt)
├─ template/
│  └─ base.html                          # (Optioneel) losse basis‑template
├─ vercel.json                           # Vercel: output map = public/
├─ netlify.toml                          # Netlify: publish map = public/
├─ package.json                          # Scripts (Node 20)
├─ .gitignore                            # Negeer buildmap (dist/) maar commit public/
└─ README.md                             # Uitleg en prompt‑schema
```

---

## 🚀 Snel aan de slag
1) **Maak een lege GitHub‑repo** en zet onderstaande bestanden 1‑op‑1 in die repo (zelfde paden/bestandsnamen).
2) **Koppel de repo aan Vercel of Netlify** (Project → Link Git Repository).
   - **Vercel**: Project Settings → *Build & Output Settings* → `Output Directory = public`. (Of laat het automatisch; `vercel.json` regelt dit al.)
   - **Netlify**: `Publish directory = public` (staat in `netlify.toml`).
3) **Start de workflow** in GitHub → *Actions* → *Generate & Deploy Site* → *Run workflow* → plak je **prompt (JSON)**.
4) Binnen enkele seconden worden bestanden in `public/` gecommit → hosting pakt de change op → **site live**.

> Tip: Je kunt ook een **GitHub Issue** openen met label `deploy` en als issue‑tekst je JSON‑prompt; de workflow pakt dat automatisch op.

---

## 🧩 Prompt‑schema (JSON)
Plak dit als input bij *Run workflow* **of** als issue‑tekst (met label `deploy`). Pas aan naar jouw wensen.

```json
{
  "site_name": "3Core Solutions",
  "brand_color": "#0ea5e9",
  "hero": {
    "headline": "IT die wél werkt",
    "subheadline": "Cloud, security en integraties voor MKB – zonder gedoe."
  },
  "cta": { "text": "Plan een call", "href": "mailto:info@3coresolutions.nl" },
  "sections": [
    { "id": "diensten", "title": "Diensten", "content": "Cloud migraties, integraties, beheer en support." },
    { "id": "cases", "title": "Cases", "content": "Resultaten bij klanten in retail en zorg." },
    { "id": "contact", "title": "Contact", "content": "Binnen 24 uur reactie." }
  ],
  "pages": [
    { "slug": "privacy", "title": "Privacy", "html": "<h1>Privacy</h1><p>Je data is veilig.</p>" }
  ]
}
```

---

## 📄 Bestanden (kopieer exact zo)

### `.github/workflows/generate-site.yml`
```yaml
name: Generate & Deploy Site

on:
  workflow_dispatch:
    inputs:
      prompt:
        description: "Beschrijf de site (JSON, zie README)"
        required: true
        type: string
  issues:
    types: [opened, edited, labeled]

permissions:
  contents: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repo
        uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install deps (if any)
        run: npm ci || true

      - name: Extract prompt (workflow_dispatch or issue with label 'deploy')
        run: |
          if [ "${{ github.event_name }}" = "workflow_dispatch" ]; then
            echo '${{ github.event.inputs.prompt }}' > prompt.json
          else
            echo "Event: issues"
            echo '${{ toJson(github.event.issue.labels) }}' | grep -q '"name":"deploy"' || { echo "No deploy label — skipping"; exit 0; }
            printf "%s" '${{ github.event.issue.body }}' > prompt.json
          fi
          echo "Prompt saved to prompt.json"

      - name: Generate site from prompt
        run: |
          node generator/generate-site.js prompt.json dist

      - name: Commit & push changes to public/
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          mkdir -p public
          rm -rf public/*
          cp -r dist/* public/
          git add public
          git commit -m "chore: generate site from prompt [skip ci]" || echo "No changes"
          git push
```

### `generator/generate-site.js`
```js
// generator/generate-site.js
// Gebruik: node generator/generate-site.js prompt.json dist
const fs = require('fs');
const path = require('path');

function ensureDir(p) { if (!fs.existsSync(p)) fs.mkdirSync(p, { recursive: true }); }

function renderHTML(data) {
  const title = data.site_name || 'Mijn Website';
  const color = data.brand_color || '#0ea5e9';
  const sections = Array.isArray(data.sections) ? data.sections : [];
  const cta = data.cta || { text: 'Neem contact op', href: '#' };

  const nav = sections.map(s => `<a href="#${s.id}">${s.title}</a>`).join(' ');
  const body = sections.map(s => `
    <section id="${s.id}" style="padding:48px 0;border-bottom:1px solid #eee">
      <h2 style="margin:0 0 12px 0">${s.title}</h2>
      <p>${s.content || ''}</p>
    </section>
  `).join('\n');

  return `<!doctype html>
<html lang="nl">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>${title}</title>
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
  <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;600;800&display=swap" rel="stylesheet">
  <style>
    html,body{margin:0;padding:0;font-family:Inter,system-ui,-apple-system,Segoe UI,Roboto,Ubuntu,Cantarell,Noto Sans,'Helvetica Neue',Arial}
    .container{max-width:1100px;margin:0 auto;padding:0 20px}
    .btn{display:inline-block;padding:12px 18px;border-radius:12px;text-decoration:none}
    header{background:${color};color:white;padding:18px 0}
    header a{color:white;text-decoration:none;margin-right:16px;font-weight:600}
    .hero{padding:64px 0}
    .footer{padding:48px 0;color:#777}
  </style>
</head>
<body>
  <header>
    <div class="container">
      <strong>${title}</strong>
      <nav style="float:right">${nav}</nav>
      <div style="clear:both"></div>
    </div>
  </header>

  <main class="container">
    <section class="hero">
      <h1 style="font-size:42px;margin:0 0 12px 0">${data.hero?.headline || title}</h1>
      <p style="max-width:700px">${data.hero?.subheadline || 'Snel online met een moderne website.'}</p>
      <p style="margin-top:18px">
        <a class="btn" href="${cta.href}" style="background:black;color:white">${cta.text}</a>
      </p>
    </section>

    ${body}
  </main>

  <div class="footer container">
    © ${new Date().getFullYear()} ${title}
  </div>
</body>
</html>`;
}

(function main() {
  const [, , promptPath, outDir] = process.argv;
  if (!promptPath || !outDir) {
    console.error('Usage: node generator/generate-site.js prompt.json dist');
    process.exit(1);
  }
  const data = JSON.parse(fs.readFileSync(promptPath, 'utf8'));
  const dist = path.resolve(outDir);
  ensureDir(dist);
  fs.writeFileSync(path.join(dist, 'index.html'), renderHTML(data), 'utf8');

  // Extra pagina's indien opgegeven
  if (Array.isArray(data.pages)) {
    data.pages.forEach(p => {
      const html = `<!doctype html><meta charset="utf-8"><title>${p.title}</title>${p.html || '<h1>Lege pagina</h1>'}`;
      fs.writeFileSync(path.join(dist, `${p.slug || 'pagina'}.html`), html, 'utf8');
    });
  }
  console.log('Site generated in', dist);
})();
```

### `template/base.html` (optioneel, je kunt de styles/layout hieruit halen)
```html
<!doctype html>
<html lang="nl">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>{{ title }}</title>
</head>
<body>
  <!-- Jouw basis‑HTML template -->
</body>
</html>
```

### `vercel.json`
```json
{
  "buildCommand": "",
  "outputDirectory": "public"
}
```

### `netlify.toml`
```toml
[build]
  publish = "public"
  command = ""
```

### `package.json`
```json
{
  "name": "prompt-to-site",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "generate": "node generator/generate-site.js prompt.json dist"
  }
}
```

### `.gitignore`
```gitignore
node_modules/
dist/
# Laat public/ NIET in .gitignore staan; de workflow commit public/
```

### `README.md` (mag gelijk aan deze uitleg zijn)
```md
# Prompt → Site → Live
Zie de workflow‑beschrijving in de hoofd‑README.
```

---

## 🧪 Test lokaal (optioneel)
```bash
npm run generate
# → leest prompt.json en schrijft HTML naar dist/
```

---

## 🔁 Variaties
- **Issue‑only:** verwijder `workflow_dispatch` en werk alleen met issues + label `deploy`.
- **Next.js/Tailwind:** vervang de generator door een Next.js starter en map JSON → pagina’s.
- **Meertaligheid:** meerdere prompts → matrix build die meerdere mappen in `public/` vult.
- **AI‑content:** laat de generator ontbrekende secties/teksten aanvullen (API‑call toevoegen).

---

## ✅ Verwachte eindresultaten
- Elke run overschrijft de inhoud van `public/` met jouw nieuwe site.
- Vercel/Netlify ziet de commit → **auto‑deploy** → iedereen kan de link bezoeken.
- Domein koppelen? Zet in Vercel/Netlify jouw eigen domein naar dit project.

