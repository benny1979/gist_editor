# Conventions

- Single-file static site by design — don't split into multiple files or add a
  build step/framework.
- Never add per-project logic (hardcoded gist IDs, field names, schemas) to
  `index.html` — anything project-specific must come in via URL query params.
  If a new project's config shape needs something this page can't express
  generically, extend the generic form (e.g. support for nested objects), don't
  fork the file.
- No GitHub token, Pushover token, or any other secret ever belongs in this
  repo or in `index.html` — those live only in the Pipedream relay's
  environment variables.
