# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Open CAD Studio: Rust 2D drafting + 3D modeling app (DWG/DXF native read/write), GPL-3.0. One codebase builds a native desktop app (`iced` multi-window daemon, `wgpu`) and a browser app (wasm32, WebGL2, single window). Package/binary name is `OpenCADStudio` (note `#![allow(non_snake_case)]`). Versions are weekly: `YYYY.WW.patch`.

## Commands

```bash
cargo build --release --bin OpenCADStudio     # desktop
cargo run                                      # default-run = OpenCADStudio
cargo test --workspace --locked                # what CI runs (LC_ALL=C in CI: some tests compare English text)
cargo test --lib <path::or_name>               # single unit test / module, e.g. cargo test --lib i18n::tests
cargo test --test integration <name>           # integration tests (see below)
cargo check --locked --target wasm32-unknown-unknown -p OpenCADStudio -p ocs_web_worker   # web-check CI job; run after touching shared code
cargo fmt --all -- --check ; cargo clippy --workspace --all-targets --locked   # CI lint is informational only
cargo bench --bench <name>                     # ~30 criterion-style benches in benches/ (harness = false)
trunk serve                                    # web dev server (needs wasm32 target, trunk, wasm-bindgen-cli)
```

- Linux needs GL/X11/Wayland/fontconfig/freetype dev libs (list in README "Build from source").
- `tests/` uses `autotests = false`: every integration test file must be registered as a `#[path]` module in `tests/integration.rs` or it never runs. They compile into a single `integration` binary.
- GPU-backed tests are `#[ignore]`d (need an adapter). `.cargo/config.toml` sets `ICED_TEST_BACKEND=tiny-skia` because parallel wgpu devices segfault. Any test that builds a headless iced renderer must take `crate::headless_ui_lock()` first.
- `.cargo/config.toml` also sets `RUST_MIN_STACK=64MB` (rustc overflows on this crate's size otherwise) and large link stacks on Windows.
- Dev profile: workspace crates at `opt-level = 0`, dependencies at 2, `debug = 0`. If you add a workspace member, add it to `[profile.dev.package]` too ("keep in sync" comment in `Cargo.toml`).
- Headless/automation entry points on the binary: `--export in out [--target-version R14]`, `--serve [--port]`, `--mcp`, `--script`/`/b`, `--batch`/`--each` (see `src/cli.rs`, `src/main.rs`, `docs/automation/`).
- Translations: `locales/*/opencadstudio.ftl`, labels map through `src/locale_catalog.rs`. After edits run `python3 scripts/export-locales.py`, then `python3 scripts/test_site.py` and `cargo test --lib i18n::tests`. New user-visible strings need entries in all 23 locales.
- Release notes: add entries to `changelogs/unreleased.md` (and the Unreleased row in `CHANGELOG.md`).

## Architecture

### Workspace
Root crate `OpenCADStudio` (lib `src/lib.rs` + bin `src/main.rs`) plus `crates/`:
- `ocs_plugin_api` — semver-stable, dependency-light plugin contract (ribbon vocabulary `CadModule`/`ToolDef`, manifest/version gates; `host` feature adds IPC, shm snapshots, process runner). Read `crates/ocs_plugin_api/ARCHITECTURE.md` before touching it.
- `ocs_save` — shared SAVE serialization dispatch + web-worker wire format (used by inline native path and the worker).
- `ocs_web_worker` — wasm Web Worker (DWG/DXF parse, hashing, save/autosave); built by `scripts/build-web-worker.sh` as a Trunk post-build hook.
- `dwg-thumbnailer`, `dwg-thumbnailer-win` — OS file-manager thumbnails (host also exposes `--dwg-thumbnail`).

Core engine crates are **git dependencies pinned by rev** (not in this repo): `opencaddocument` (imported as `document` — drawing database, `EntityType`, `Handle`), `opencadkernel` (as `kernel` — B-rep solids, booleans, ACIS), `opencadgraph`, and a fork of `iced`. Bumping those revs changes behaviour repo-wide.

### Application shape (`src/`)
- `app/` — the iced Elm-style app. `app/mod.rs` holds the giant `Message` enum and the `OpenCADStudio` state; `app/update/*` handles messages, `app/view/*` builds UI, `app/commands/*` implements command-line commands, `app/command_driver/*` runs interactive commands (clipboard, transform, modify, solid3d…). Dialogs/panels (layers, blocks, block editor, properties, text styles…) are siblings in `app/`. Native opens extra OS windows for dialogs; web renders them as in-canvas modals — check both when adding a dialog (`docs/dialog-windows.md`, `docs/native-vs-web.md`).
- `command.rs` — `CadCommand` trait: interactive commands receive viewport events and return `CmdResult` tokens. Commands are dispatched by name through `dispatch_command` in `app/commands/mod.rs`; aliases come from `assets/ocad.pgp`. `docs/commands-reference.md` is a generated registry listing.
- `modules/` — core ribbon tabs. Each implements `CadModule` and is listed by hand in `modules/registry.rs::all_modules()`. Add-on plugin tabs use the same vocabulary from `ocs_plugin_api`.
- `scene/` — rendering/model side: `Scene`, entity → render conversion (`scene/convert`, entry `tess.rs` → `tessellate.rs`), picking (`scene/pick`), cached kernel bodies, block-editor authoring state, and the `wgpu` pipelines (`scene/pipeline/*` with WGSL in `src/shaders/*.wgsl`). `docs/tessellation.md` explains the three geometry paths (kernel B-rep mesh / curve-sampled via `entities::curve::entity_curve` / direct). Entity geometry that several features need (snap, extrude profiles, hatch boundaries, clipping) must come from `entity_curve`, not be re-derived.
- `entities/` — per-entity-type logic (grips, properties, transforms) over `document::EntityType`.
- `io/` — file formats and platform file access (STEP/IGES/OBJ/STL/SVG/PDF/WMF import-export, plot, xref, recovery, and the `web_*` browser-only pieces).
- `snap.rs` (OSNAP/OTRACK), `parametric.rs` + `scene/parametric_solve` + kernel-constraints (geometric/dimensional constraints), `i18n.rs` (Fluent), `ui/` (ribbon, statusbar, command line, widgets).
- Automation surfaces (desktop only): `mcp.rs` / `mcp_intent.rs` / `mcp_ops.rs` (MCP server driving the live editor via an authenticated bridge), `rest.rs` (+ `rest_openapi.json`), `app/automation.rs` + `app/control` (line-based JSON API, shared action names).
- Plugins: external plugins are `cdylib`s loaded in a **separate runner process** (host re-execs itself with `--ocs-plugin-runner <socket> <cdylib>`); host side in `src/plugin/` and `app/plugin_host`. Native only. See `docs/plugin-architecture.md`, `docs/plugin-template*/`, `plugins/registry.json`.

### Platform split
Anything desktop-only is gated `#[cfg(not(target_arch = "wasm32"))]` (cli, mcp, rest, gpu_backend, parametric, rayon, ureq). Code outside those gates must compile for wasm — hence the dedicated wasm `cargo check` CI job. Web-specific `web-sys` features are enumerated explicitly in `Cargo.toml`; new browser API use needs its feature added there.

### Build-time generation
`build.rs` stamps version/git revision into env vars (`OCS_APP_VERSION`, `OCS_GIT_REV`, …). `crates/ocs_plugin_api/build.rs` builds the serde-reflection type registry and entity-coverage tables from `entity_coverage_policy.json`.

## Repo notes
- Large files are normal (`src/command.rs`, `src/mcp.rs`, `src/snap.rs`, `src/locale_catalog.rs`); the `app/update` and `app/commands` trees were mechanically split from monoliths, so many files use `use super::util::*` glob imports.
- Benchmarks of note: `benches/performance_benchmarks.rs`, `benches/drawing_startup.py` (portable startup benchmark with `test_drawing_startup.py`).
- Docs worth reading before large changes: `docs/automation/README.md` (+ `API-SPEC.md`), `docs/plugin-host-model.md`, `docs/tessellation.md`, `docs/native-vs-web.md`.
