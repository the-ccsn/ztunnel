# CCSN adaptive ChaCha20 build

This fork starts from upstream tag `1.31.1`, commit
`a5cced0a5ade43273e9ba9b9c7a419450a9dacf3`.
The patch branch is `ccsn/1.31.1-chacha20`.

## Cipher selection

The AWS-LC (default) and ring providers advertise TLS 1.3 ChaCha20-Poly1305
alongside both upstream AES-GCM suites. Runtime detection checks AES and
PCLMULQDQ on x86/x86_64, and AES and PMULL on aarch64. Other architectures
conservatively prefer ChaCha20. Virtual machines use the features exposed
to the guest; the binary must be built for a generic CPU, without
`-C target-cpu=native` or unconditional AES target features.

| Client | Server | Negotiated TLS 1.3 cipher |
| --- | --- | --- |
| Accelerated patched | Accelerated patched | AES-256-GCM |
| Software-only patched | Accelerated patched | ChaCha20-Poly1305 |
| Accelerated patched | Software-only patched | ChaCha20-Poly1305 |
| Software-only patched | Software-only patched | ChaCha20-Poly1305 |
| Any patched | AES-only upstream | AES-GCM |
| AES-only upstream | Any patched | AES-GCM |

Software-only servers enforce their own suite order; accelerated servers
honor the client's order. Both peers must advertise ChaCha20 for it to be
negotiated. Upgrade both ends of a workload connection to benefit. Mutual
authentication, SPIFFE identity verification, ALPN `h2`, and protocol version
selection remain intact. The shared provider also advertises ChaCha20 on
outbound control-plane TLS; that peer chooses the negotiated cipher.

BoringSSL FIPS and OpenSSL retain the upstream AES-only policy. ChaCha20 is
not a FIPS-approved algorithm. TLS 1.2 suites remain AES-only, even when
`TLS12_ENABLED=true`. `COMPLIANCE_POLICY=pqc` key exchange remains intact.
No throughput improvement is claimed without benchmarking the target device.

## Build and verify

With upstream's documented build dependencies available:

```sh
cargo test --locked --lib tls::
cargo test --locked --lib --no-default-features --features tls-ring tls::
VERSION=1.31.1-ccsn-chacha20 cargo build --locked --release --bin ztunnel
out/rust/release/ztunnel version
```

On NixOS, provide missing dependencies with `nix shell`, including `protobuf`
for `protoc` and `llvmPackages.libclang` for the default jemalloc build. Set
`LIBCLANG_PATH` to that package's `lib` directory. The build script invokes
the build-info helper via Bash on PATH so it does not require `/bin/bash`.

The regression test `adaptive_cipher_negotiation` runs all nine combinations
of accelerated, software-only, and AES-only peers using the actual workload
certificate configs. It verifies the negotiated cipher, TLS 1.3, mTLS, ALPN,
and encrypted data in both directions without requiring special CPU hardware.

Build the container with the pinned upstream build-tools environment and
upstream runtime image:

```sh
podman build -f docker/Dockerfile.ccsn -t ztunnel:1.31.1-ccsn-chacha20 .
podman run --rm ztunnel:1.31.1-ccsn-chacha20 version
```

The GitHub Actions workflow tests and builds native amd64 and arm64 OCI image
archives. Download the architecture-specific artifact from the successful
workflow run and load it with `podman load -i ztunnel-amd64.tar` (or arm64).
Images are build artifacts; the workflow does not publish to a registry.
Before deploying, publish the selected image to your registry and update
the ztunnel Helm values to reference it. Keep the control plane, CNI, and
ztunnel on a compatible Istio version; this repository does not modify
the cluster or the GitOps configuration.
