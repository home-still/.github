# Home Still Style Guide

**Version:** 1.0  
**Applies to:** All repositories in the `home-still` GitHub organization  
**Canonical location:** `home-still/.github/HOME_STILL_STYLE_GUIDE.md`

This document is the authoritative reference for every agent, contributor, and code generation session working on any Home Still project. When in doubt, follow this guide. When this guide conflicts with a local README, this guide wins.

---

## 1. Repository Structure

### 1.1 The Three-Repo Rule

The Home Still org maintains exactly three categories of repository:

| Category | Example | Rule |
|----------|---------|------|
| Shared library | `hs-style` | Standalone repo, consumed as git submodule |
| Standalone tool | `paper-fetch`, `pdf-mash`, `librarian` | Own repo, own releases, own README |
| Unified CLI | `hs` | Own repo, depends on tool `-core` crates via git deps |

Do not create a monorepo. Do not create a new repo for a crate that belongs inside an existing tool's workspace.

### 1.2 Every Tool Repo Layout

```
<tool-name>/
├── .git/
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── release.yml
├── Cargo.toml                  # Workspace virtual manifest
├── Cargo.lock
├── README.md
├── CHANGELOG.md
├── LICENSE
├── hs-style/                   # Git submodule — always present
│   └── ...
├── crates/
│   ├── <tool>-core/            # Pure business logic — NO CLI deps
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── models.rs
│   │       ├── error.rs
│   │       ├── config.rs
│   │       └── ...
│   └── <tool>/                 # Binary crate — CLI, output, hs-style
│       ├── Cargo.toml
│       └── src/
│           ├── main.rs         # Minimal: parse args, detect mode, call run()
│           ├── cli.rs          # All clap structs
│           ├── commands/
│           │   ├── mod.rs
│           │   └── <noun>.rs   # One file per subcommand noun
│           └── output.rs       # All display/formatting functions
├── docker/
│   └── Dockerfile
└── config/
    └── default.yaml
```

The `-core` / binary split is **mandatory**. The `hs` unified CLI depends on `-core` crates. Business logic that lives only in the binary crate cannot be reused.

### 1.3 The `hs-style` Submodule

Every tool repo adds `hs-style` as a git submodule at the root:

```bash
git submodule add https://github.com/home-still/hs-style hs-style
```

Reference it in the workspace `Cargo.toml`:

```toml
[workspace]
resolver = "2"
members = ["crates/*", "hs-style"]

[workspace.dependencies]
hs-style = { path = "hs-style" }
```

Never copy `hs-style` source files into a tool repo. Never vendor it. Always submodule.

### 1.4 Config Directory

All tools read and write config under `~/.home-still/`. The layout:

```
~/.home-still/
├── config.yaml                 # Global config (applies to all tools)
├── paper-fetch/
│   └── config.yaml             # Tool-specific config
├── librarian/
│   └── config.yaml
└── pdf-mash/
    └── config.yaml
```

The environment variable `HOME_STILL_CONFIG_DIR` overrides the base path entirely. This is required for container deployments.

---

## 2. Cargo Workspace Standards

### 2.1 Virtual Manifest

Every repo uses a virtual workspace manifest (no `[package]` in the root `Cargo.toml`):

```toml
[workspace]
resolver = "2"
members = ["crates/*", "hs-style"]

[workspace.dependencies]
# Pin all shared versions here. Crates inherit with `{ workspace = true }`.
tokio        = { version = "1",   features = ["full"] }
clap         = { version = "4.5", features = ["derive", "env", "wrap_help"] }
serde        = { version = "1",   features = ["derive"] }
serde_json   = "1"
anyhow       = "1"
thiserror    = "2"
tracing      = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }
figment      = { version = "0.10", features = ["yaml", "env"] }
reqwest      = { version = "0.12", features = ["json", "rustls-tls"], default-features = false }
hs-style     = { path = "hs-style" }

[profile.release]
opt-level       = 3
lto             = "fat"
codegen-units   = 1
panic           = "abort"
strip           = "symbols"
```

### 2.2 Crate Cargo.toml

Each crate inherits from workspace deps:

```toml
[package]
name    = "paper-fetch-core"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio     = { workspace = true }
serde     = { workspace = true }
anyhow    = { workspace = true }
thiserror = { workspace = true }
```

### 2.3 Feature Flags

The `-core` crate has no features. The binary crate defines:

```toml
[features]
default = ["cli"]
cli     = ["hs-style/cli", "dep:indicatif", "dep:owo-colors"]
k8s     = ["hs-style/k8s", "dep:tonic-health"]
```

`cli` is the default for local development and standalone use.  
`k8s` enables headless JSON tracing output for Kubernetes deployments.

### 2.4 Allocator

All binaries include mimalloc to avoid musl malloc degradation:

```rust
// crates/<tool>/src/main.rs
#[global_allocator]
static GLOBAL: mimalloc::MiMalloc = mimalloc::MiMalloc;
```

---

## 3. CLI Command Structure

### 3.1 Naming Pattern

Home Still uses the **noun-verb pattern** exclusively. This matches `docker container create`, `gh pr list`, and is optimal for agent use because resource nouns group related operations.

```
<tool> <noun> <verb> [flags] [args]
```

Examples:
```
paper-fetch paper search "crispr"
paper-fetch paper download --doi 10.1234/...
paper-fetch paper list
librarian index watch ~/notes/
librarian index build
librarian search query "attention mechanism"
pdf-mash page extract --input doc.pdf
```

In the unified CLI, the tool name becomes the top-level noun:
```
hs paper search "crispr"
hs library index watch ~/notes/
hs library search query "attention mechanism"
hs pdf page extract --input doc.pdf
```

### 3.2 Standard CRUD Verbs

Use these verbs consistently. Never invent synonyms.

| Action | Verb |
|--------|------|
| Create a resource | `create` |
| Remove a resource | `delete` |
| List all resources | `list` |
| Show one resource | `show` |
| Modify a resource | `update` |
| Search/query | `search` |
| Download a file | `download` |
| Upload a file | `upload` |
| Start a watcher/daemon | `watch` |
| Run a one-shot job | `run` |
| Initialize config/data | `init` |

### 3.3 Universal Flags

Every binary, every subcommand must support these via `GlobalArgs` (from `hs-style`):

| Flag | Short | Values | Default | Description |
|------|-------|--------|---------|-------------|
| `--color` | | `auto\|always\|never` | `auto` | Color output control |
| `--output` | `-o` | `text\|json\|ndjson` | `text` | Output format |
| `--quiet` | `-q` | boolean | false | Suppress all non-result output |
| `--verbose` | `-v` | boolean | false | Show debug-level output |
| `--config-dir` | | path | `~/.home-still/` | Override config directory |
| `--dry-run` | `-n` | boolean | false | Show what would happen, no mutations |
| `--yes` | `-y` | boolean | false | Skip confirmation prompts |
| `--version` | | | | Print version and exit |
| `--help` | `-h` | | | Print help and exit |

All flags are declared `global = true` in clap so they work on any subcommand.

### 3.4 Flag Naming Rules

- Always `--kebab-case` for multi-word flags
- Always provide a `--long` form; short `-x` forms only for the universal flags above
- Boolean flags take no value: `--force`, not `--force=true`
- Negation uses `--no-` prefix: `--no-color`, `--no-cache`
- Value flags support both forms: `--output file.txt` and `--output=file.txt`

### 3.5 Positional Arguments

Limit to one positional argument per command. If a command needs more than one distinct input, use named flags. Multiple values of the same type are acceptable (`paper-fetch paper download doi1 doi2 doi3`).

Usage string conventions:
- Required: `<query>`
- Optional: `[query]`
- Repeatable: `<doi>...`

### 3.6 Help Text Structure

```
paper-fetch - Academic paper search and download

Usage:
  paper-fetch <noun> <verb> [flags]

Examples:
  $ paper-fetch paper search "transformer architecture"
  $ paper-fetch paper download --doi 10.48550/arXiv.1706.03762
  $ paper-fetch paper search "crispr" --output json | jq '.[] | .doi'

Commands:
  paper       Search and download academic papers

Flags:
  -h, --help             Show this help
      --version          Show version
  -o, --output <format>  Output format: text|json|ndjson [default: text]
  -q, --quiet            Suppress progress output
  -v, --verbose          Show debug output
      --color <when>     Color: auto|always|never [default: auto]
      --config-dir       Override config directory

Use "paper-fetch <noun> --help" for subcommand help.
```

Rules:
- One-line description after the tool name on line 1
- Examples section comes before Commands — agents and humans both read examples first
- Show the most common usage, not alphabetical order
- Print to stdout; exit 0

---

## 4. Output Standards

### 4.1 The stdout/stderr Contract

This is non-negotiable and enforced by CI:

- **stdout**: result data only — one item per line (text mode) or one JSON object per line (json/ndjson mode)
- **stderr**: everything else — progress bars, spinners, status messages, warnings, errors, timing summaries

This enables: `paper-fetch paper search "crispr" | jq .doi`  
Without progress artifacts corrupting the pipe.

### 4.2 Output Modes

Detected at startup in `main()`, never changed mid-run:

| Mode | When | Behavior |
|------|------|----------|
| `Rich` | stderr is TTY, no `NO_COLOR` | Colors + progress bars + spinners |
| `Plain` | TTY but `NO_COLOR` or `TERM=dumb` or `--color never` | Progress bars, no color |
| `Pipe` | stderr is not a TTY | Plain line output, no progress |
| `Json` | `--output json` or `--output ndjson` | NDJSON events on stdout, no progress on stderr |
| `Headless` | `KUBERNETES_SERVICE_HOST` or `CI` env set | Structured tracing JSON, no progress |

Detection order (precedence high to low):
1. `--output json` / `--output ndjson` → `Json`
2. `KUBERNETES_SERVICE_HOST` or `CI` set → `Headless`
3. `--color never` → `Plain`
4. `--color always` → `Rich`
5. stderr not a TTY → `Pipe`
6. `NO_COLOR` env set (non-empty) → `Plain`
7. `TERM=dumb` → `Plain`
8. Default → `Rich`

### 4.3 The Reporter Trait

All output in binary crates flows through `Reporter`. Business logic in `-core` crates never touches `Reporter`, `indicatif`, or `owo-colors` directly.

```rust
// From hs-style — do not reimplement
pub trait Reporter: Send + Sync {
    fn status(&self, verb: &str, message: &str);       // "Fetching" "arxiv..."
    fn warn(&self, message: &str);
    fn error(&self, message: &str);
    fn begin_stage(&self, name: &str, total: Option<u64>) -> Box<dyn StageHandle>;
    fn finish(&self, summary: &str);                   // "Found 1,204 papers in 3.2s"
}

pub trait StageHandle: Send {
    fn set_message(&self, msg: &str);
    fn set_length(&self, total: u64);                  // Transition spinner → bar
    fn inc(&self, delta: u64);
    fn finish_with_message(&self, msg: &str);
    fn finish_and_clear(&self);
}
```

Construct the correct `Reporter` impl once in `main()`, pass as `Arc<dyn Reporter>` to all command functions. The four impls from `hs-style`:

| Impl | Used when |
|------|-----------|
| `TtyReporter` | `Rich` or `Plain` mode |
| `PipeReporter` | `Pipe` mode |
| `JsonReporter` | `Json` mode |
| `SilentReporter` | `--quiet` flag |

### 4.4 Progress Bar Rules

- Start as spinner when total is unknown; transition to bounded bar via `set_length()` when total arrives
- `enable_steady_tick(Duration::from_millis(120))` for network-bound spinners
- Throttle: delay first draw 500ms, then max 100ms between redraws
- Always `finish_and_clear()` on success, then print a one-line summary via `reporter.finish()`
- Always `pb.reset_eta()` after any pause (rate limit, retry, backoff)
- Never render progress in `Pipe`, `Json`, or `Headless` mode

### 4.5 Color Palette

Use the `Styles` struct from `hs-style`. Do not hardcode ANSI colors in tool code.

| Role | Color | Usage |
|------|-------|-------|
| `verb` / action word | green bold | "Fetching", "Indexing", "Done" |
| `title` | bold | Paper titles, document names |
| `doi` / identifier | cyan underline | DOIs, IDs, hashes |
| `url` | cyan | HTTP URLs |
| `date` | dimmed | Publication dates, timestamps |
| `label` | white | Field labels ("Authors:", "Source:") |
| `warning` | yellow | Warning prefix |
| `error` | red bold | Error prefix |
| `success` | green | Completion messages |
| `source` | varies per source | Provider names (OpenAlex=blue, Crossref=green, etc.) |

`Styles::plain()` returns `Style::default()` for all fields — zero overhead, no ANSI output.

### 4.6 JSON Output Format

When `--output json`, emit one JSON object per line (NDJSON). Each event has a `type` discriminator:

```json
{"type":"stage_start","stage":"search","source":"arxiv","query":"crispr"}
{"type":"stage_progress","stage":"search","source":"arxiv","found":42,"total":1200}
{"type":"result","title":"CRISPR-Cas9...","doi":"10.1234/...","source":"arxiv"}
{"type":"stage_complete","stage":"search","source":"arxiv","total":1200,"elapsed_ms":1840}
{"type":"error","stage":"search","source":"crossref","message":"rate limited"}
```

Always flush stdout after each line. Never buffer NDJSON.

### 4.7 Text Table Output

Tables use minimal decoration — aligned columns, no borders:

```
 SOURCE           TITLE                                    YEAR   DOI
 arxiv            Attention Is All You Need                2017   10.48550/arXiv.1706.03762
 crossref         BERT: Pre-training of Deep...            2018   10.18653/v1/N19-1423
```

Rules:
- Left-align text, right-align numbers
- Truncate to terminal width (respect `COLUMNS` env or default 80)
- Support `--no-headers` for scripting
- Support `--no-truncate` for full output

---

## 5. Error Handling

### 5.1 Error Architecture

Use `thiserror` in `-core` crates for typed errors. Use `anyhow` in binary crates for rich context:

```rust
// In -core: typed, matchable
#[derive(Debug, thiserror::Error)]
pub enum PaperFetchError {
    #[error("provider {provider} rate limited, retry after {retry_after}s")]
    RateLimited { provider: String, retry_after: u64 },
    #[error("provider {provider} returned permanent error: {message}")]
    Permanent { provider: String, message: String },
    #[error("network error: {0}")]
    Network(#[from] reqwest::Error),
}

// In binary: anyhow for context chain
fn run() -> anyhow::Result<()> {
    fetch_paper(&doi).with_context(|| format!("Failed to fetch DOI {doi}"))?;
    Ok(())
}
```

### 5.2 Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | General error (not found, parse failure) |
| 2 | Config or usage error (bad flag, missing required arg) |
| 3 | Network or provider error (retriable) |
| 4 | Partial success (some providers failed, results returned) |

Document exit codes in `--help`. Treat them as a public API — changing them is a breaking change.

### 5.3 Error Message Format

Three parts: what, why, how to fix.

```
Error: cannot write to /tmp/papers/
  caused by: permission denied (os error 13)
  fix: run `chmod 755 /tmp/papers/` or set --config-dir to a writable path
```

Never expose raw OS errors or library panics to users. Wrap and rewrite. Suggestions go to stderr, exit non-zero.

---

## 6. Configuration

### 6.1 Precedence (highest to lowest)

1. CLI flags
2. `HOME_STILL_*` environment variables
3. `~/.home-still/<tool>/config.yaml`
4. `~/.home-still/config.yaml`
5. `/etc/home-still/config.yaml` (Kubernetes ConfigMap)
6. Compiled defaults

Implement with `figment`:

```rust
fn load_config(cli: &Cli) -> Result<AppConfig, figment::Error> {
    Figment::new()
        .merge(Serialized::defaults(AppConfig::default()))
        .merge(Yaml::file("/etc/home-still/config.yaml"))
        .merge(Yaml::file(config_dir.join("config.yaml")))
        .merge(Yaml::file(config_dir.join(tool_name).join("config.yaml")))
        .merge(Env::prefixed("HOME_STILL_").split("_"))
        .merge(Serialized::defaults(CliOverrides::from(cli)))
        .extract()
}
```

### 6.2 Environment Variables

All `HOME_STILL_*` vars map to config fields. Standard vars every tool must respect:

| Variable | Maps to |
|----------|---------|
| `HOME_STILL_CONFIG_DIR` | Config directory override |
| `HOME_STILL_COLOR` | `--color` flag |
| `HOME_STILL_OUTPUT` | `--output` flag |
| `HOME_STILL_QUIET` | `--quiet` flag |
| `HOME_STILL_VERBOSE` | `--verbose` flag |
| `NO_COLOR` | Disable all color (org-wide standard) |

### 6.3 Config File Format

YAML only. Provide a documented `default.yaml` in every repo under `config/`:

```yaml
# ~/.home-still/paper-fetch/config.yaml
download_dir: ~/papers
cache_dir: ~/.cache/home-still/paper-fetch
resilience:
  rate_limit_per_second: 1
  max_retries: 3
  circuit_breaker_threshold: 5
providers:
  arxiv:
    timeout_secs: 30
    base_url: "http://export.arxiv.org/api/query"
```

---

## 7. Logging and Observability

### 7.1 Tracing Setup

All tools initialize `tracing` in `main()`. Mode determines the subscriber:

```rust
match output_mode {
    OutputMode::Rich | OutputMode::Plain | OutputMode::Pipe => {
        tracing_subscriber::fmt()
            .with_env_filter(EnvFilter::from_default_env())
            .with_target(false)
            .init();
    }
    OutputMode::Headless => {
        tracing_subscriber::fmt()
            .json()
            .flatten_event(true)
            .init();
    }
}
```

`RUST_LOG` controls verbosity: `RUST_LOG=paper_fetch=debug`.

### 7.2 Structured Log Events (Headless/K8s)

In `Headless` mode, emit structured events that Loki/ELK/Datadog can parse:

```json
{"timestamp":"2026-03-08T12:00:00Z","level":"INFO","stage":"search","source":"arxiv","found":42,"total":1200}
```

Use `tracing` spans to carry context automatically — don't manually thread fields through every function.

---

## 8. The `hs` Unified Binary

### 8.1 Command Mapping

The unified `hs` binary reuses commands from each tool's binary crate directly. It does not shell out to subprocesses.

```
hs paper <verb>     →   paper-fetch paper <verb>
hs library <verb>   →   librarian library <verb>
hs pdf <verb>       →   pdf-mash pdf <verb>
hs embed <verb>     →   hs-embed embed <verb>
```

### 8.2 Cargo Dependency

`hs` references tool binary crates as git deps:

```toml
[dependencies]
paper-fetch    = { git = "https://github.com/home-still/paper-fetch", features = ["lib"] }
librarian      = { git = "https://github.com/home-still/librarian",   features = ["lib"] }
pdf-mash       = { git = "https://github.com/home-still/pdf-mash",    features = ["lib"] }
hs-style       = { path = "hs-style", features = ["cli"] }
```

Each tool must expose a `lib` feature that exports its `Commands` enum and `run()` function.

### 8.3 Structure

```rust
// hs/src/main.rs
#[derive(Parser)]
#[command(name = "hs", about = "Home Still — knowledge acquisition toolkit")]
struct Cli {
    #[command(flatten)]
    global: hs_style::GlobalArgs,
    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    /// Search and download academic papers
    #[command(subcommand)]
    Paper(paper_fetch::Commands),
    /// Index and search your document library
    #[command(subcommand)]
    Library(librarian::Commands),
    /// PDF segmentation and extraction
    #[command(subcommand)]
    Pdf(pdf_mash::Commands),
}
```

---

## 9. CI Conformance Checks

### 9.1 Required CI Jobs

Every repo's `ci.yml` must include:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive          # Required — fetches hs-style
      - run: cargo build --release

  test:
    runs-on: ubuntu-latest
    steps:
      - run: cargo test --all

  cli-conformance:
    runs-on: ubuntu-latest
    steps:
      - run: cargo build --release
      - name: --help exits 0
        run: ./target/release/${{ env.BINARY }} --help
      - name: --version exits 0
        run: ./target/release/${{ env.BINARY }} --version
      - name: NO_COLOR respected
        run: |
          output=$(NO_COLOR=1 ./target/release/${{ env.BINARY }} --help)
          echo "$output" | grep -qvP '\x1b\[' || (echo "ANSI found with NO_COLOR set" && exit 1)
      - name: pipe produces no ANSI
        run: |
          output=$(./target/release/${{ env.BINARY }} --help | cat)
          echo "$output" | grep -qvP '\x1b\[' || (echo "ANSI leaked into pipe" && exit 1)
      - name: --output json produces valid NDJSON
        run: |
          ./target/release/${{ env.BINARY }} <noun> list --output json \
            | head -5 | jq . > /dev/null
```

### 9.2 Reusable Workflow

Call the shared conformance workflow from `home-still/.github`:

```yaml
# In each repo's .github/workflows/ci.yml
jobs:
  conformance:
    uses: home-still/.github/.github/workflows/hs-conformance.yml@main
    with:
      binary: paper-fetch
```

---

## 10. Git Conventions

### 10.1 Branch Names

```
main          — production, always releasable
feat/<noun>   — new feature
fix/<noun>    — bug fix
chore/<noun>  — dependency updates, CI, tooling
docs/<noun>   — documentation only
```

### 10.2 Commit Messages

Conventional Commits format:

```
<type>(<scope>): <short description>

feat(paper): add semantic scholar provider
fix(output): respect NO_COLOR in pipe mode
chore(deps): update indicatif to 0.18
docs(readme): add installation instructions
```

Types: `feat`, `fix`, `chore`, `docs`, `test`, `refactor`, `perf`

### 10.3 Versioning

Semantic versioning. `*-core` crates and binary crates version together (same version in the same release).

Breaking changes in public API or CLI flags require a major version bump. Changes to `hs-style`'s `Reporter` trait are breaking for all consumers — version carefully.

### 10.4 Updating the `hs-style` Submodule

```bash
cd hs-style
git pull origin main
cd ..
git add hs-style
git commit -m "chore(deps): update hs-style to <commit-sha>"
```

Do not update `hs-style` in the same commit as other changes.

---

## 11. Checklist for New Tools

When creating a new Home Still tool, verify each item before opening a PR:

**Repo structure**
- [ ] `hs-style` added as git submodule
- [ ] Tool split into `-core` and binary crate
- [ ] `config/default.yaml` present and documented
- [ ] `CHANGELOG.md` present

**CLI compliance**
- [ ] Noun-verb subcommand pattern
- [ ] `GlobalArgs` from `hs-style` flattened into root `Cli`
- [ ] `--color auto|always|never` works
- [ ] `--output text|json|ndjson` works
- [ ] `--quiet` suppresses all non-result output
- [ ] `NO_COLOR=1` produces no ANSI codes
- [ ] Piped output produces no ANSI codes and no progress artifacts
- [ ] `--help` exits 0, prints to stdout
- [ ] `--version` exits 0, prints semver

**Output compliance**
- [ ] Results go to stdout
- [ ] Progress, warnings, errors go to stderr
- [ ] `--output json` produces valid NDJSON (one object per line)
- [ ] Exit codes match the standard table

**CI**
- [ ] `ci.yml` includes conformance job
- [ ] `submodules: recursive` in checkout step
- [ ] All conformance checks pass

**`hs` integration**
- [ ] Binary crate exposes `lib` feature with public `Commands` and `run()`
- [ ] `hs` repo updated with new subcommand mapping
