# Architecture

Static site, no build step, no backend code in this repo.

- `index.html` — the whole app. Entirely driven by URL query params
  (`gist`, `gistId`, `file`) — no per-project hardcoding. Fetches the
  target Gist's raw content, infers form fields from whatever keys appear in
  the JSON array's objects, and POSTs the edited array back through a Pipedream
  relay (not directly to GitHub — the page never holds write credentials).
- **`relay` is not a param and must never become one.** `RELAY_URL` is a
  hardcoded constant in `index.html` — deliberately, as a security decision, not
  an oversight. The page POSTs the typed password to whatever that value is, so
  a param would let a crafted link redirect the password to an attacker's
  endpoint. Do not "restore" it as a missing feature. Rationale in `NOTES.md`,
  under "Design notes / why it's built this way" → "Relay URL is hardcoded,
  not a URL param", and repeated as a `ponytail:` comment above the constant.
- The Pipedream relay (shared across every project that uses this tool) is
  configured in Pipedream's UI, documented in `NOTES.md` — not repo code. It
  holds the GitHub token and the shared password; the page only ever sends the
  password, not the token.
- Hosting: GitHub Pages, deployed from this repo's default branch.

Full setup steps and design rationale: `NOTES.md`.
