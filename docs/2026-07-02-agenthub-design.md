# AgentHub — Design Spec

**Status:** Approved (brainstorming complete)
**Date:** 2026-07-02
**Source input:** [agency-agents](https://github.com/msitarzewski/agency-agents) (MIT) — 232 specialized AI agent files across 16 divisions
**Owner:** Ritesh

---

## 1. Vision

AgentHub is the **npm for AI agents**: a central registry + publishing layer + CLI for AI agent personas (the Markdown-with-frontmatter files that turn a coding assistant into a specialized expert).

The wedge is the **registry + publishing layer**. The existing `agency-agents-app` is a desktop GUI browser for GitHub files; it does not solve centralized discovery, semantic versioning, creator monetization, quality control, dependencies, or enterprise governance. AgentHub does.

**Source data.** Each agent is a Markdown file with YAML frontmatter (`name`, `description`, `color`, optional `emoji`, `vibe`) plus a body of `##` sections. This format is already validated by upstream's `lint-agents.sh` and already transformed for 14 target tools by upstream's `convert.sh` + `lib.sh`. We adopt the format, port the transformers, and build the registry/publishing layer around them.

---

## 2. Key Decisions (locked in brainstorming)

| Decision | Choice | Rationale |
|---|---|---|
| Strategic wedge | Registry + publishing layer | The defensible moat; matches Red Hat + App Store monetization |
| CLI language | **Go** | Single static binary, fast startup, easy cross-compile; backend also Go — one language |
| Backend language | Go | Same as CLI; keeps the whole stack one language for Phase 1 |
| Phase 1 scope | Read-only registry + CLI | Smallest loop that proves the value prop; publish/auth/Stripe are Phase 2 |
| Canonical format | Extend agency-agents frontmatter | 232 agents import with zero content edits |
| Naming | npm-style scopes: `@scope/name` | Namespaces for free; Phase 1 ships `@agency/*` only |
| DB | PostgreSQL | Relational metadata |
| Search | Meilisearch | Typo-tolerant fuzzy search |
| Storage | Cloudflare R2 (S3-compatible) | Zero egress fees; same API as S3 |
| Web dashboard | Deferred to Phase 2 | CLI + registry prove the value prop first |
| Architecture | **Transform-at-install (CLI-side)** | Registry stays simple; CLI carries all adapters; clean upgrade path to paid server-side transformer API |

---

## 3. System Architecture

Four deployable pieces over one shared data plane.

```
                         ┌─────────────────────────────────────────┐
   developer machine     │            AgentHub Registry             │
  ┌───────────────┐      │                                          │
  │   agent CLI   │────▶ │   Registry API (Go)                      │
  │  (Go binary)  │ HTTP │   - GET /packages/{scope}/{name}         │
  │               │      │   - GET /packages/{scope}/{name}/{ver}   │
  │  • install    │      │   - GET /search?q=...                    │
  │  • search     │      │   - GET /catalog (divisions, tools)      │
  │  • update     │      │                                          │
  │  • init       │      │   Postgres (metadata)  Meilisearch (sx)  │
  │  • info       │      │          │                   │          │
  │  • (p2) pub   │      │          └────────┬──────────┘          │
  └───────┬───────┘      │                   ▼                      │
          │              │   Cloudflare R2 (canonical .md blobs)    │
          ▼              └─────────────────────────────────────────┘
   target tool dirs
   .cursor/rules/        Phase 1 = read-only: GET endpoints only, no auth.
   .claude/agents/       Transform runs in the CLI; registry only serves
   .gemini/agents/        canonical bytes + manifest JSON.
   ...14 tools
```

**The four pieces:**

1. **`agent` CLI (Go)** — the only thing a user installs. Contains: command parser, registry HTTP client, **all 14 format adapters** (ported from `convert.sh`), local config (`agent.config.json`), install/update logic, tool auto-detection. Single static binary; cross-compiled for `windows/amd64`, `darwin/amd64`, `darwin/arm64`, `linux/amd64`, `linux/arm64`. Distribution via `curl|bash` (Windows: PowerShell one-liner), Homebrew tap, Scoop/winget, and GitHub Releases.

2. **Registry API (Go)** — HTTP/JSON service. Phase 1 endpoints are read-only: package metadata, version manifest, search proxy to Meilisearch, catalog (divisions + supported tools). Stateless; horizontally scalable; sits behind a CDN for the read-heavy endpoints.

3. **Data plane** — Postgres for relational metadata (packages, versions, divisions, tools, stats), Meilisearch indexed from Postgres for fuzzy search, Cloudflare R2 for the canonical agent file blobs (keyed by content hash for dedup + immutability).

4. **Web dashboard (Phase 2)** — Next.js + Tailwind. Phase 1 ships a minimal static landing/catalog page only.

**Why this shape works for Phase 1 → Phase 2 migration:** the read-only API contract (`GET /packages/...`) is identical to what the publish flow will later write to. Adding `POST /packages`, auth, and Stripe changes nothing about how installs work — `agent install` keeps hitting the same GET endpoints. Phase 1 is a real subset, not a throwaway.

**Build-time input.** A one-shot importer (`cmd/import-agents`) reads the vendored agency-agents repo, parses frontmatter, uploads canonical blobs to R2, inserts metadata rows into Postgres + Meilisearch. The vendored copy is build-time input, not part of the shipped product.

---

## 4. The Canonical Format

The data contract every publisher, adapter, search index, and DB row keys off.

**Source:** the agency-agents frontmatter (validated by `lint-agents.sh`), extended with registry fields. Every existing field name and meaning is identical so the 232 agents import with zero content edits.

### 4.1 Frontmatter

```yaml
---
# --- Inherited from agency-agents (unchanged semantics) ---
name: Security Architect            # required, human display name
description: Expert security...     # required, one-line summary
color: red                          # required, named or #RRGGBB
emoji: "🛡️"                         # optional
vibe: Designs the security...       # optional

# --- AgentHub extensions (registry-aware) ---
version: 1.0.0                      # required at publish; SemVer
author: msitarzewski                # required at publish; maps to scope owner
license: MIT                        # required at publish; SPDX identifier
division: security                  # required; one of divisions.json keys
tags: [threat-modeling, zero-trust, appsec]   # optional, free-form
homepage: https://github.com/.../security-architect.md   # optional
keywords: [security, architecture]  # optional, search-boost only

# --- Reserved for Phase 2+ (validated-but-no-op in Phase 1) ---
# dependencies:
#   - "@agency/coding-standards": "^1.0.0"
# paid:
#   price_usd: 10.0
#   model: one-time
---

# Security Architect Agent

<the full markdown body, ## sections, etc — identical to upstream>
```

### 4.2 Naming & identity (npm-style scopes)

A package's fully-qualified identity is **`@scope/name`**:

- `scope` = publisher namespace. Reserved root scopes: `@agency` (official mirror of agency-agents), `@agenthub` (first-party curated). Personal/org scopes arrive with Phase 2 publishing.
- `name` = the **slugified** `name` frontmatter (`Security Architect` → `security-architect`). Slug rules ported verbatim from `lib.sh`'s `slugify()`: lowercase, non-alphanumeric → `-`, collapse repeats, trim ends.
- **Phase 1 ships only `@agency/*`** (232 packages). CLI accepts both `agent install @agency/security-architect` and the bare shorthand `agent install security-architect` (resolves to `@agency/` when unambiguous).

### 4.3 Versioning

- **SemVer** (`MAJOR.MINOR.PATCH`), stored as three integer columns + canonical string. Uses `golang.org/x/mod/semver` for parsing/comparison.
- Each published version is **immutable**: its canonical blob is stored in R2 keyed by `sha256(content)`, and the `(scope, name, version)` row points at it. Republishing the same version is rejected; bumping required. This makes `agent update` and lockfile reasoning sound.
- A package has exactly one `latest` pointer (highest non-prerelease version). Phase 1 importer tags every imported agent as `1.0.0` + `latest`.

### 4.4 The manifest (what the API serves)

The registry never serves raw `.md` alone — it serves a **manifest** JSON envelope so the CLI gets everything in one fetch:

```json
{
  "id": "@agency/security-architect",
  "scope": "agency",
  "name": "security-architect",
  "displayName": "Security Architect",
  "division": "security",
  "version": "1.0.0",
  "description": "Expert security architect specializing in...",
  "emoji": "🛡️",
  "color": "#E74C3C",
  "license": "MIT",
  "author": "msitarzewski",
  "tags": ["threat-modeling", "zero-trust"],
  "canonicalSha256": "ab12...ef",
  "canonicalUrl": "https://r2.../agency/security-architect/ab12ef.md",
  "canonicalSizeBytes": 8421,
  "publishedAt": "2026-07-02T13:00:00Z",
  "supportedFormats": ["identity","cursor-mdc","codex-toml","gemini-md", ...]
}
```

`supportedFormats` is **derived** from the global tools table (Section 6), not declared by the author — every canonical agent can transform into every format the engine knows. "This agent isn't available for Cursor" cannot happen in Phase 1.

### 4.5 Validation rules (enforced at import and publish)

One Go validator used by both the importer and the (Phase 2) publish endpoint:

1. File is UTF-8, **LF line endings** (CRLF rejected).
2. Starts with `---`, has a closing `---`, frontmatter parses as YAML.
3. Required: `name`, `description`, `color`, and (at publish) `version`, `author`, `license`, `division`.
4. `version` is valid SemVer with no `+build` metadata.
5. `division` ∈ the keys of the served `divisions.json` catalog.
6. `license` is a valid SPDX expression.
7. `color` is a known named color (from the upstream mapping table) or `#RRGGBB`.
8. Body word count ≥ 50 (warn in Phase 1; hard-fail at publish).
9. Body contains at least one `## Identity`-class header (warn).

### 4.6 Explicitly not in the format

- **No `tools` array** — author doesn't declare supported tools; the engine decides.
- **No `globs` / `alwaysApply`** in canonical — those are Cursor-output concerns, injected by the adapter.
- **No `dependencies` resolution in Phase 1** — field reserved, parsed, validated for shape, ignored.
- **No `private: true`** — enterprise namespaces are a Phase 4 concern.

---

## 5. Repository Layout & Naming

Monorepo, one Go module at the root, with clear subdirectory boundaries so each deployable/component is independently testable and buildable.

```
Agents-Package-Manager/
├── go.mod                          # module github.com/agenthub-dev/agenthub
│                                   # (replace `agenthub-dev` with the real GitHub owner/org)
├── go.sum
├── Makefile                        # build, test, lint, generate, release targets
├── README.md
├── LICENSE                         # MIT
├── .golangci.yml
├── .gitignore
│
├── cmd/                            # one subdir per binary entry point
│   ├── agent/                      # CLI binary: `agent`
│   │   └── main.go
│   ├── registry/                   # API server binary: `agenthub-registry`
│   │   └── main.go
│   └── import-agents/              # one-shot bulk importer
│       └── main.go
│
├── internal/                       # non-importable; only this module uses it
│   ├── cli/                        # CLI command layer (install, search, ...)
│   ├── server/                     # HTTP handlers, routing, middleware
│   ├── adapter/                    # ★ the 14 format transformers
│   ├── canonical/                  # frontmatter parse/validate/manifest
│   ├── slug/                       # slugify (ported from lib.sh)
│   ├── semver/                     # version wrap over x/mod/semver
│   ├── store/                      # data access: pg, r2, search backends
│   ├── httpclient/                 # registry client used by the CLI
│   ├── config/                     # agent.config.json read/write, tool detect
│   └── version/                    # build version injected via -ldflags
│
├── pkg/                            # importable; stable public API
│   └── agenthub/                   # shared types: Package, Version, Manifest
│                                   # (CLI ↔ API contract lives here)
│
├── deploy/                         # infra-as-code, not Go
│   ├── docker/
│   ├── compose/                    # docker-compose for local dev (pg, meili, minio)
│   └── terraform/
│
├── docs/
│   ├── architecture.md
│   ├── canonical-format.md
│   ├── adapter-spec.md
│   └── superpowers/specs/         # this doc
│
├── scripts/                        # dev/release tooling
│   ├── release.sh
│   └── seed-local.sh
│
├── test/
│   ├── integration/
│   ├── golden/                     # expected output per adapter per fixture
│   └── fixtures/                   # canonical sample agents
│
└── vendor/
    └── agency-agents/              # vendored upstream repo (importer input)
```

### 5.1 The adapter directory

```
internal/adapter/
├── adapter.go              # interface: Transform(manifest, body) ([]OutputFile, error)
├── registry.go             # map[string]Adapter keyed by format name; List()
├── color.go                # resolveColor(name) → #RRGGBB (ported from convert.sh)
├── slug.go                 # delegates to internal/slug
├── identity.go             # "identity" format (claude-code, copilot) — passthrough
├── cursor_mdc.go           # "cursor-mdc" → .cursor/rules/{slug}.mdc
├── codex_toml.go           # "codex-toml" with proper TOML escaping
├── gemini_md.go            # "gemini-md"
├── qwen_md.go              # "qwen-md" (with optional tools field)
├── opencode_md.go          # "opencode-md" with mode + color
├── skill_md.go             # "skill-md" (antigravity, osaurus) with slugPrefix
├── kimi_agent.go           # "kimi-agent" → agent.yaml + system.md pair
├── openclaw_workspace.go   # "openclaw-workspace" → SOUL/AGENTS/IDENTITY triple
├── aider_conventions.go    # "aider-conventions" roster accumulator
├── windsurf_rules.go       # "windsurf-rules" roster accumulator
└── *_test.go               # golden-driven, one per adapter
```

**Adapter contract:** an adapter never touches the filesystem or the network. It takes `(manifest, canonicalBody)` and returns `[]OutputFile` (filename + bytes). The CLI's installer decides *where* to write. This makes adapters unit-testable and lets the same adapter power `install` and a future `--dry-run`/`convert` command.

### 5.2 Naming conventions (project-wide)

- **Binaries:** `agent` (CLI), `agenthub-registry` (server), `agenthub-import` (importer).
- **Go packages:** single-word lowercase (`cli`, `server`, `adapter`, `canonical`).
- **CLI commands & flags:** `agent install <pkg> [--tool cursor] [--scope user|project]`, etc. POSIX long flags, single-dash single-char (`-v`, `-h`).
- **Config file:** `agent.config.json` (project root, project-scoped) and `~/.agenthub/config.json` (user defaults + auth token Phase 2).
- **Env vars:** prefixed `AGENTHUB_` (e.g. `AGENTHUB_REGISTRY_URL`).
- **HTTP API paths:** `/api/v1/...` — versioned from day one.

### 5.3 Why one Go module

Single module keeps `go mod tidy`, CI, and releases trivial. `internal/` enforces boundaries (CLI can't import server internals and vice versa). If the web dashboard (Phase 2) lands in this repo, it will be a separate `web/` subdir with its own `package.json` — the JS/Go split is the only one worth physically separating.

### 5.4 Build & release

- **Makefile:** `make build`, `make test`, `make test-integration` (needs `make dev-up`), `make lint`, `make generate`.
- **GoReleaser:** cross-compilation for 5 OS/arch targets, Homebrew tap updates, Scoop manifest, GitHub Releases artifacts. Triggered by git tag `v*`.
- **CI (GitHub Actions):** lint + test on every PR; build all binaries on main; release on tag. Three workflows.

---

## 6. Data Model

Postgres holds the relational truth; Meilisearch is a derived read-replica rebuilt from Postgres; R2 holds immutable blobs. Phase 1 schema is designed so Phase 2 publishing/accounts and Phase 4 enterprise namespaces are *additive* migrations, never rewrites.

### 6.1 Schema (Phase 1)

```sql
-- Namespaces: @scope. Phase 1 seeds one row: 'agency'.
CREATE TABLE scopes (
  id          BIGSERIAL PRIMARY KEY,
  name        TEXT NOT NULL UNIQUE,          -- 'agency', 'agenthub', '@acme'
  kind        TEXT NOT NULL CHECK (kind IN ('official','personal','org','private')),
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Packages: one row per (scope, slugified name).
CREATE TABLE packages (
  id            BIGSERIAL PRIMARY KEY,
  scope_id      BIGINT NOT NULL REFERENCES scopes(id),
  name          TEXT NOT NULL,                -- slug: 'security-architect'
  display_name  TEXT NOT NULL,                -- 'Security Architect'
  division      TEXT NOT NULL,
  description   TEXT NOT NULL,
  emoji         TEXT,
  color         TEXT NOT NULL,                -- normalized to #RRGGBB at write
  author        TEXT NOT NULL,
  license       TEXT NOT NULL,                -- SPDX id
  tags          TEXT[] NOT NULL DEFAULT '{}',
  keywords      TEXT[] NOT NULL DEFAULT '{}',
  homepage      TEXT,
  latest_version_id BIGINT REFERENCES package_versions(id) DEFERRABLE INITIALLY DEFERRED,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (scope_id, name)
);
CREATE INDEX packages_division_idx ON packages(division);
CREATE INDEX packages_tags_gin     ON packages USING gin(tags);
CREATE INDEX packages_name_trgm    ON packages USING gin(name gin_trgm_ops);

-- Versions: immutable rows. Content lives in R2, addressed by sha256.
CREATE TABLE package_versions (
  id                BIGSERIAL PRIMARY KEY,
  package_id        BIGINT NOT NULL REFERENCES packages(id) ON DELETE CASCADE,
  version           TEXT NOT NULL,              -- '1.0.0' (SemVer canonical form)
  major             INT NOT NULL,
  minor             INT NOT NULL,
  patch             INT NOT NULL,
  prerelease        TEXT,                       -- 'rc.1' or NULL
  canonical_sha256  TEXT NOT NULL,              -- content hash; immutability key
  canonical_size    INT NOT NULL,               -- bytes
  manifest_json     JSONB NOT NULL,             -- full manifest envelope
  published_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (package_id, version),
  UNIQUE (canonical_sha256)                     -- identical content shares a blob
);
CREATE INDEX versions_pkg_semver_idx ON package_versions(package_id, major DESC, minor DESC, patch DESC);

-- Catalog: divisions + supported tools. Mirrors upstream divisions.json/tools.json.
CREATE TABLE divisions (
  id      TEXT PRIMARY KEY,           -- 'security'
  label   TEXT NOT NULL,              -- 'Security'
  icon    TEXT NOT NULL,              -- 'ShieldCheck' (Lucide)
  color   TEXT NOT NULL,              -- '#EF4444'
  ord     INT NOT NULL,
  enabled BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE tools (
  id            TEXT PRIMARY KEY,     -- 'cursor'
  label         TEXT NOT NULL,
  kebab         TEXT NOT NULL,        -- 'cursor'
  format        TEXT NOT NULL,        -- 'cursor-mdc' → adapter registry key
  install_kind  TEXT NOT NULL CHECK (install_kind IN ('per-agent','roster','plugin')),
  slug_from     TEXT,                 -- 'name' | 'source' | NULL
  slug_prefix   TEXT,                 -- 'agency-' or NULL
  scope_user    BOOLEAN NOT NULL,
  scope_project BOOLEAN NOT NULL,
  detect_dirs   JSONB NOT NULL,       -- [".cursor"]
  dest_user     JSONB NOT NULL,       -- []
  dest_project  JSONB NOT NULL,       -- [".cursor/rules/{slug}.mdc"]
  ord           INT NOT NULL,
  enabled       BOOLEAN NOT NULL DEFAULT true
);

-- Stats: cheap counters, updated async.
CREATE TABLE package_stats (
  package_id     BIGINT PRIMARY KEY REFERENCES packages(id) ON DELETE CASCADE,
  install_count  BIGINT NOT NULL DEFAULT 0,
  updated_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE daily_installs (
  package_id BIGINT NOT NULL REFERENCES packages(id) ON DELETE CASCADE,
  day        DATE NOT NULL,
  count      BIGINT NOT NULL DEFAULT 0,
  PRIMARY KEY (package_id, day)
);
```

### 6.2 Why these shapes

- **`scopes` is its own table**, not a string column on `packages`. Phase 4 enterprise private registries are just `scopes` rows with `kind='private'` plus a membership join added later. Zero changes to `packages`.
- **`latest_version_id` is deferred-FK and nullable.** A package inserted before its first version is valid; the importer sets it in the same transaction.
- **`canonical_sha256` is globally unique** and is the R2 object key (`<scope>/<name>/<sha256[:12]>.md`). Two versions with identical content share one blob; content-addressing makes integrity checks trivial.
- **`manifest_json` is denormalized.** The API's hottest path returns the manifest. Storing it pre-built means one row read, no joins, cacheable as flat JSON at the CDN edge. Rebuilding it is the importer's/publisher's job.
- **SemVer stored three ways**: canonical string (identity), three int columns (efficient `ORDER BY`), prerelease segment (so `1.0.0-rc.1` sorts below `1.0.0`). Never parsed at read time.
- **`divisions`/`tools` are DB rows**, not config files. Add a tool or toggle `enabled=false` without a redeploy. Seeded from upstream JSON at importer time.

### 6.3 Meilisearch index (derived, rebuildable)

One index `agents` over a flattened projection of `packages` joined to their latest version:

```json
{
  "id": "@agency/security-architect",
  "scope": "agency",
  "name": "security-architect",
  "displayName": "Security Architect",
  "division": "security",
  "divisionLabel": "Security",
  "description": "...",
  "tags": ["threat-modeling","zero-trust"],
  "keywords": ["security","architecture"],
  "emoji": "🛡️",
  "version": "1.0.0",
  "author": "msitarzewski"
}
```

Searchable attributes: `displayName`, `name`, `description`, `tags`, `keywords`, `divisionLabel`. Ranking: downloads desc, then exactness. Rebuilt from Postgres by `make search-reindex` (and on every importer/publish run). Phase 1 does not need live incremental sync.

### 6.4 Migrations

- **`golang-migrate`** with versioned `up`/`down` SQL files under `internal/store/migrations/`. CI runs `migrate up` against a fresh containerized PG before tests.
- The Phase 1 schema is migration `0001_init.up.sql`. Phase 2 adds `0002_accounts.up.sql`, `0003_publishing.up.sql`; Phase 4 adds `0004_enterprise_namespaces.up.sql`. No destructive migrations in 1.x.

### 6.5 Phase 1 read paths

| Endpoint | Query shape |
|---|---|
| `GET /api/v1/catalog` | `SELECT * FROM divisions ORDER BY ord` + `SELECT * FROM tools WHERE enabled ORDER BY ord` |
| `GET /api/v1/packages/{scope}/{name}` | one row on `packages` join `package_versions` on `latest_version_id` |
| `GET /api/v1/packages/{scope}/{name}/versions` | `SELECT version FROM package_versions WHERE package_id=? ORDER BY major,minor,patch` |
| `GET /api/v1/packages/{scope}/{name}/{ver}` | `package_versions` row by `(package_id, version)`; return `manifest_json` + redirect URL to R2 |
| `GET /api/v1/search?q=` | proxy to Meilisearch; PG not in the hot path |

Every read is a single-table or single-join lookup by indexed key. No N+1, no aggregations at request time.

---

## 7. The Adapter Engine

The subsystem most likely to grow messy; the design's whole job is to make adding a 15th tool a 30-minute mechanical task with one new file and zero edits elsewhere.

### 7.1 The contract

```go
package adapter

// OutputFile is a logical artifact: a path relative to the install root
// plus its bytes. Adapters don't know whether they're writing to user-scope
// or project-scope — the installer applies the dest template from the tools table.
type OutputFile struct {
    Path string   // e.g. "rules/security-architect.mdc" or "system.md"
    Data []byte
}

// Adapter transforms one canonical agent into one or more output files
// for a target tool's format. Pure function: same input → same bytes.
type Adapter interface {
    Format() string                          // "cursor-mdc"; registry key
    InstallKind() InstallKind                // PerAgent | Roster | Plugin
    Transform(m *agenthub.Manifest, body []byte) ([]OutputFile, error)
}

type InstallKind int
const (
    PerAgent InstallKind = iota   // one render per agent (cursor, claude-code, ...)
    Roster                         // one combined file for all agents (aider, windsurf)
    Plugin                         // built artifact, not per-agent renderable (hermes)
)
```

Roster and plugin kinds need different orchestration than `per-agent`.

### 7.2 The registry

```go
package adapter

var registry = map[string]Adapter{}

func Register(a Adapter) { registry[a.Format()] = a }
func Get(format string) (Adapter, bool) { a, ok := registry[format]; return a, ok }
func List() []Adapter { /* sorted by Format() */ }

// init() in each adapter file calls Register(&CursorMDCAdapter{}).
// A single blank-import in cmd/agent/main.go pulls them all in:
//   _ "github.com/agenthub-dev/agenthub/internal/adapter"
//   (same owner placeholder as Section 5)
```

Adding tool #15 = create `internal/adapter/mytool_xxx.go` with an `init()` Register call, add a row to the `tools` catalog, write a golden test. No core code changes.

### 7.3 Per-tool behavior (ported faithfully from `convert.sh`)

Each adapter is a direct Go port of the corresponding `convert_<tool>()` shell function. The shell is the spec; we match its output **byte-for-byte** against golden files captured from upstream `integrations/<tool>/` output.

| Format | Adapter file | Behavior (vs upstream `convert.sh`) |
|---|---|---|
| `identity` | `identity.go` | Frontmatter + body passed through unchanged. Used by `claude-code`, `copilot`. |
| `cursor-mdc` | `cursor_mdc.go` | Replaces frontmatter with `description`/`globs:""`/`alwaysApply:false`; body verbatim. |
| `codex-toml` | `codex_toml.go` | TOML basic string with full escaping (`\n`, `\r`, `\t`, control chars as `\uXXXX`). |
| `gemini-md` | `gemini_md.go` | New frontmatter `name:<slug>`/`description:<desc>` + body. |
| `qwen-md` | `qwen_md.go` | Like gemini-md but includes `tools:` line only if present in source. |
| `opencode-md` | `opencode_md.go` | Adds `mode: subagent` and `color:` resolved to `#RRGGBB`. |
| `skill-md` | `skill_md.go` | `name: agency-<slug>` prefix + `description` + body. Drives `antigravity`, `osaurus`. |
| `kimi-agent` | `kimi_agent.go` | Two outputs: `agent.yaml` + `system.md`. |
| `openclaw-workspace` | `openclaw_workspace.go` | **Three** outputs: splits body's `##` sections into SOUL.md vs AGENTS.md by keyword; IDENTITY.md from emoji/vibe or name/description. |
| `aider-conventions` | `aider_conventions.go` | Roster: one `CONVENTIONS.md` concatenating all agents with `---`/`## Name`/`> desc` separators. |
| `windsurf-rules` | `windsurf_rules.go` | Roster: one `.windsurfrules` with heavy `===` separators. |
| `hermes-router-plugin` | *(Phase 1 stub)* | Plugin kind; built by upstream's `build-hermes-plugin.py`. Phase 1 declares it but returns `ErrNotImplemented`. |

**Color resolution** (`color.go`) is a direct port of upstream's `resolve_opencode_color()` case table (22 named colors → hex), plus pass-through for `#RRGGBB` / `RRGGBB`, plus fallback `#6B7280`.

**Slug derivation** is centralized in `internal/slug` and respects the tool's `slugFrom` / `slugPrefix` from the tools catalog.

### 7.4 Orchestration

The installer branches on `tools.install_kind`:

```go
switch tool.InstallKind {
case PerAgent:
    // one Transform() per agent, write each OutputFile to dest template
case Roster:
    // collect all selected agents, pass them all to one RosterAdapter
case Plugin:
    // fetch pre-built artifact from registry (Phase 1: hermes → friendly error)
}
```

Roster adapters get a different entry point since they need *all* agents at once:

```go
type RosterAdapter interface {
    Adapter
    TransformRoster(items []RosterItem) ([]OutputFile, error)
}
type RosterItem struct {
    Manifest *agenthub.Manifest
    Body     []byte
}
```

`aider-conventions` and `windsurf-rules` implement both (the single-agent `Transform` returns the body wrapped once, useful for `agent convert --tool aider <one-agent>`).

### 7.5 Where adapters get their input

The CLI fetches once per agent install: the **manifest** (JSON, tiny, cacheable) and the **canonical body** (bytes, cacheable by sha256). Adapters never re-fetch; they never parse frontmatter themselves — they read fields off the already-parsed `Manifest`, and receive `body` already stripped of frontmatter. This keeps each adapter under ~60 lines.

### 7.6 Testing strategy — golden files

Every adapter has a golden test driven by real upstream output:

```
test/
├── fixtures/
│   ├── security-architect.md          # canonical input (real upstream file)
│   ├── frontend-developer.md
│   └── ...                             # ~5 agents chosen for variety
└── golden/
    └── cursor-mdc/
        ├── security-architect.mdc      # captured from upstream integrations/cursor/rules/
        └── ...
```

Each adapter test runs `Transform()` on each fixture and asserts `bytes.Equal(output, goldenFile)`. When upstream `convert.sh` changes a format's output, goldens regenerate via `make generate-golden` (shells out to upstream's `convert.sh` for ground truth) and the diff is reviewed. **Byte-identical output to the shell = provably correct port.**

`openclaw-workspace` and `kimi-agent` (multi-file outputs) get one golden *directory* each, compared with `filepath.Walk` + per-file `bytes.Equal`.

### 7.7 Determinism guarantee

Every adapter's output is a pure function of `(manifest, body)`. No timestamps, no random IDs — directly matches upstream's "deterministic — no date stamp" comment. Two installs of the same version produce identical bytes, making `agent update`'s "has this changed?" check a trivial hash compare.

---

## 8. CLI UX

The CLI is the product in Phase 1. Every command must feel instant, predictable, and scriptable. North star: a developer installs an agent into Cursor in under 10 seconds, including the first-ever run.

### 8.1 Command surface (Phase 1)

```
agent <command> [flags] [args]

Commands:
  init          Detect installed tools, write agent.config.json
  install       Install one or more agents for a target tool
  update        Bring installed agents to their latest (or locked) versions
  search        Search the public registry
  info          Show details for one package
  list          List agents installed locally or available in a division
  which         Show which tool/config the CLI will use
  config        Get/set local config values
  doctor        Diagnose environment: connectivity, tool detection, perms
  version       CLI + registry version
  help          Top-level and per-command help

Flags (global):
  --registry <url>        override registry base URL (env: AGENTHUB_REGISTRY_URL)
  --tool <name>           target tool for this invocation (skip prompts)
  --scope user|project    install scope (default: per tool's capability + config)
  --non-interactive       never prompt; fail on ambiguity (for CI)
  -v, --verbose           debug logging
  -q, --quiet             errors only
  -h, --help
```

Publishing commands (`login`, `publish`, `unpublish`, `deprecate`) are **not** in Phase 1 — stubbed in help as "(coming in Phase 2)".

### 8.2 The seven substantive Phase 1 commands

(Plus four trivial ones not detailed here: `which`, `config`, `version`, `help` — standard getters/usage, each <50 lines.)

**`agent init`** — One-shot bootstrap. Scans for the 14 tools (using `detect_dirs` from the tools catalog), writes `agent.config.json` in cwd. Non-interactive by default; `--interactive` opens a checkbox picker.

```json
{
  "version": 1,
  "registry": "https://registry.agenthub.dev",
  "tools": {
    "cursor":     { "enabled": true, "scope": "project" },
    "claude-code": { "enabled": true, "scope": "user" }
  },
  "installed": {}
}
```

The `installed` map is the local ledger of what's been installed where, keyed by `@scope/name` → `{version, tool, path, sha256}`. This is what `update` diffs against. One source of truth at each scope.

**`agent install <pkg>[@version] [--tool <t>]`** — Resolution order:
1. **Package spec**: `security-architect` → `@agency/security-architect` (Phase 1 shorthand); `@agency/security-architect` literal; `@agency/security-architect@1.2.0` pinned.
2. **Tool selection**: `--tool` → else the only enabled tool → else if `--non-interactive` fail, else prompt.
3. **Fetch**: `GET manifest` + `GET canonical body` (cached by sha256 in `~/.agenthub/cache/`).
4. **Verify**: recompute sha256 of body, compare to `manifest.canonicalSha256`. Mismatch = hard error.
5. **Transform**: dispatch to adapter by tool's `format`.
6. **Write**: apply `dest` templates to each `OutputFile`; create parent dirs; write atomically (temp + rename).
7. **Record**: update `installed` map; bump `package_stats.install_count` fire-and-forget.

Multiple packages: `agent install frontend-developer security-architect --tool cursor` loops, batching manifest fetches. `agent install --division security --tool cursor` installs every agent in a division. `--dry-run` shows files that *would* be written without touching disk.

**`agent update`** — Walks the `installed` map; for each, fetches latest manifest; if `latest_version != installed_version`, runs the install flow. `agent update security-architect` updates one; `agent update` updates all. Respects a lockfile if present. Prints a summary: `package | had | now | changed?`.

**`agent search <query>`** — Hits `/api/v1/search?q=`. Output:

```
NAME                              DIVISION       VERSION   DOWNLOADS
@agency/security-architect        security       1.0.0     1,204
@agency/cloud-security-architect  security       1.0.0       487
```

`--division`, `--tag`, `--json` (for `jq`/`fzf`), `--limit`, `--offset`. Empty query returns a curated "browse" view grouped by division.

**`agent info <pkg>`** — Full manifest, human-formatted:

```
@agency/security-architect  1.0.0  (latest)
Division:   security   🛡️
Author:     msitarzewski   License: MIT   Size: 8.4 KB
Description: Expert security architect specializing in threat modeling...
Tags:       threat-modeling, zero-trust, appsec
Available formats: identity, cursor-mdc, codex-toml, gemini-md, opencode-md, +7 more
Install:    agent install @agency/security-architect --tool cursor
```

**`agent list [--tool <t>] [--division <d>]`** — No args: everything in the local `installed` map with versions and paths. `--division`: lists all packages in a division from the registry.

**`agent doctor`** — "Why isn't this working" command. Checks registry reachability + version compat; tool detection per enabled tool; write permissions on each dest dir; cache integrity; config file validity. Exits non-zero if broken. The command we ask users to paste output of when filing issues.

### 8.3 Default flag behaviors

- **Tool default**: if exactly one tool detected and `--tool` absent, use it silently. If multiple and not interactive, error with the list. Kills the most common "which tool?" friction.
- **Scope default**: prefer `project` for tools that support it (Cursor, Aider, Windsurf, OpenCode, Qwen), `user` otherwise. `--scope` overrides.
- **Non-interactive in CI**: `AGENTHUB_NON_INTERACTIVE=1` (auto-set if `CI=true`) flips every prompt to an error. Combined with explicit `--tool`, installs are fully scriptable.

### 8.4 Output & UX principles

- **Color when TTY, plain when piped.** Respect `NO_COLOR` and `isatty(stdout)`. Minimal box style; no heavy TUI for Phase 1 (no `bubbletea`).
- **Progress on long ops**: `install --division all` shows `[12/232] writing cursor rules…` on stderr so stdout stays clean.
- **Errors are actionable**: every error names the command, input, and suggested fix. E.g. `ERROR: tool "cursor" requires --scope project but no agent.config.json was found. Run 'agent init' first, or pass --scope user.` Never just "failed".
- **Exit codes**: `0` success, `1` usage, `2` no network/registry, `3` verification failure (sha mismatch), `4` partial install. Scriptable.

### 8.5 Install ledger & optional lockfile

The `installed` map in `agent.config.json` is the runtime ledger — what's actually on disk. Separately, users may commit an `agent.lock` (optional; generated by `agent install --save` or `agent update --save`) for reproducibility:

```json
{
  "packages": {
    "@agency/security-architect": "1.0.0",
    "@agency/frontend-developer": "1.0.0"
  },
  "tools": ["cursor"]
}
```

`agent install` with a lockfile present installs the locked versions; `agent update` refuses to advance past lockfile pins unless `--lockfile-bump`. Teams get CI reproducibility without forcing it on individuals — the npm `package.json` + `package-lock.json` split, deliberately echoed.

### 8.6 Day-1 distribution

- **One-liner**: `curl -fsSL https://registry.agenthub.dev/install.sh | bash` (posix) / `irm https://registry.agenthub.dev/install.ps1 | iex` (Windows). Detects OS/arch, fetches the right binary from GitHub Releases, installs to `/usr/local/bin` or `%USERPROFILE%\.agenthub\bin`.
- **Homebrew tap** and **Scoop bucket** for the package-manager crowds.
- Install scripts versioned in the registry repo, served from the registry's own static path.

---

## 9. Phase 1 Build Plan

### 9.1 Definition of done

A user on a fresh Windows/macOS/Linux machine can:

1. Install the `agent` CLI via one-liner.
2. Run `agent init` and have it detect Cursor (or any installed tool).
3. Run `agent search security` and see real results from the live registry.
4. Run `agent install @agency/security-architect --tool cursor` and get a working `.mdc` file written to `.cursor/rules/` that is **byte-identical** to what upstream's `convert.sh` produces.
5. Run `agent update` after we bump a version server-side and get the new bytes.

### 9.2 Build sequence (each milestone independently shippable)

**M1 — Foundation (no network, no DB yet)**
- Go module init, repo skeleton per Section 5.
- `pkg/agenthub` types: `Manifest`, `Package`, `Version`, `OutputFile`.
- `internal/slug`, `internal/semver`, `internal/canonical` (parse + validate).
- `Makefile`: `build`, `test`, `lint`.
- *Done when:* `go test ./...` passes on slug/semver/canonical units; `make build` produces three binaries.

**M2 — The adapter engine, end-to-end (still no network)**
- `internal/adapter` interface + registry.
- Port `color.go` + slug helpers from `lib.sh`.
- Port **three** adapters first as trailblazers: `identity`, `cursor-mdc`, `codex-toml` (trivial, frontmatter-rewrite, real-escaping — covers the three difficulty classes).
- `test/golden/` harness: fixtures + golden capture from upstream `convert.sh`.
- *Done when:* those three adapters produce byte-identical output vs upstream across the 5 fixture agents.

**M3 — All adapters**
- Port remaining 9 adapters (`gemini-md`, `qwen-md`, `opencode-md`, `skill-md`, `kimi-agent`, `openclaw-workspace`, `aider-conventions`, `windsurf-rules`, `hermes` stub).
- *Done when:* full golden suite green for all 11 implemented formats; hermes returns its documented error.

**M4 — Local data plane + importer**
- `deploy/compose` with Postgres + Meilisearch + MinIO.
- `internal/store` with pgx pool, migrations via golang-migrate (Section 6 schema = `0001_init`).
- `cmd/import-agents/main.go`: reads `vendor/agency-agents/**/*.md`, validates, uploads canonical blobs to MinIO, inserts packages + versions, sets `latest_version_id`, seeds `divisions` + `tools`, builds Meilisearch index.
- *Done when:* `make import` against local compose loads all 232 agents; `SELECT count(*) FROM packages` = 232.

**M5 — Registry API (read-only)**
- `cmd/registry/main.go` + `internal/server`: five GET endpoints, versioned `/api/v1/`.
- `pkg/agenthub` extended with API response shapes.
- MinIO → R2 swap is config-only.
- *Done when:* `curl /api/v1/packages/agency/security-architect` returns the manifest; `curl /api/v1/search?q=security` returns hits; dockerized registry passes integration tests.

**M6 — The CLI client half**
- `internal/httpclient` (registry client with caching by sha256).
- `internal/config` (read/write `agent.config.json`, tool detector using the catalog's `detect_dirs`).
- *Done when:* `agent search`, `agent info`, `agent list` work against the local registry.

**M7 — The install/update flow**
- `agent install` full pipeline (Section 8): resolve → fetch → verify sha → transform → write atomically → record ledger.
- `agent update`, `agent init`, `agent which`, `agent doctor`, `agent version`.
- Install ledger + optional `agent.lock`.
- *Done when:* end-to-end test installs `security-architect` to `.cursor/rules/` and the written file matches the M3 golden for cursor-mdc.

**M8 — Distribution & polish**
- GoReleaser config: 5 OS/arch targets, Homebrew tap, Scoop manifest, GitHub Releases.
- `install.sh` + `install.ps1` one-liners, served by the registry.
- `agent doctor` hardened; error messages audited.
- README + `docs/` filled in.
- *Done when:* a teammate can install via the one-liner on a clean machine and complete the Phase-1 DoD loop without help.

### 9.3 Out of scope for Phase 1

| Feature | Why deferred | Phase |
|---|---|---|
| `agent publish`, accounts, auth | Needs registry write path + identity model | P2 |
| Web dashboard (Next.js) | CLI + registry prove the value prop | P2 |
| Premium / paid agents (Stripe) | Needs accounts + publish first | P3 |
| Enterprise private registries, SSO | Needs scopes `kind='private'` + membership; table ready, joins not | P4 |
| Dependency resolution (`dependencies:`) | Field reserved + validated, no resolver | P3 |
| Hermes adapter | Needs Python at build time; <1% of users. Returns documented `ErrNotImplemented` | P2 |
| Real-time Meilisearch sync | Phase 1 rebuilds index on import/publish | P2 |
| TUI wizard (interactive `init`) | Plain `--interactive` checkbox is fine for Phase 1 | P3 |
| Self-hosted registry packaging (Helm) | One hosted registry for Phase 1 | P4 |

### 9.4 Risks

1. **Upstream `convert.sh` drift.** Mitigation: pin a specific upstream commit in `vendor/agency-agents`; regenerate goldens deliberately when bumping. The vendored copy is the contract, not `main`.
2. **Tool detection on Windows.** `detect_dirs` assume POSIX home. Mitigation: respect `USERPROFILE`/`HOME` per-OS; `agent doctor` is the smoke test.
3. **Naming squatting once publishing opens.** Phase 1 has only `@agency/*` so no risk now; Phase 2 needs a claim/trademark policy before launch.
4. **Registry cost at scale.** Manifests and canonical blobs cache well; `/search` (Meilisearch proxy) is the watch item in Phase 2.

### 9.5 The single Phase 1 deliverable

A 60-second demo: clone a fresh project, `agent init`, `agent install @agency/security-architect --tool cursor`, open Cursor, type "use the security architect agent to review this code", watch it work. If that loop runs against a deployed registry on day one of launch, Phase 1 is done.

---

## Appendix A — Source repo facts (agency-agents)

- License: **MIT** (verified in `LICENSE`). Repackaging commercially permitted.
- **232 agents** across **16 divisions**: academic, design, engineering, finance, game-development, gis, marketing, paid-media, product, project-management, sales, security, spatial-computing, specialized, support, testing.
- Canonical frontmatter fields: `name`, `description`, `color` (required); `emoji`, `vibe` (optional).
- Required body sections (warned): `Identity`, `Core Mission`, `Critical Rules`. Min 50 body words.
- **14 target tools** with a fully-specified install contract in `tools.json` (format, installKind, slugFrom, dest templates, detect dirs).
- Transformation engine: `scripts/convert.sh` (~700 lines) + `scripts/lib.sh` (slug, frontmatter, color helpers).
- Existing competitor: `agency-agents-app` (native desktop, click-to-install + auto-update). AgentHub's wedge vs. it: registry + publishing + versioning + discovery + monetization + governance.
