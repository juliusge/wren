# Local setup (npm / Tauri)

Checklist to get `npm run tauri:dev` working on macOS.

## Toolchain versions

| Tool | Required | Notes |
|------|----------|--------|
| macOS + Xcode CLT | — | `xcode-select --install` |
| Node.js | 20+ | nvm recommended |
| Rust | 1.96+ | via [rustup](https://rustup.rs); 1.97.x works with current lockfile |
| `protoc` | any recent | needed by `lance-encoding` (LanceDB) |
| Tauri JS ↔ Rust | same major.minor | e.g. `@tauri-apps/api` `~2.10` ↔ crate `tauri` `2.10.x` |

Verified on: Node 25 / npm 11, rustc 1.97.1, protoc 35.1, Tauri 2.10.

## One-time installs

```bash
# Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source "$HOME/.cargo/env"

# Xcode Command Line Tools (skip if already installed)
xcode-select --install

# Protobuf compiler (LanceDB build)
brew install protobuf

# Bundled pdfium (not in git) — Apple Silicon
mkdir -p src-tauri/resources
curl -L https://github.com/bblanchon/pdfium-binaries/releases/latest/download/pdfium-mac-arm64.tgz \
  | tar -xzf - -C src-tauri/resources --strip-components=1 lib/libpdfium.dylib
# Intel: use pdfium-mac-x64 instead of pdfium-mac-arm64
```

## JS deps and run

```bash
npm install
npm run tauri:dev
```

Do **not** install the npm package `tauri` (legacy v1). This repo uses `@tauri-apps/cli` / `@tauri-apps/api` from `package.json`.

## Pitfalls

- **`npm audit` / `npm audit fix --force`** — ignore for setup. It only churns `epubjs` / xmldom and does not unblock Tauri.
- **Tauri version mismatch** — keep JS packages on `~2.10` (tilde) so they stay aligned with `src-tauri` crates. After a forced upgrade, re-pin:
  ```bash
  npm install @tauri-apps/api@~2.10.1 @tauri-apps/cli@~2.10.1 \
    @tauri-apps/plugin-dialog@~2.6.0 @tauri-apps/plugin-fs@~2.4.5
  ```
- **`ethnum` on Rust 1.97** — needs `>= 1.5.3` (already in `Cargo.lock`). If you see `E0512` / transmute in `ethnum`:
  ```bash
  cargo update -p ethnum --manifest-path src-tauri/Cargo.toml
  ```
- **`resources/*` not found** — means `src-tauri/resources/libpdfium.dylib` is missing; run the pdfium curl above.
