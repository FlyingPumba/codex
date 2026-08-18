# Local build notes

How to build this fork for macOS (the copy on PATH) and for linux x86_64 (shipped
to remote pods). Commands run from `codex-rs/` unless stated otherwise.

## macOS arm64

```bash
cd codex-rs
cargo build --release -p codex-cli --bin codex
```

The binary lands at `codex-rs/target/release/codex`. `~/bin/codex` symlinks to it,
and `~/bin` comes before Homebrew on PATH, so it shadows the `codex` cask.

The code-execution tool needs a second binary, `codex-code-mode-host`, which
embeds V8 through the `v8` crate. Building it from source pulls V8, so
`~/bin/codex-code-mode-host` points at the cask's copy instead:

```bash
ln -sf /opt/homebrew/Caskroom/codex/<version>/bin/codex-code-mode-host ~/bin/codex-code-mode-host
```

Keep that cask version in step with this workspace's version whenever code-mode
matters.

## linux x86_64, for remote pods

Cross-compiled on the mac, with zig providing the C toolchain. One-time setup:

```bash
brew install zig cmake
cargo install --locked cargo-zigbuild
rustup target add x86_64-unknown-linux-musl
```

The usual path goes through remote-vm-scripts, which builds, strips, gzips, and
caches the result:

```bash
source ~/src/remote-vm-scripts/main.sh
build-codex-linux
```

Underneath, that runs:

```bash
cd codex-rs
CARGO_TARGET_DIR="$HOME/.cache/codex-linux-target" \
CARGO_PROFILE_RELEASE_STRIP=symbols \
cargo zigbuild --release --target x86_64-unknown-linux-musl -p codex-cli --bin codex
```

Artifacts go to `~/.cache/codex-linux/`: `codex-x86_64-unknown-linux-musl.gz`
(96MB, 239MB unpacked, static ELF) alongside a JSON sidecar recording the commit,
version, and binary sha256. `setup-codex` and `setup-k8s-codex` push that archive
to pods and gunzip it there, so nothing is rebuilt or recompressed per pod.

### Why it is built this way

Building `--bin codex` alone keeps V8 out of the picture: `codex-cli` has no
dependency on the `v8` crate, which only `code-mode-runtime` and `v8-poc` use. A
full-workspace build would pull it in and need a V8 toolchain. cmake and a C
cross-compiler are still required, since aws-lc-sys, ring, and oniguruma all
compile C for the target.

The target is musl rather than gnu because `core/Cargo.toml` declares
`openssl-sys` with the `vendored` feature for the two musl triples, so a musl
build compiles OpenSSL from source and needs nothing from the host. A gnu build
still reaches `openssl-sys` through `reqwest` and `native-tls`, but unvendored,
which would mean supplying linux OpenSSL headers on the mac.

The separate `CARGO_TARGET_DIR` keeps `target/release/codex`, the mac binary on
PATH, from being rebuilt or overwritten, and keeps the two toolchains'
fingerprints from invalidating each other.

Pods get their sibling binaries from upstream releases rather than from here.
`codex-code-mode-host` embeds V8 and `bwrap` needs a musl libcap that CI builds
from source, so both are fetched per pod from
`https://github.com/openai/codex/releases/download/rust-v<version>/` for the same
version as this workspace. `bundled_bwrap` only verifies a digest when
`CODEX_BWRAP_SHA256` is set at compile time, which cargo builds leave unset, so
the release `bwrap` is accepted.

### Cargo.lock will change

The committed `codex-rs/Cargo.lock` lists workspace members at version `0.0.0`,
and any plain cargo build rewrites them to the real workspace version. `--locked`
fails the build instead of preventing the rewrite, so just commit the churn.

## Build times

A cold cross-build is about 27 minutes on an M-series mac: 991 crates, thin LTO,
plus vendored OpenSSL and aws-lc compiled through zig. A warm no-op relink takes
about 2 seconds, and `build-codex-linux` skips the 239MB gzip when the linked
binary's sha256 still matches the cached archive.
