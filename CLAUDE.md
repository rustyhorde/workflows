# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This repo contains **only reusable GitHub Actions workflows** for Rust projects (consumed via `workflow_call`). There is no Rust crate, `Cargo.toml`, or application source here — every file under `.github/workflows/` is meant to be called by *other* repositories (the `rustyhorde` org and related projects). Changes here affect the CI of every downstream consumer, so treat edits as changes to a published API.

There is no local build/test/lint for this repo itself. Validation happens by calling these workflows from a consuming repo, or by linting YAML. Conventional Commits are used (`feat(ci):`, `fix(coverage):`, `chore:`).

## The workflows

All workflows are `on: workflow_call` and expose inputs (not all share the same set):

- **`rustfmt.yml`** — `cargo fmt --all -- --check`. No inputs.
- **`clippy-all-features.yml`** — clippy with `-Dwarnings` across the feature matrix.
- **`test-all-features.yml`** — runs the test matrix.
- **`build-tier2.yml`** — builds Tier-2 targets using `cross` + nightly (always nightly, unlike the others which take a `channel`).
- **`coverage.yml`** — `cargo llvm-cov` coverage, optional LCOV/HTML output, uploads to Codecov.

Common inputs (where applicable): `os`, `channel` (stable/beta/nightly/version), `target`, `num_chunks`/`chunk` (cargo-matrix sharding), `project` (for workspace member selection), `update` (run `cargo update` first).

## Key architectural conventions

These patterns repeat across nearly every workflow — preserve them when editing, and replicate them when adding a new workflow:

1. **`cargo-matrix` drives feature-combination testing.** Builds/tests/clippy run via `cargo matrix ... <subcommand>` rather than plain `cargo`. The `--num-chunks N --chunk M` flags shard the feature matrix across parallel jobs. When `project` is empty the command runs workspace-wide; when set it adds `-p <project>`. This is why most workflows have paired steps (e.g. "🧪 Test" / "🧪 Test P") gated on `inputs.project == ''` vs `!= ''`.

2. **`cargo-matrix` requires nightly to install.** Even on stable channels it is installed under a temporary nightly override:
   ```
   rustup toolchain install nightly
   rustup override set nightly
   cargo binstall --no-confirm --no-symlinks cargo-matrix --force
   rustup override remove
   ```
   The same nightly-override dance applies to `cargo-llvm-cov` and `cargo-nextest` in `coverage.yml`.

3. **Pre-installed binary cleanup before toolchain setup.** Runner images ship `rustfmt`/`cargo-fmt`/`rust-analyzer` in `~/.cargo/bin` that conflict with freshly installed toolchains. Each non-trivial workflow removes them first — with an OS-conditional step (bash `rm -rf` for `ubuntu-latest`, PowerShell `Remove-Item` for `windows-latest`).

4. **All `cargo binstall` steps set guardrails:** `GITHUB_TOKEN: ${{ github.token }}` and `BINSTALL_MAXIMUM_RESOLUTION_TIMEOUT: "20"`. Keep these on any new binstall step.

5. **All third-party actions are SHA-pinned** with the version in a trailing comment (e.g. `actions/checkout@de0fac...  # v6.0.2`). When bumping an action, update both the SHA and the comment.

6. **Rust cache is skipped for forked PRs:** every `Swatinem/rust-cache` step is gated on `github.event_name != 'pull_request' || ! github.event.pull_request.head.repo.fork`.

7. **Database secrets are wired through for integration tests.** `clippy-all-features`, `test-all-features`, and `coverage` expose ArangoDB / MongoDB credentials (`ARANGODB_*`, `BB8_MONGODB_*`) and `CODECOV_TOKEN` via `secrets.*`. These are inherited from the calling repo (`secrets: inherit`), so downstream callers must define them.

8. **Step names use emoji prefixes** (✅ Checkout, 🧰 Toolchain, 💾 Install, 🧪 Test, etc.). Match this style for consistency.
