# rustyhorde/workflows

Reusable GitHub Actions workflows for Rust projects in the `rustyhorde` organization.

## Workflows

| Workflow | File | Purpose |
|---|---|---|
| Formatting | `rustfmt.yml` | `cargo fmt --all -- --check` |
| Lints | `clippy-all-features.yml` | clippy across full feature matrix |
| Test | `test-all-features.yml` | tests across full feature matrix |
| Build (Tier 2) | `build-tier2.yml` | cross-compile to Tier-2 targets |
| Coverage | `coverage.yml` | `cargo llvm-cov` + optional Codecov upload |

All workflows use [`cargo-matrix`](https://github.com/rustyhorde/cargo-matrix) to exhaustively test feature combinations.

---

## Usage

Call workflows from another repository using `workflow_call`. Reference a specific commit SHA or tag to pin the version.

### Formatting

```yaml
jobs:
  rustfmt:
    uses: rustyhorde/workflows/.github/workflows/rustfmt.yml@main
    secrets: inherit
```

### Lints

```yaml
jobs:
  clippy:
    uses: rustyhorde/workflows/.github/workflows/clippy-all-features.yml@main
    with:
      os: ubuntu-latest
      channel: nightly
      target: x86_64-unknown-linux-gnu
    secrets: inherit
```

### Test

```yaml
jobs:
  test:
    uses: rustyhorde/workflows/.github/workflows/test-all-features.yml@main
    with:
      os: ubuntu-latest
      channel: stable
      target: x86_64-unknown-linux-gnu
    secrets: inherit
```

### Build (Tier 2 targets)

```yaml
jobs:
  build-armv7:
    uses: rustyhorde/workflows/.github/workflows/build-tier2.yml@main
    with:
      target: armv7-unknown-linux-gnueabihf
    secrets: inherit
```

### Coverage

```yaml
jobs:
  coverage:
    uses: rustyhorde/workflows/.github/workflows/coverage.yml@main
    with:
      os: ubuntu-latest
      channel: nightly
      target: x86_64-unknown-linux-gnu
      lcov: true
    secrets: inherit
```

### Matrix sharding example

Use `num_chunks`/`chunk` to split a large feature matrix across parallel jobs:

```yaml
jobs:
  test-1:
    uses: rustyhorde/workflows/.github/workflows/test-all-features.yml@main
    with:
      os: ubuntu-latest
      channel: stable
      target: x86_64-unknown-linux-gnu
      num_chunks: 3
      chunk: 1
    secrets: inherit

  test-2:
    uses: rustyhorde/workflows/.github/workflows/test-all-features.yml@main
    with:
      os: ubuntu-latest
      channel: stable
      target: x86_64-unknown-linux-gnu
      num_chunks: 3
      chunk: 2
    secrets: inherit

  test-3:
    uses: rustyhorde/workflows/.github/workflows/test-all-features.yml@main
    with:
      os: ubuntu-latest
      channel: stable
      target: x86_64-unknown-linux-gnu
      num_chunks: 3
      chunk: 3
    secrets: inherit
```

### Workspace member example

Use `project` to run against a single crate in a Cargo workspace:

```yaml
jobs:
  test:
    uses: rustyhorde/workflows/.github/workflows/test-all-features.yml@main
    with:
      os: ubuntu-latest
      channel: stable
      target: x86_64-unknown-linux-gnu
      project: my-crate
    secrets: inherit
```

---

## Inputs reference

### `rustfmt.yml`

No inputs.

---

### `clippy-all-features.yml`

| Input | Required | Default | Description |
|---|---|---|---|
| `os` | yes | — | Runner OS: `ubuntu-latest`, `macos-latest`, or `windows-latest` |
| `channel` | yes | — | Rust channel: `stable`, `beta`, `nightly`, or a version like `1.80.0` |
| `target` | yes | — | Rust target triple, e.g. `x86_64-unknown-linux-gnu` |
| `num_chunks` | no | `1` | Total number of cargo-matrix chunks (for sharding) |
| `chunk` | no | `1` | Which chunk to run (1-based) |
| `project` | no | `""` | Workspace member to pass as `-p <project>`; empty means entire workspace |
| `update` | no | `false` | Run `cargo update` before clippy |

---

### `test-all-features.yml`

| Input | Required | Default | Description |
|---|---|---|---|
| `os` | yes | — | Runner OS: `ubuntu-latest`, `macos-latest`, or `windows-latest` |
| `channel` | yes | — | Rust channel: `stable`, `beta`, `nightly`, or a version like `1.80.0` |
| `target` | yes | — | Rust target triple, e.g. `x86_64-unknown-linux-gnu` |
| `num_chunks` | no | `1` | Total number of cargo-matrix chunks (for sharding) |
| `chunk` | no | `1` | Which chunk to run (1-based) |
| `project` | no | `""` | Workspace member to pass as `-p <project>`; empty means entire workspace |
| `update` | no | `false` | Run `cargo update` before tests |
| `nextest` | no | `false` | Use `cargo-nextest` (`nextest run`) instead of `cargo test` |

---

### `build-tier2.yml`

Always runs on `ubuntu-latest` with the `nightly` toolchain using `cross` for cross-compilation.

| Input | Required | Default | Description |
|---|---|---|---|
| `target` | yes | — | Rust target triple to cross-compile for, e.g. `armv7-unknown-linux-gnueabihf` |
| `num_chunks` | no | `1` | Total number of cargo-matrix chunks (for sharding) |
| `chunk` | no | `1` | Which chunk to run (1-based) |
| `project` | no | `""` | Workspace member to pass as `-p <project>`; empty means entire workspace |

---

### `coverage.yml`

| Input | Required | Default | Description |
|---|---|---|---|
| `os` | yes | — | Runner OS: `ubuntu-latest`, `macos-latest`, or `windows-latest` |
| `channel` | yes | — | Rust channel: `stable`, `beta`, `nightly`, or a version like `1.80.0` |
| `target` | yes | — | Rust target triple, e.g. `x86_64-unknown-linux-gnu` |
| `clean` | no | `false` | Run `cargo llvm-cov clean --workspace` before collecting coverage |
| `lcov` | no | `false` | Generate `lcov.info` and upload to Codecov |
| `html` | no | `false` | Generate an HTML coverage report |
| `run_cmd` | no | `cargo llvm-cov --no-report` | Full command used to collect coverage; override to add flags like `--all-features` or nextest integration |

---

## Secrets reference

Pass secrets to calling workflows with `secrets: inherit`. The following secrets are read by the workflows:

| Secret | Used by | Description |
|---|---|---|
| `CODECOV_TOKEN` | `coverage` | Codecov upload token (only required when `lcov: true`) |
