# Gist Config Editor

A generic, reusable admin form for editing a JSON-array Gist that some other
project uses as its live config (e.g. `qr_notify`'s `buttons.json`). Built once,
reused across future projects of this shape by pointing it at a different Gist —
no code changes needed per project.

```
Admin page (this repo, GitHub Pages)
  -> reads current content from a Gist's raw URL, renders one form row per array entry,
     with one text input per key found in the data (inferred, not hardcoded)
  -> on Save, POSTs {password, gistId, filename, content} to a shared Pipedream relay
       -> relay checks the password, then calls GitHub's API with a stored token
            -> updates the Gist
```

The page never holds a GitHub token — only the Pipedream relay does. The page
itself is safe to be public; the password + relay are what gate the actual write.

## URL params (bookmark one set per project)

```
https://<user>.github.io/gist_editor/?gist=<raw-fetch-url>&gistId=<id>&file=<filename>&relay=<relay-webhook-url>
```

- `gist` — the Gist's raw content URL (same one the target project's page fetches).
- `gistId` — the Gist's ID (the hex string in its URL), needed to write back.
- `file` — filename inside the Gist, e.g. `buttons.json`.
- `relay` — the shared Pipedream relay's webhook URL (same for every project).

Example for `qr_notify`:
```
https://<user>.github.io/gist_editor/?gist=https://gist.githubusercontent.com/<user>/<gistId>/raw/buttons.json&gistId=<gistId>&file=buttons.json&relay=<relay-url>
```

## One-time setup

### 1. GitHub token for the relay
Create a classic Personal Access Token at github.com/settings/tokens with only
the **gist** scope checked. This is what lets the relay write to your gists —
keep it out of any repo, it only ever goes into Pipedream's environment
variables.

### 2. Pipedream relay workflow (shared across all projects using this tool)
- New workflow, trigger = **HTTP / Webhook**. Copy its URL — this is `relay` above.
- Set environment variables (Pipedream account Settings -> Environment Variables):
  - `RELAY_PASSWORD` — a password you choose, typed into the admin page's password field.
  - `GITHUB_TOKEN` — the token from step 1.
- Add a **Node.js** code step:

```javascript
export default defineComponent({
  async run({ steps, $ }) {
    const { password, gistId, filename, content } = steps.trigger.event.body;

    if (password !== process.env.RELAY_PASSWORD) {
      return $.respond({ status: 401, body: { error: "Wrong password" } });
    }

    const res = await fetch(`https://api.github.com/gists/${gistId}`, {
      method: "PATCH",
      headers: {
        "Authorization": `token ${process.env.GITHUB_TOKEN}`,
        "Content-Type": "application/json",
        "Accept": "application/vnd.github+json",
      },
      body: JSON.stringify({ files: { [filename]: { content } } }),
    });

    if (!res.ok) {
      return $.respond({ status: 502, body: { error: `GitHub API error ${res.status}` } });
    }

    return $.respond({ status: 200, body: { ok: true } });
  },
});
```

- Deploy the workflow.

### 3. Use it
Bookmark the full URL (with all four params) for each project's config on
whatever device you'll edit it from — phone home screen, PC bookmark bar, etc.
Opening it shows the current entries as editable rows; **+ Add entry** appends a
blank row (with the same fields as existing ones); **✕** removes a row; **Save
to Gist** writes the whole array back.

## Design notes / why it's built this way

- **Form fields are inferred from the data**, not hardcoded per project — reuse
  across future Gist-backed configs requires zero code changes, only different
  URL params.
- **Password + relay, not a token in the page** — the same reasoning as
  `qr_notify`'s Pushover relay: a public GitHub Pages page can't hold a secret
  that grants write access to your gists.
- **One shared relay workflow**, not one per project — `gistId`/`filename` are
  passed per-request, so adding a new project's admin page later needs no new
  Pipedream workflow, just a new bookmarked URL.
- Password is a light gate, not strong access control (same caveat as elsewhere
  in these projects) — good enough for "don't let a stray link let anyone edit
  my config," not intended to resist a determined attacker.
