# Git Connector (GitHub RAG Ingestion)

**Turn Git repositories into always-current RAG knowledge bases — cloned, chunked, embedded, and kept in sync on a schedule.**

| | |
|---|---|
| Plugin ID | `com.tyk.github-rag-ingest` |
| Tier | Community |
| Runs in | AI Studio only |
| Hooks | `studio_ui` (management UI) + scheduled tasks |
| Admin UI | **Git Connector** sidebar section: Dashboard (`/admin/git-connector`), Repositories, Jobs |
| Requires | AI Studio ≥ 2.6, < 3.0 |

## What it does

The Git Connector ingests the contents of Git repositories into AI Studio **RAG datasources**. For each repository you register, it:

1. Clones or fetches the repository (public, or authenticated with a personal access token or SSH key).
2. Filters files by path, glob mask, size limit, and ignore rules (`.gitignore` and a RAG-specific `.ragignore` are respected).
3. Splits files into chunks using a strategy suited to the content — plain text, code-aware, or markdown-heading-aware.
4. Writes the chunks, with rich metadata (file path, line numbers, commit SHA, a direct GitHub link, and more), into the datasource you assign.
5. On later runs, syncs **incrementally**: only files changed since the last ingested commit are re-processed, and chunks belonging to deleted or renamed files are removed.

Syncs can be triggered manually from the UI or run automatically on a per-repository cron schedule.

## When to use it

- **Chat over your documentation** — point it at your docs repo, attach the datasource to a chat experience, and answers stay current with every merge to `main`.
- **Codebase-aware assistants** — ingest source code with the code-aware chunker so retrieval returns whole functions and classes rather than arbitrary text windows.
- **Internal knowledge bases from private repos** — runbooks, ADRs, RFCs in private GitHub repos, authenticated via PAT or SSH.
- **Multi-tenant knowledge** — the datasource cloning helper makes it quick to ingest different repos into separate namespaces on the same vector database.

## Setting up a repository

In **Git Connector → Repositories → Add Repository** you configure:

- **Identity** — display name, owner, repository URL, and the branch to track.
- **Authentication** — `public` (none), `pat` (GitHub personal access token), or `ssh` (OpenSSH private key, with optional passphrase). Credentials are stored in the secrets backend, not in the repository record (see *Secrets* below).
- **Datasource** — the RAG datasource that receives the chunks. A **Clone** button next to the picker creates a copy of an existing datasource under a new name and vector-store namespace, so you don't have to re-enter connection details for each repository.
- **File filtering** — target paths (e.g. `docs/, src/`) and file masks (e.g. `*.md, *.go` — simple patterns automatically match subdirectories). Beyond these, fixed filters always apply: `.gitignore` and `.ragignore` files are respected, files over 1024 KB are skipped, and binary files are always skipped.
- **Chunking** — strategy and chunk size (default 1000 characters). Chunks overlap by 200 characters.
- **Schedule** — an optional cron expression (e.g. `0 2 * * *` for 2 AM daily); entering one enables automatic sync. Leave empty for manual-only syncing.

### Chunking strategies

| Strategy | Behavior | Best for |
|----------|----------|----------|
| `simple` | Fixed-size chunks with overlap, splitting on line boundaries | Plain text and general content |
| `code_aware` | Extracts functions, classes, and methods (Go, Python, JavaScript/TypeScript) | Source code |
| `hybrid` *(recommended)* | Picks per file: code files → code-aware, markdown → heading-based, everything else → simple | Mixed repositories |

### What lands in the vector store

Each chunk carries metadata usable in retrieval and for citations, including the file name and path, chunk position and line range, the branch and commit SHA at ingestion time, and a `github_url` deep-linking to the exact lines. (Deep links are built as github.com URLs, so they are only correct for repositories hosted on GitHub.)

## Syncing

- **Sync** (incremental) — diffs the tracked branch against the last synced commit; only added/changed files are re-chunked and re-embedded, and chunks of deleted or renamed files are cleaned up. This keeps embedding costs proportional to change volume.
- **Full Sync** — re-processes every matching file. Use after changing filters or chunking settings, since configuration changes only affect files processed in subsequent syncs.
- **Scheduled sync** — runs the incremental sync on the repository's cron expression (UTC).

Every run creates a **job** with status (`queued`, `running`, `success`, `failed`, or `cancelled`), statistics (files scanned/added/changed/deleted/skipped, chunks written/deleted, errors), and level-filterable execution logs — all visible under **Git Connector → Jobs**. A failed run doesn't stop the schedule; the next run proceeds normally.

Deleting a repository removes its configuration **and** every chunk it ingested from the vector store (the confirmation dialog says so before it happens).

## Plugin configuration

Global settings (per-repository settings live in the UI):

| Option | Default | Description |
|--------|---------|-------------|
| `secrets_backend` | `kv` | Where repository credentials are stored: `kv` (AI Studio's KV store) or `vault` (HashiCorp Vault) |
| `vault_address` | — | Vault server address (when using `vault`) |
| `vault_token` | — | Vault auth token |
| `vault_mount_path` | `secret` | Vault secrets mount |
| `vault_secret_path` | `github-rag` | Base path for this plugin's secrets in Vault |
| `cache_path` | `/tmp/github-rag-cache` | Directory where cloned repositories are cached between syncs (removed when the repository is deleted) |
| `cache_max_age_hours` | `168` | Declared in the schema but **not currently enforced** — no age-based cache cleanup runs today |

## Secrets

PATs, SSH keys, and passphrases are stored in the configured secrets backend — by default AI Studio's KV store; a HashiCorp Vault backend can be selected in configuration. They are write-only from the UI's perspective: entered when configuring a repository, then referenced internally.

> **Caveats:** In the current version, credentials stored via the Vault backend cannot be read back during syncs (authenticated syncs fail with an invalid secret reference), so use the default `kv` backend for now. Also, when *editing* a repository with PAT or SSH auth, re-enter the credential — saving the form re-stores whatever is in the credential fields.

## Things to know

- AI Studio needs network access to your Git host, and the assigned datasource's embedding provider is called during ingestion — large first-time syncs of big repositories consume embedding tokens accordingly.
- Only text content is ingested; binary files are skipped automatically.
- Filter or chunking changes apply from the next sync; run a **Full Sync** to re-process existing content under new settings.
