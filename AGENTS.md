# AGENTS.md — TNG (Trusted Network Gateway)

> This file is intended for AI coding agents working on the TNG repository.
> All information below is derived from the actual project files (`Cargo.toml`,
> `Makefile`, `Dockerfile`, `trusted-network-gateway.spec`, `CLAUDE.md`,
> `.github/workflows/`, `docs/`, and source code).

---

## Project Overview

**TNG (Trusted Network Gateway)** is a transparent network gateway that
establishes end-to-end encrypted tunnels with remote attestation for
confidential-computing environments. It is based on the IETF RATS standard
(RFC 9334) and integrates with the CoCo (confidential-containers) ecosystem.

- **Repository:** `https://github.com/inclavare-containers/tng`
- **License:** Apache-2.0
- **Workspace version:** `2.6.0` (defined in the root `Cargo.toml`)
- **Primary language:** Rust (edition 2021)
- **Documentation language:** English (Chinese translations live in `*_zh.md` files)

TNG runs as a pair of instances:

- **Ingress** — the tunnel entry on the client side. Plaintext traffic enters
  here, gets encrypted, and is forwarded through the tunnel.
- **Egress** — the tunnel exit, usually inside a TEE. It decrypts traffic and
  forwards it to the destination service.

The supported transport protocols are:

- **RA-TLS** (default) — TLS 1.3 handshake extended with remote-attestation
  evidence, suitable for arbitrary TCP traffic.
- **OHTTP** — Oblivious HTTP message-level encryption, suitable for HTTP APIs
  and browser SDK integration.

---

## Repository Layout

```
/                       Cargo workspace root
├── tng/                Core gateway crate (CLI + library)
├── rats-cert/          Certificate and remote-attestation library
├── tng-testsuite/      Integration / e2e test crate
├── tng-wasm/           WebAssembly module + JavaScript SDK
├── tng-python/         Python SDK (not a Cargo member; uses hatch/maturin)
├── deps/
│   └── hyper-util-shim/  Local Cargo patch for hyper-util compatibility
├── docs/               Architecture, configuration, scenario, and developer guides
├── scripts/            Build / deploy helpers
├── dist/               Default config.json and systemd service file
├── rpm/                RPM packaging files
├── APPLICATION/        Internal image buildspec
├── Cargo.toml          Workspace manifest
├── Makefile            Common build/test/packaging targets
├── Dockerfile          Official container image build
├── trusted-network-gateway.spec   RPM spec
└── rust-toolchain.toml MSRV: 1.89.0
```

### Crate responsibilities

| Crate | Role |
|-------|------|
| `tng` | Main executable `tng`, CLI, JSON config parsing, `TngRuntime`, ingress/egress services, RA-TLS/OHTTP protocol handling, observability. |
| `rats-cert` | Certificate generation, evidence/token handling, CoCo/ITA attestation providers, optional builtin attestation-service verifiers. |
| `tng-testsuite` | Scenario-based integration tests (HTTP proxy, netfilter, SOCKS5, OHTTP, RA flows). Feature-gated for running against source code, a prebuilt binary, or Podman. |
| `tng-wasm` | Browser SDK (`tng_fetch`) compiled with `wasm-pack` to an npm package. |
| `tng-python` | Python SDK that launches a local TNG `http_proxy` subprocess and wraps `requests`, `httpx`, and `openai` clients. |
| `deps/hyper-util-shim` | Replaces `hyper-util` via `[patch.crates-io]` so the workspace pins compatible APIs. |

---

## Technology Stack

- **Async runtime:** Tokio (multi-thread on native, `tokio_with_wasm` in the browser).
- **HTTP/TLS stack:** hyper, hyper-util, rustls, tokio-rustls.
- **Web framework:** axum (control interface, OHTTP server, HTTP proxy ingress).
- **Serialization:** serde + serde_json.
- **CLI:** clap.
- **Observability:** tracing, tracing-subscriber, OpenTelemetry metrics/traces.
- **Remote attestation:** attestation-service, reference-value-provider-service,
  kbs-types, ttrpc (for Attestation Agent), protobuf/tonic (for Attestation Service).
- **Clustering / key sharing:** serf (used by OHTTP peer-shared key mode).
- **WASM toolchain:** wasm-bindgen, wasm-pack, nightly Rust.

Several upstream crates are patched from the Inclavare-Containers forks
(`rustls`, `rustls-webpki`, `reqwest`, `tokio-graceful`, `hyper`, `memberlist`).
`hyper-util` is replaced by the local `deps/hyper-util-shim`.

---

## Build Requirements

- **MSRV:** Rust `1.89.0` (from `rust-toolchain.toml`).
- **Nightly:** `nightly-2025-07-07` is required for:
  - Building `tng-wasm`.
  - Creating the vendored source tarball via `make create-tarball`.
- **System dependencies:** `protoc` (Protocol Buffers compiler), `clang` (see
  `.cargo/config.toml`), `openssl-devel`, `iptables`, `iproute`, `git`, `gcc`.
- **Cargo config:** `.cargo/config.toml` forces `CC = "clang"` because
  `aws-lc-sys` triggers a false-positive compiler-bug detection with GCC 10.2.1.
  Do not remove this.

---

## Common Commands

### Build the core binary

```bash
# Debug build
RUSTFLAGS="--cfg tokio_unstable" cargo build -p tng

# Release build (Makefile wraps this)
make bin-build

# Install locally
cargo install --locked --path ./tng/ --root /usr/local/
```

### Lint and format

```bash
make clippy              # cargo clippy --all-targets -- -D warnings
cargo fmt --check        # formatting check
```

The project pre-commit checklist (from `README.md` / `CLAUDE.md`) is:

```bash
make clippy && cargo fmt --check && cargo build
```

### Run TNG

```bash
tng launch --config-file /etc/tng/config.json
# or
tng launch --config-content='<json string>'
```

### Run tests

```bash
# Full source-code test suite (needs privileged environment + service deps)
make run-test

# With coverage (uses cargo-llvm-cov)
make run-test-coverage

# Run tests against a prebuilt binary instead of compiling tng into tests
make run-test-on-bin

# Run tests against a Podman container
make run-test-on-podman
```

Many integration tests need a running **Attestation Agent** (AA) and
**Attestation Service** (AS):

```bash
# In one terminal / background
make test-dep-aa   # Provides AA via ttrpc socket and ASR HTTP proxy on 127.0.0.1:8006
make test-dep-as   # Provides restful AS on 0.0.0.0:8080
```

Then run `cargo test` or `make run-test`.

### WASM SDK

```bash
# Debug / release builds (requires nightly-2025-07-07 and wasm-pack)
make wasm-build-debug
make wasm-build-release

# Pack an installable .tgz
make wasm-pack-debug
make wasm-pack-release

# Browser unit tests
make wasm-unit-test-chrome
make wasm-unit-test-firefox

# Integration test with the JS SDK
make wasm-integration-test
```

### Python SDK

```bash
# Build a wheel for the current platform (embeds target/release/tng)
make python-wheel

# Run Python tests
make python-test
```

### Packaging

```bash
# Docker image
make docker-build

# Vendored source tarball + RPM
make create-tarball
make rpm-build
# or in a clean container
make rpm-build-in-docker

# Cross-compiled binaries (CI uses zigbuild)
make mac-cross-build
make windows-cross-build
```

### Benchmark

```bash
make bench            # raw TCP vs stunnel vs TNG in network namespaces
make bench-multiplex  # same with rats_tls multiplex=true
```

### Version bump

```bash
make bump-version-patch
make bump-version-minor
make bump-version-major
```

---

## Code Organization

### `tng/src` layout

| Module | Purpose |
|--------|---------|
| `bin/tng/` | CLI (`cli.rs`) and `main.rs`. |
| `config/` | JSON configuration types (`TngConfig`, ingress/egress/RA/observability args). |
| `control_interface/` | RESTful and TTRPC admin/control APIs. |
| `observability/` | OpenTelemetry metrics and traces, plus structured logging setup. |
| `runtime.rs` | `TngRuntime` — builds services from config and runs them under graceful shutdown. |
| `service.rs` | `RegisteredService` trait that every ingress/egress service implements. |
| `state.rs` | Shared runtime state. |
| `tunnel/` | Core tunnel logic. |

### `tunnel/` submodules

- `ingress/` — mapping, http_proxy, socks5, netfilter ingress implementations.
- `egress/` — mapping and netfilter egress implementations.
- `ingress/protocol/rats_tls/` and `egress/protocol/rats_tls/` — RA-TLS security layer.
- `ingress/protocol/ohttp/` and `egress/protocol/ohttp/` — OHTTP security layer.
- `provider/` — polymorphic attestation-provider layer (CoCo, ITA). Defines
  `TngAttester`, `TngConverter`, `TngVerifier`, `TngEvidence`, `TngToken`.
- `utils/runtime/` — `TokioRuntime` wrapper. Use its `spawn_*` methods instead
  of raw `tokio::spawn`.

### `rats-cert` layout

| Module | Purpose |
|--------|---------|
| `attester.rs` | Generates attestation evidence via the Attestation Agent. |
| `verifier.rs` | Verifies evidence/tokens via the Attestation Service. |
| `converter.rs` | Converts evidence to attestation tokens (passport model). |
| `evidence.rs` / `token.rs` | Wire-format types. |
| `builtin_as.rs` | Optional local evidence verification (TDX/SGX/SNP). |

### Feature flags

`rats-cert` features control which providers are compiled in:

- `attester-coco`, `verifier-coco`, `attester-ita`, `verifier-ita`
- `crypto-rustcrypto`
- `builtin-as-tdx`, `builtin-as-sgx`, `builtin-as-snp`, `builtin-as-all`

`tng` features control ingress/egress modes and observability:

- `ingress-all`, `ingress-mapping`, `ingress-http-proxy`, `ingress-netfilter`, `ingress-socks5`
- `egress-all`, `egress-mapping`, `egress-netfilter`
- `tokio-console`, `metric`, `trace`
- `builtin-as-*` (forwarded to `rats-cert`)

Default features include `tokio-console`, `metric`, `trace`, and all
ingress/egress modes.

---

## Code Style Guidelines

- **Deny `unwrap`/`expect` in production code:** `tng/src/lib.rs` and
  `tng/src/bin/tng/main.rs` both contain `#![deny(clippy::unwrap_used)]` and
  `#![deny(clippy::expect_used)]`. Tests are allowed to use them via
  `clippy.toml`.
- **No raw `tokio::spawn`:** `clippy.toml` disallows `tokio::task::spawn`,
  `tokio::spawn`, and `tokio::runtime::handle::Handle::spawn`. Use the
  `TokioRuntime::spawn_*()` family instead.
- **Error handling:** prefer `anyhow::Result` at application boundaries and
  `thiserror`-derived errors in libraries (`rats-cert`).
- **Formatting:** standard `rustfmt` (`cargo fmt`).
- **Clippy:** `make clippy` treats warnings as errors.
- **Comments:** preserve existing comments, especially URLs that point to issues
  or documentation (see `CLAUDE.md`).

---

## Testing Strategy

- **Unit tests:** live in `tng/src/...` and `rats-cert/src/...`, run via
  `cargo test --workspace --bins --lib --exclude tng-wasm`.
- **Integration tests:** organized by functional domain under
  `tng-testsuite/tests/` (basic, http, netfilter, ohttp, js_sdk). Each test file
  is declared with a `[[test]]` entry in `tng-testsuite/Cargo.toml`.
- **Test runner:** `tng-testsuite/run-test.sh` compiles test binaries, runs unit
  tests, then iterates over all integration test files.
- **Coverage:** `make run-test-coverage` uses `cargo-llvm-cov`; results go to
  `target/codecov.json`.
- **External service dependencies:**
  - Attestation Agent (AA) ttrpc socket:
    `/run/confidential-containers/attestation-agent/attestation-agent.sock`
  - API Server Rest (ASR) HTTP proxy: `127.0.0.1:8006` (started by `make test-dep-aa`)
  - Attestation Service (AS) REST: `0.0.0.0:8080` (started by `make test-dep-as`)
- **WASM tests:** headless browser tests via `wasm-pack test`.
- **Python tests:** `pytest tests/test_unit.py` and `pytest tests/test_integration.py -m e2e`.

---

## Security Considerations

- **Remote attestation is the trust root.** TNG verifies the peer TEE identity
  and runtime measurements before allowing traffic.
- **`no_ra` is for debugging only.** It disables attestation and must not be
  used in production.
- **OHTTP key management modes:**
  - `self_generated` — ephemeral/local keys (default for simple deployments).
  - `file` — load HPKE keys from disk with automatic reload on changes.
  - `peer_shared` — distribute keys over a serf cluster via an existing RA-TLS
    channel.
- **Builtin Attestation Service:** the `builtin-as-*` features allow local
  evidence verification without a remote AS. They need a correctly configured
  PCCS. Use `scripts/setup-vendor-config` (e.g. `setup-vendor-config aliyun`) to
  configure vendor-specific PCCS endpoints.
- **AS policies:** production deployments should enforce strict OPA policies
  instead of the default-allow debug policy shown in the developer guide.
- **Credentials:** AS signer certificates/keys and AA sockets must be provided
  via secure configuration, never hard-coded.
- **Supply chain:** upstream crypto crates (`aws-lc-sys`) require `clang`; the
  workspace enforces this in `.cargo/config.toml`.

---

## Deployment & Packaging

| Form | How it's built | Notes |
|------|----------------|-------|
| Docker image | `make docker-build` / `Dockerfile` | Two-stage build based on Alibaba Cloud Linux 3; installs TDX DCAP runtime deps. |
| RPM package | `make create-tarball && make rpm-build` | Produces `trusted-network-gateway-<version>-1.x86_64.rpm`; installs `/usr/bin/tng`, `/etc/tng/config.json`, and a systemd unit. |
| Binary archive | CI `build-binary.yml` via `zigbuild.yml` | Cross-compiles to Linux, macOS, and Windows targets. |
| Python wheel | `make python-wheel` | Embeds the platform `tng` binary. |
| npm package | `make wasm-pack-release` | Published as `@inclavare-containers/tng`. |

The installed systemd unit (`dist/trusted-network-gateway.service`) runs:

```
/usr/bin/tng launch --config-file /etc/tng/config.json
```

---

## Key Documentation References

- `docs/architecture.md` — Ingress/Egress model, RA roles, RA-TLS vs OHTTP.
- `docs/configuration.md` — Complete JSON configuration reference.
- `docs/developer.md` — Developer environment, build from source, test setup.
- `docs/scenarios/` — Deployment topology examples with full configs.
- `docs/version_compatibility.md` — Breaking changes and migration notes.
- `tng-wasm/README.md` and `tng-python/README.md` — SDK usage.
- `CLAUDE.md` — Additional project-specific guidelines for refactoring, commits,
  PRs, and known environment limitations.

---

## Known Environment Limitations

(From `CLAUDE.md` — these are environment issues, not code defects.)

- `aws-lc-sys` may report a compiler bug with GCC 10.2.1; the workaround is the
  `CC=clang` setting in `.cargo/config.toml`.
- If `clippy` or `rustfmt` is missing, install them:
  `rustup component add clippy` / `rustup component add rustfmt`.
- Some CI environments hit 403 errors from crates.io for certain crates due to
  network restrictions.
- `rats-cert` requires `protoc`; install it if you see a "Could not find protoc"
  error.

---

## Quick Agent Checklist

Before committing or creating a PR:

1. `make clippy` passes.
2. `cargo fmt --check` passes.
3. `cargo build` passes.
4. Relevant integration tests in `tng-testsuite` pass (start `make test-dep-aa`
   and `make test-dep-as` first if the test touches attestation).
5. Documentation updated (`docs/configuration.md` and `docs/configuration_zh.md`
   for config changes).
6. No `Co-Authored-By:` trailers, no GPG signatures, and no AI attribution in
   commits (per `CLAUDE.md`).
