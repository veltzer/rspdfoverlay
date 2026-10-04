# TOFIX

Findings from a code scan on 2026-10-04.

## High

- `src/main.rs:1` - the crate is still a `println!("Hello, World!")` stub, yet `release.toml:15` (`publish = true`) has already pushed four versions (latest 0.1.4) to crates.io under the description "Overlay data on pdf files". Either implement the MVP from `DESIGN.md` section 9 before the next release, or stop publishing (set `publish = false`) until there is a working tool, so crates.io users do not install an empty binary.

## Medium

- `src/main.rs:12` - the only test calls `main()` and asserts nothing (the comment says so); once any real code lands, replace it with tests of the pure parts first, e.g. the screen-to-PDF coordinate conversion specified in `DESIGN.md:94`-`DESIGN.md:96`.
- `src/main.rs:1` - `build.rs` exports `GIT_SHA`, `GIT_DESCRIBE`, `BUILD_TIMESTAMP`, `RUSTC_SEMVER` and `RUST_EDITION`, but `main.rs` reads none of them. Add the fleet's `--version` / version-info output (as `rsconstruct/src/main.rs` does) so the binary reports what build it is.

## Low

- `config/project.lua:7` - keyword `cli` contradicts the design (`DESIGN.md:3` "Desktop Application", egui GUI); replace with e.g. `gui` or `desktop`.
- `Cargo.toml:1` - `[package]` has no `keywords`, `categories` or `readme`, so the published crate shows none on crates.io; add them (e.g. keywords from `config/project.lua`, categories `["gui", "text-processing"]`).
- `README.md:9` - says "Documentation lives in the `docs/` mdBook", but `docs/src/introduction.md` is a single sentence and the real design lives only in `DESIGN.md`; add `DESIGN.md` to the book (a `docs/src/design.md` page linked from `docs/src/SUMMARY.md`) or point the README at `DESIGN.md`.
