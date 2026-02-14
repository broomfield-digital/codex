# ARM32 Maintenance (JunoOS / TS-7800-v2)

This branch tracks the minimum deltas required to build `codex` for
`armv7-unknown-linux-gnueabihf` using the JunoOS cross-compilation setup.

## Baseline Environment

- Host: macOS (Apple Silicon)
- Target: TS-7800-v2 (`armv7l`, 32-bit Debian)
- Rust target: `armv7-unknown-linux-gnueabihf`
- Cross compiler prefix: `armv7-unknown-linux-gnueabihf-*`
- Toolchain location: `/opt/homebrew/bin`
- Sysroot:
  `/opt/homebrew/Cellar/armv7-unknown-linux-gnueabihf/13.3.0/toolchain/armv7-unknown-linux-gnueabihf/sysroot`

## Artifact Retention

Keep ARM32 intermediates and final outputs under the repo-local build directory:

- Intermediate artifacts: `build/arm32/intermediate/`
- Final binaries: `build/arm32/bin/`
- Build metadata/checksums: `build/arm32/ARTIFACTS.txt`

Avoid relying on `/tmp` for anything you need to keep.

## Branch Deltas Kept for ARM32

1. `codex-rs/linux-sandbox/src/landlock.rs`
   - Cast `libc::SYS_*` constants to `i64` for seccomp rule map compatibility on ARM32.
   - Skip seccomp filter install on unsupported architectures instead of panicking.
2. `codex-rs/linux-sandbox/build.rs`
   - If vendored bubblewrap cannot compile (for example missing `libcap` in cross sysroot),
     emit `cargo:warning` and continue instead of hard failing.
3. `codex-rs/Cargo.toml`
   - Set workspace `zip` dependency to `default-features = false`.
   - Enable features: `aes-crypto`, `deflate`, `time`, `zstd`.
   - Avoids pulling `bzip2/lzma` link requirements in this ARM32 cross environment.
4. `codex-rs/core/Cargo.toml`
   - Add `[target.armv7-unknown-linux-gnueabihf.dependencies]`.
   - Use `openssl-sys` with `features = ["vendored"]`.

## Cross-Build Command Template

```bash
PATH="/opt/homebrew/opt/rustup/bin:/opt/homebrew/bin:$PATH" \
RUSTFLAGS="-C link-arg=../build/arm32/intermediate/libendiancompat.a" \
CARGO_TARGET_ARMV7_UNKNOWN_LINUX_GNUEABIHF_LINKER=armv7-unknown-linux-gnueabihf-gcc \
CC_armv7_unknown_linux_gnueabihf=armv7-unknown-linux-gnueabihf-gcc \
CXX_armv7_unknown_linux_gnueabihf=armv7-unknown-linux-gnueabihf-g++ \
AR_armv7_unknown_linux_gnueabihf=armv7-unknown-linux-gnueabihf-ar \
RANLIB_armv7_unknown_linux_gnueabihf=armv7-unknown-linux-gnueabihf-ranlib \
PKG_CONFIG_ALLOW_CROSS=1 \
/opt/homebrew/opt/rustup/bin/cargo build -p codex-cli -p codex-linux-sandbox --release --target armv7-unknown-linux-gnueabihf
```

Notes:
- `build/arm32/intermediate/libendiancompat.a` is currently a temporary shim used to satisfy
  `le16toh`/`be16toh` symbols for this toolchain.
- Resulting binaries are dynamically linked and expect `/lib/ld-linux-armhf.so.3`.

## Validation Checklist (Target Device)

```bash
./codex --version
./codex --help
ldd ./codex
ldd ./codex-linux-sandbox
```

## Open Items

- Replace the `libendiancompat.a` temporary shim with an in-repo or toolchain-level fix.
- Revisit vendored bubblewrap support when `libcap` is available in ARM32 sysroot.
- Keep this branch rebased on `origin/main`; re-run ARM32 build + runtime checks after each rebase.
