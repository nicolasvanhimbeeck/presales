# Sharing Presales Mockups Securely: GitLab Pages + Cloudflare Access
 
## Goal
 
Host HTML mockups on GitLab Pages, organized per client in separate folders, with real
server-side access control per client (not a client-side password).
 
- Client A can only access `/credendo-p2p/`
- Client B can only access `/digipolis-btp/`
- No password ever ships inside the HTML/JS itself
---
 
## Part 1 — Repo structure (do this first, in VS Code)
 
Create (or reuse) a dedicated GitLab repo for presales mockups, e.g. `flexso-presales-mockups`.
 
```
flexso-presales-mockups/
├── .gitlab-ci.yml
├── public/
│   ├── credendo-p2p/
│   │   └── index.html
│   ├── digipolis-btp/
│   │   └── index.html
│   └── index.html          (optional landing page, or leave empty/404)
└── README.md
```
 
Each client's mockup(s) go in their own subfolder under `public/`. GitLab Pages serves
everything under `public/` as static files, so the final URLs will look like:
 
```
https://<group>.gitlab.host/credendo-p2p/
https://<group>.gitlab.host/digipolis-btp/
```
 
(Exact domain depends on your GitLab Pages config — see Part 2.)
 
### `.gitlab-ci.yml`
 
```yaml
pages:
  stage: deploy
  script:
    - echo "Deploying static mockups"
  artifacts:
    paths:
      - public
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
```
 
That's it — GitLab Pages just needs a `public/` folder as a pipeline artifact.
 
### Adding a new client mockup
 
1. Create a new folder under `public/`, e.g. `public/new-client-name/`
2. Drop the generated `index.html` (and any assets) in there
3. Commit + push to `main` → pipeline runs → live automatically
---
 
## Part 2 — Enable GitLab Pages
 
1. In the GitLab project → **Settings → Pages**
2. Confirm Pages is enabled for the project (may need to ask a GitLab admin if it's disabled instance-wide)
3. After the first successful pipeline run, the Pages URL appears here
4. Optional: set up a custom domain (e.g. `mockups.flexso.be`) under the same settings page if you want cleaner URLs than the default `*.gitlab.host` one
At this point, everything is **publicly reachable by URL** — no access control yet. That's expected; access control is added in front of it via Cloudflare.
 
---
 
## Part 3 — Put Cloudflare Access in front of it
 
Cloudflare Access sits between the visitor and GitLab Pages, checking identity (via email one-time code) **before** any HTML is served. The password/OTP never touches the file itself.
 
### 3.1 Add the domain to Cloudflare
 
1. Create a free Cloudflare account (or use an existing Flexso one)
2. Add the domain you'll use for mockups (e.g. `mockups.flexso.be`) — if you don't have a spare domain/subdomain, ask whoever manages Flexso's DNS to delegate a subdomain to Cloudflare
3. Point that subdomain's DNS (CNAME) to your GitLab Pages URL
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
4. Only then does GitLab Pages content load
No password lives in the HTML. No file gets flagged by mail/SharePoint scanners, because you're only ever sharing a link.
 
---
 
## Part 4 — Sending the link to the client
 
Just share the URL directly (chat, email body, whatever) — since there's no attachment, mail filters have nothing to flag. If the client isn't already on the Access allow-list, add their email under the relevant Application's policy before sending.
 
---
 
## Notes / things to double check
 
- **Domain ownership**: you need control over a domain/subdomain's DNS to point it at Cloudflare. Check with whoever manages `flexso.be` DNS if `mockups.flexso.be` (or similar) is available to delegate.
- **Free tier limits**: Cloudflare Access free tier supports up to 50 users — almost certainly enough for presales use, but worth knowing.
- **Revoking access**: removing a client's email from the Application policy immediately cuts off access — useful once a deal closes or falls through.
- **Multiple mockups per client**: just add more subfolders under the same client path; the Cloudflare Application rule covers the whole path prefix.