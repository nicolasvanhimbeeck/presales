# flexso-presales-mockups

Static presales mockups, one folder per client under `public/`, served via GitLab Pages
and gated by Cloudflare Access (see `.claude/plan.md` for the full design).

## Adding a new client mockup

1. Create a new folder under `public/`, e.g. `public/new-client-name/`
2. Drop the generated `index.html` (and any assets) in there
3. Commit + push to `main` → pipeline runs → live automatically
4. Add a Cloudflare Access Application + policy for that path before sharing the link
