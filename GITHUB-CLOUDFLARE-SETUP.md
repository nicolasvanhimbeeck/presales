# Sharing Presales Mockups Securely: GitHub Pages + Cloudflare Access

## Goal

Host HTML mockups on GitHub Pages, organized per client in separate folders, with real
server-side access control per client (not a client-side password).

- Client A can only access `/credendo-p2p/`
- Client B can only access `/digipolis-btp/`
- No password ever ships inside the HTML/JS itself

---

## Part 1 — Repo structure

Repo: **https://github.com/nicolasvanhimbeeck/presales.git**

```
presales/
├── .github/
│   └── workflows/
│       └── pages.yml
├── public/
│   ├── credendo-p2p/
│   │   └── index.html
│   ├── digipolis-btp/
│   │   └── index.html
│   ├── CNAME                (contains just: mockups.flexso.be — only add this once you reach Part 3)
│   └── index.html           (optional landing page, or leave empty/404)
└── README.md
```

Each client's mockup(s) go in their own subfolder under `public/`. The final URLs will look like:

```
https://nicolasvanhimbeeck.github.io/presales/credendo-p2p/
https://nicolasvanhimbeeck.github.io/presales/digipolis-btp/
```

Once the custom domain is attached (Part 3), the client-facing URL becomes
`https://mockups.flexso.be/credendo-p2p/` instead.

### `.github/workflows/pages.yml`

GitHub Pages' plain "deploy from a branch" mode expects the site at the repo root (or
`/docs`), not `public/`. Using a small Actions workflow instead lets you keep the
`public/` folder as the source without moving files around:

```yaml
name: Deploy Pages

on:
  push:
    branches: [main]

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/configure-pages@v5
      - uses: actions/upload-pages-artifact@v3
        with:
          path: public
      - id: deployment
        uses: actions/deploy-pages@v4
```

### Adding a new client mockup later

1. Create a new folder under `public/`, e.g. `public/new-client-name/`
2. Drop the generated `index.html` (and any assets) in there
3. Commit + push to `main` → Actions workflow runs → live automatically

---

## Part 2 — Enable GitHub Pages

1. In the GitHub repo → **Settings → Pages**
2. Under **Build and deployment → Source**, choose **GitHub Actions** (not "Deploy from a branch")
3. Push to `main` once — the workflow from Part 1 runs and the Pages URL appears under Settings → Pages
4. **Repo visibility**: the repo can stay **public** — Cloudflare Access is the actual access-control layer, not repo privacy. Making the repo private is optional and, on GitHub Free personal plans, has restrictions around private-repo Pages visibility, so it's not worth fighting with for this use case.

At this point, everything is **publicly reachable by URL** — no access control yet. That's expected; access control gets added in front of it via Cloudflare next.

---

## Part 3 — Put Cloudflare Access in front of it

Cloudflare Access sits between the visitor and GitHub Pages, checking identity (via email one-time code) **before** any HTML is served. The password/OTP never touches the file itself.

### 3.1 Add the domain to Cloudflare

1. Add the domain you'll use for mockups (e.g. `mockups.flexso.be`) to your Cloudflare account, if not already added
2. DNS record: **CNAME** `mockups` → `<username>.github.io` — **Proxied (orange cloud)**, required so Cloudflare Access can intercept requests
3. In the repo, add a `CNAME` file inside `public/` containing exactly `mockups.flexso.be` (no protocol, no trailing slash) — GitHub Pages needs this to know which custom domain to serve
4. In GitHub → repo **Settings → Pages → Custom domain**, enter `mockups.flexso.be` and save (GitHub will check DNS; it may show "not secure" briefly — safe to ignore since Cloudflare terminates TLS for visitors anyway)
5. Optional but recommended: verify domain ownership under your **GitHub account → Settings → Pages → Verified domains**, so no one else on GitHub can claim the same custom domain

### 3.2 Set up Cloudflare Access

1. Cloudflare dashboard → **Zero Trust → Access → Applications**
2. **Add an application** → type: *Self-hosted*
3. Application domain: `mockups.flexso.be/credendo-p2p` (path-based — repeat per client)
4. Under **Policies**, create a rule:
   - Action: Allow
   - Include: **Emails ending in** `@credendo.com` (or specific email addresses if you prefer)
5. Repeat step 3–4 for each client path (e.g. a second Application for `/digipolis-btp`, restricted to the Digipolis/AG Vespa contacts)

### 3.3 What the client experience looks like

1. Client opens the link you sent them
2. Cloudflare intercepts the request, shows a "verify your email" screen
3. Client enters their email → gets a one-time code by mail → enters it
4. Only then does the mockup content load

No password lives in the HTML. No file gets flagged by mail/SharePoint scanners, because you're only ever sharing a link.

---

## Part 4 — Sending the link to the client

Just share the URL directly (chat, email body, whatever) — since there's no attachment, mail filters have nothing to flag. If the client isn't already on the Access allow-list, add their email under the relevant Application's policy before sending.

---

## Notes / things to double check

- **Domain ownership**: you need control over a domain/subdomain's DNS to point it at Cloudflare. Check with whoever manages `flexso.be` DNS if `mockups.flexso.be` (or similar) is available to delegate.
- **Free tier limits**: Cloudflare Access free tier supports up to 50 users — almost certainly enough for presales use. Zero Trust Free requires a card on file for verification (no charge expected at this scale).
- **Revoking access**: removing a client's email from the Application policy immediately cuts off access — useful once a deal closes or falls through.
- **Multiple mockups per client**: just add more subfolders under the same client path; the Cloudflare Application rule covers the whole path prefix.
- **Personal GitHub account**: since this lives on a personal account rather than a Flexso-owned org, consider whether a Flexso-owned GitHub org would be more appropriate long-term.
