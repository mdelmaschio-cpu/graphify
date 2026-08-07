# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

graphify (PyPI package `graphifyy`) is a Claude Code / multi-assistant skill backed by a standalone
Python library. The skill orchestrates the library; the library (`graphify/`) can also be used
directly from the CLI (`graphify <command>`) or imported. It turns a folder of code/docs/PDFs/images/
videos into a queryable knowledge graph (`graphify-out/graph.json`, `GRAPH_REPORT.md`, `graph.html`).

This project has a graphify knowledge graph at `graphify-out/`, if present:
- Before answering architecture or codebase questions, read `graphify-out/GRAPH_REPORT.md` for god
  nodes and community structure.
- If `graphify-out/wiki/index.md` exists, navigate it instead of reading raw files.
- After modifying code files in this session, run `graphify update .` to keep the graph current
  (AST-only, no API cost).

## Dev setup

```bash
uv sync --all-extras         # venv + graphify + all extras + dev group (pytest, ruff, pyright, ...)
uv run graphify --version
uv run python -c "import graphify; print(graphify.__file__)"
```

Active development happens on the **`v8`** branch (not `main`).

## Commands

```bash
# Tests
uv run pytest tests/ -q                    # full suite
uv run pytest tests/test_extract.py -q     # one module
uv run pytest tests/ -q -k "python"         # filter by name/keyword

# Lint / type-check
uv run ruff check --config pyproject.toml .
uv run pyright

# Security / dependency audit (also run in CI, non-blocking there)
uv run bandit -r graphify -ll
uv run pip-audit --strict

# Pre-commit (installs the ruff hook + the skillgen anti-drift hook)
uv run pre-commit install
uv run pre-commit run --all-files

# Skill artifact generation (see "Skill generation" below)
python -m tools.skillgen                  # regenerate every platform's skill artifacts
python -m tools.skillgen --platform claude
python -m tools.skillgen --check          # fails if generated files drifted from fragments
python -m tools.skillgen --bless          # rewrite tools/skillgen/expected/ after an intentional fragment edit

# Exercise the CLI itself
uv run graphify --help
uv run graphify install                  # end-to-end install smoke test (also run in CI)
```

macOS note: the test suite includes both `sample.f90` and `sample.F90` fixtures, which collide on
case-insensitive HFS+/APFS. Run on Linux/Docker if you need both Fortran variants.

Commit style: `fix: <description>` / `feat: <description>` / `docs: <description>`. Run the full
test suite before opening a PR.

## Architecture

### Pipeline

```
detect()  →  extract()  →  build_graph()  →  cluster()  →  analyze()  →  report()  →  export()
```

Each stage is a single function in its own module. They communicate through plain Python dicts and
NetworkX graphs — no shared state, no side effects outside `graphify-out/`.

| Module | Function | Input → Output |
|--------|----------|----------------|
| `detect.py` | `collect_files(root)` | directory → filtered `[Path]` list |
| `extract.py` | `extract(path)` | file path → `{nodes, edges}` dict |
| `build.py` | `build_graph(extractions)` | list of extraction dicts → `nx.Graph` |
| `cluster.py` | `cluster(G)` | graph → graph with `community` attr per node |
| `analyze.py` | `analyze(G)` | graph → analysis dict (god nodes, surprises, questions) |
| `report.py` | `render_report(G, analysis)` | graph + analysis → `GRAPH_REPORT.md` string |
| `export.py` | `export(G, out_dir, ...)` | graph → Obsidian vault, `graph.json`, `graph.html`, `graph.svg` |
| `callflow_html.py` | `write_callflow_html(...)` | `graphify-out/` files → Mermaid architecture/call-flow HTML |
| `ingest.py` | `ingest(url, ...)` | URL → file saved to corpus dir |
| `cache.py` | `check_semantic_cache` / `save_semantic_cache` | files → (cached, uncached) split |
| `security.py` | validation helpers | URL / path / label → validated or raises |
| `validate.py` | `validate_extraction(data)` | extraction dict → raises on schema errors |
| `serve.py` | `start_server(graph_path)` | graph file path → MCP stdio/HTTP server |
| `watch.py` | `watch(root, flag_path)` | directory → writes flag file on change |
| `benchmark.py` | `run_benchmark(graph_path)` | graph file → corpus vs subgraph token comparison |

`extract.py` is the largest module by far (dispatches to `graphify/extractors/` for a handful of
languages — C#, Razor/Blade templates, Elixir, Zig — plus inline tree-sitter extraction for most of
the 36+ supported languages). `__main__.py` is the CLI entry point: it is a plain `sys.argv`-based
dispatcher (`if cmd == "install": ... elif cmd == "uninstall": ...`), not argparse — new subcommands
are added as another `elif cmd == "..."` branch, with usage text hand-maintained in the `--help`
block near the top of `main()`. Besides the pipeline stages above, `__main__.py` also dispatches
platform-install subcommands (`install`/`uninstall`/`claude`/`codex`/`cursor`/`copilot`/`amp`/`kilo`/
`kiro`/`pi`/`devin`/`vscode`/`antigravity`/`codebuddy`/`gemini`/`falkordb`/... — one per supported
assistant/backend) and the graph-tooling subcommands backed by the extension modules below
(`affected`, `prs`, `reflect`, `diagnose`, `global`, `wiki`, `tree`, `html`/`callflow-html`, `svg`,
`graphml`, `merge-*`, `label`, `query`, `cache-check`, `hook`/`hook-check`).

### Extension modules

Beyond the core pipeline, `graphify/` has grown a number of supporting modules, grouped by concern:

**Ingestion / introspection** (feed non-code sources into extraction):
`manifest_ingest.py` (deterministic, non-LLM parsing of package manifests — `package.json`,
`requirements.txt`, etc.), `scip_ingest.py` (SCIP JSON ingestion), `mcp_ingest.py` (MCP server config
files), `cargo_introspect.py` (Cargo workspace-internal crate deps), `pg_introspect.py` (Postgres
schema introspection), `google_workspace.py` (Google Docs/Sheets shortcut files), `file_slice.py`
(intra-file slicing for oversized text documents so they don't blow the LLM context window).

**Graph resolution / maintenance**: `resolver_registry.py` (registry of per-language cross-file
resolution passes, invoked from `build.py`), `symbol_resolution.py` (generic deterministic symbol
indexing shared by resolvers), `ruby_resolution.py` (Ruby-specific member-call resolver registered
through `resolver_registry.py`), `dedup.py` (entity deduplication pipeline), `semantic_cleanup.py`
(validates/sanitizes LLM-produced semantic fragments before merge), `multigraph_compat.py` (runtime
capability probe for MultiDiGraph mode), `diagnostics.py` (read-only diagnostics for MultiDiGraph
readiness), `ids.py` (single source of truth for node-ID normalization), `paths.py` (single source of
truth for the `graphify-out/` directory name and path resolution), `manifest.py` (back-compat
re-export shim over `detect.py`'s manifest helpers), `_minhash.py` (MinHash + band-LSH, a
datasketch-compatible drop-in used by `dedup.py`).

**Query / analysis tooling**: `affected.py` (blast-radius query — given a changed symbol, finds
affected nodes; backs `graphify affected`), `querylog.py` (append-only, fail-silent JSONL query
logging), `reflect.py` (deterministic "work memory" reflection over `graphify-out/memory/`),
`global_graph.py` (registry for the cross-project "global" graph — `graphify global add/remove/list`),
`prs.py` (graph-aware PR dashboard), `wiki.py` (Markdown wiki export), `tree_html.py` (D3 v7
collapsible-tree HTML view).

**LLM / media**: `llm.py` (backend detection, cost estimation, parallel corpus extraction, community
labeling), `transcribe.py` (audio/video transcription via Whisper for `ingest.py`).

**Editor/agent integration**: `hooks.py` (install/uninstall/status for the SessionStart hook that
keeps the graph fresh during a coding session).

### Extraction output schema

Every extractor returns:

```json
{
  "nodes": [
    {"id": "unique_string", "label": "human name", "source_file": "path", "source_location": "L42"}
  ],
  "edges": [
    {"source": "id_a", "target": "id_b", "relation": "calls|imports|uses|...", "confidence": "EXTRACTED|INFERRED|AMBIGUOUS"}
  ]
}
```

`validate.py` enforces this schema before `build_graph()` consumes it.

### Confidence labels

| Label | Meaning |
|-------|---------|
| `EXTRACTED` | Explicitly stated in source (import statement, direct call) |
| `INFERRED` | Reasonable deduction (call-graph second pass, co-occurrence) |
| `AMBIGUOUS` | Uncertain; flagged for human review in `GRAPH_REPORT.md` |

### Adding a new language extractor

1. Add `extract_<lang>(path: Path) -> dict` in `extract.py` (tree-sitter parse → walk nodes → collect
   `nodes`/`edges` → call-graph second pass for INFERRED `calls` edges), or a new module under
   `graphify/extractors/` for template/hybrid languages.
2. Register the file suffix in `extract()`'s dispatch and `collect_files()`.
3. Add the suffix to `CODE_EXTENSIONS` in `detect.py` and `_WATCHED_EXTENSIONS` in `watch.py`.
4. Add the tree-sitter package to `pyproject.toml` dependencies (and to the relevant optional-extras
   group if it's not a universally-installed grammar).
5. Add a fixture file to `tests/fixtures/` and tests to `tests/test_languages.py`.

### Security

All external input passes through `graphify/security.py` before use:

- URLs → `validate_url()` (http/https only) + `_NoFileRedirectHandler` (blocks `file://` redirects)
  + `_SSRFGuardedHTTPConnection`/`_SSRFGuardedHTTPSConnection` (resolve-then-check the IP against a
  private/link-local/loopback blocklist before connecting, closing DNS-rebinding SSRF gaps)
- Fetched content → `safe_fetch()` / `safe_fetch_text()` (size cap, timeout)
- Graph file paths → `validate_graph_path()` (must resolve inside `graphify-out/`) and
  `check_graph_file_size_cap()` (rejects oversized graph files before parsing)
- Node labels → `sanitize_label()` (strips control chars, caps 256 chars, HTML-escapes); arbitrary
  metadata values → `sanitize_metadata()`

See `SECURITY.md` for the full threat model.

### Skill generation (`tools/skillgen/`)

The skill files that ship under `graphify/skill*.md`, `graphify/command-*.md`, and
`graphify/skills/<platform>/references/*.md` are **generated artifacts** — the source of truth is
the fragments under `tools/skillgen/fragments/` (`core`, `dispatch`, `extra`, `query-stub`,
`references`, `shell`, `always-on`) combined per platform via `tools/skillgen/platforms.toml`.

- Never hand-edit a generated file under `graphify/skill*.md` or `graphify/skills/*/references/*`.
  Edit the fragment, then regenerate.
- After any fragment edit: `python -m tools.skillgen` to regenerate, then `python -m tools.skillgen
  --bless` to update `tools/skillgen/expected/` baselines.
- CI (`skillgen-check` job) and the local pre-commit hook both run `python -m tools.skillgen --check`
  and fail the build on drift between fragments and committed generated files. CI additionally runs
  `--audit-coverage`, `--schema-singleton`, `--monolith-roundtrip`, and `--always-on-roundtrip`,
  which diff each platform's generated body against an immutable pre-split baseline commit fetched
  from git history (`fetch-depth: 0` in CI) — these need full git history to run for real.
- `graphify/always_on/*.md` are the always-on injection blocks (e.g. `CLAUDE.md`/`AGENTS.md`
  boilerplate this tool inserts into a host project) — also generated, same rules apply.

### Repo layout

- `graphify/` — the library + CLI (`graphify.__main__:main`) + MCP server entry (`graphify.serve:_main`).
- `graphify/extractors/` — per-language extractor modules for languages that don't fit the generic
  tree-sitter path in `extract.py` (C#, Razor, Blade, Elixir, Zig).
- `graphify/skills/<platform>/` — per-assistant generated skill references (`claude`, `codex`,
  `cursor` (as `agents`), `amp`, `kilo`, `kiro`, `pi`, `copilot`, `droid`, `claw`, `opencode`, `trae`,
  `vscode`, `windows`); `graphify/skill*.md` / `graphify/command-*.md` are the flat, non-directory
  skill artifacts for the remaining platforms. All are generated — see "Skill generation" above.
- `graphify/always_on/` — generated always-on injection blocks (`claude-md.md`, `agents-md.md`,
  `gemini-md.md`, `antigravity-rules.md`, `kiro-steering.md`, `vscode-instructions.md`) inserted into
  a host project's own config files by `graphify install`.
- `tools/skillgen/` — build-time generator for the skill/CLAUDE.md/AGENTS.md artifacts (see above).
- `tests/` — one test file per module (`test_<module>.py`), plus `tests/fixtures/` (per-language
  sample repos/files used by extractor tests).
- `worked/` — worked examples (real corpora run through graphify, with a `review.md` critiquing what
  the graph got right/wrong). This is the main contribution vector besides bug fixes.
- `docs/` — architecture notes (`how-it-works.md`, `node-summaries-rfc.md`), `docs/superpowers/`
  (Claude "superpower" plugin notes), and translated READMEs (`docs/translations/`).
- `ARCHITECTURE.md` — canonical architecture doc (kept in sync with this section).
- `BENCHMARKS.md` — retrieval-quality benchmark methodology and results (LOCOMO, LongMemEval-S, etc).

### Packaging notes

- Optional dependency extras (`pyproject.toml` `[project.optional-dependencies]`) gate heavy/native
  deps behind opt-in installs: `mcp`, `neo4j`, `falkordb`, `pdf`, `watch`, `svg`, `leiden`, `office`,
  `google`, `postgres`, `video`, `chinese` (jieba word segmentation), `sql` (tree-sitter-sql), and
  per-LLM-backend extras (`kimi`, `ollama`, `bedrock`, `anthropic`, `gemini`, `openai`). `all` pulls in
  everything except the niche/native-toolchain ones (`dm`, `terraform` stay separate since they
  require a C toolchain or ship non-portable wheels).
- `ruff` lint selection is intentionally narrow (`E9, F63, F7, F82` — syntax/undefined-name errors
  only); don't assume a broader ruleset is enforced.
- `uv.lock` is committed; CI runs with `--frozen` and must never cause the lock to churn.
