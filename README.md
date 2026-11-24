# Shared CI/CD (`trykA123/ci`)

Reusable GitHub Actions workflows and composite actions for Windows-first Rust desktop apps (`navo`, `sidewise`, `lazysheet`, and future `vane`/`braid`).

Callers pin every workflow and action to a full commit SHA with a `# vX.Y.Z` tag comment (Dependabot keeps SHAs updated).

## Contents

| Path | Type | What it proves |
| --- | --- | --- |
| `.github/workflows/rust-windows-check.yml` | Reusable workflow | Parallel Windows `clippy -D warnings`, build-once `cargo nextest archive`, `N` hash-partitioned Windows test shards, optional targeted `-p` mode on non-`main` branches, optional release preview artifact |
| `.github/workflows/rust-linux-gate.yml` | Reusable workflow | Linux `fmt`, `clippy`, `taplo` (if `taplo.toml`), `typos` (if `typos.toml`), `cargo deny` (if `deny.toml`), `cargo audit`, `nextest`, doc-tests — or `cargo xtask gate [--coverage]` |
| `.github/workflows/rust-windows-release.yml` | Reusable workflow | `main`-only & semver tag guard, green-CI check (or `full-checks`), `cargo auditable` release build, `cargo audit bin`, PDB zip + `SHA256SUMS.txt`, `actions/attest-build-provenance`, environment-gated publish, release retention (default: keep 5) |
| `actions/setup-rust/action.yml` | Composite action | Resolves toolchain (input or `rust-toolchain.toml`), exports `CARGO_INCREMENTAL=0`, optionally configures bundled `rust-lld` as `lld-link`, restores/saves shared `Swatinem/rust-cache`, installs prebuilt tools via `taiki-e/install-action` |

## Cache & Sharding Design

- **Warm caches under the 10 GB repo limit:** All workflows use `Swatinem/rust-cache` with explicit `shared-key` pools (`win-clippy`, `win-test`, `win-release`, `linux-gate`) and `save-if: ${{ github.ref == 'refs/heads/main' }}`. Feature and `ci/*` branches restore `main`'s warm caches without writing duplicate entries. Do not stack `sccache` (GHA backend) with `rust-cache`: `sccache` creates thousands of small entries and triggers 10 GB LRU eviction.
- **Cheaper compiles:** `CARGO_INCREMENTAL=0`, bundled `rust-lld` invoked as `lld-link` on Windows debug/test builds (preserving `.cargo/config.toml` `rustflags`), and `[profile.dev.package."*"] debug = 0` + `[profile.test.package."*"] debug = 0` in each app's `Cargo.toml` (keeping workspace crates at `debug = "line-tables-only"` for crash backtraces).
- **Build-once, shard-many:** `build` compiles test binaries once via `cargo nextest archive` and uploads `.tar.zst` with `compression-level: 0` and `retention-days: 1`. Each `test` shard skips Rust/cache setup, extracts with `--extract-to . --extract-overwrite --workspace-remap .` (preserving compile-time `CARGO_BIN_EXE_*` and `CARGO_MANIFEST_DIR` paths), and runs `--partition hash:k/N`. Any test that invokes `rustc` or reads `.rlib`s runs in `build` via `build-host-tests-expr` and is excluded on shards.
- **Targeted atom runs:** `packages: "crate-a crate-b"` runs clippy and nextest inline on 2 runners (no archive transfer, no shard fan-out). Rejected on `refs/heads/main` so `main` always proves the full workspace.

## Measured vs Expected Wall Times (Navo, 15,567 tests)

| Run type | Before (measured) | Expected after |
| --- | --- | --- |
| Full `windows-quick` (`N = 4`) | 34.5–52.8 min (clippy 2–3m, compile ~14m, test ~17–34m) | ~16–25 min (clippy hidden; compile ~10–12m; 4 shards ~6–11m) |
| Targeted `windows-quick` (`packages: "crate-a"`) | 34.5–52.8 min | ~4–8 min |
| Small app (e.g. Lazysheet, 3 crates, `shards: 1`) | Serial clippy + test + release | Parallel clippy + test + preview build (~50% shorter) |

## Adopting in an App (15–30 lines)

Replace `<SHA>` with the full 40-char commit SHA of `trykA123/ci`.

### Navo (`windows-quick.yml`)

```yaml
name: windows-quick
on:
  push: { branches: [main] }
  pull_request:
  workflow_dispatch:
    inputs:
      packages: { type: string, default: "" }
      shards: { type: number, default: 4 }
permissions: { contents: read }
concurrency: { group: "quick-${{ github.ref }}", cancel-in-progress: true }
jobs:
  check:
    uses: trykA123/ci/.github/workflows/rust-windows-check.yml@<SHA> # v1.0.0
    with:
      packages: ${{ inputs.packages || '' }}
      shards: ${{ inputs.shards || 4 }}
      pre-archive-cmd: "./.github/scripts/fetch-pdfium.ps1 -Destination target/debug"
      build-host-tests-expr: "package(xtask) | (package(navo-bench) & test(privacy::api_tests))"
      extra-env-json: '{"NAVO_EXPECT_PDFIUM":"1","PDFIUM_TAG":"chromium/8125","PDFIUM_SHA256":"9a964261760cff52570aabf038ab346d3a5034dd8b30d0d5fe8861235d356b8b"}'
```

### Sidewise (`windows-quick.yml` + `release.yml`)

- `windows-quick.yml`: call `rust-windows-check.yml@<SHA>` with `shards: 2` (scale to `4` as tests grow) and `rust-linux-gate.yml@<SHA>` with `linux-deps: true`.
- `release.yml`: call `rust-windows-release.yml@<SHA>` with `package: sidewise-app`, `binary-name: sidewise.exe`, `asset-prefix: sidewise`, `require-green-workflow: windows-quick.yml`, `keep-releases: 5`.

### Lazysheet (`check.yml` + `release.yml`)

- `check.yml`: call `rust-windows-check.yml@<SHA>` with `shards: 1`, `dev-debug: line-tables-only` (until added to `Cargo.toml`), `build-release-pkg: lazysheet-app`, `release-artifact-path: target/release/lazysheet-app.exe`, `artifact-retention-days: 14`.
- `release.yml`: call `rust-windows-release.yml@<SHA>` with `package: lazysheet-app`, `binary-name: lazysheet-app.exe`, `asset-prefix: lazysheet`, `release-title-prefix: Lazysheet`, `notes-dir: release-notes`, `prerelease: false`, `keep-releases: 5`.

## Supply-Chain Defaults

- All third-party actions pinned to full 40-char commit SHAs.
- Least-privilege `permissions:` per job (`contents: read` by default; `id-token: write` + `attestations: write` only on release build; `contents: write` only on `publish`).
- Automatic `cargo deny check` whenever `deny.toml` is present, plus `cargo auditable` + `cargo audit bin` on release executables.

## License

Licensed under either of [Apache License, Version 2.0](LICENSE-APACHE) or [MIT license](LICENSE-MIT) at your option.
