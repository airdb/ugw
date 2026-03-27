# UGW — Unified Gateway

> A high-performance, Rust-powered unified proxy gateway designed as a complete replacement for Nginx.

[![Build Status](https://img.shields.io/badge/build-passing-brightgreen)](https://github.com/your-org/ugw)
[![License](https://img.shields.io/badge/license-MIT-blue)](LICENSE)
[![Rust](https://img.shields.io/badge/rust-stable-orange)](https://www.rust-lang.org/)

---

## Overview

**UGW** (Unified Gateway) is a modern, all-in-one proxy solution built with Rust, delivering safety, performance, and extensibility. It unifies three critical gateway roles into a single binary — replacing Nginx for reverse proxying, persistent connection management, and forward egress proxying.

```
                         ┌─────────────────────────────────────┐
                         │               UGW                   │
                         │                                     │
  HTTP/HTTPS   ────────► │  AGW  (Reverse Proxy / API GW)     │ ────► Upstream Services
  WebSocket    ────────► │  CGW  (Long-lived Connection GW)   │ ────► Backend Systems
  Outbound     ◄──────── │  EGW  (Egress / Forward Proxy)     │ ────► External Internet
                         └─────────────────────────────────────┘
```

---

## Components

### AGW — API Gateway (Reverse Proxy)

AGW handles all inbound HTTP/HTTPS traffic and acts as the front door to your internal services. It is the direct counterpart to Nginx's reverse proxy functionality.

**Key features:**
- HTTP/1.1, HTTP/2, and HTTP/3 (QUIC) support
- TLS termination with automatic certificate management (ACME / Let's Encrypt)
- Load balancing: round-robin, least-connections, weighted, consistent hashing
- Path-based and host-based routing
- Request/response rewriting, header manipulation
- Rate limiting and circuit breaking
- Middleware pipeline (authentication, logging, tracing)
- gRPC proxying support

---

### CGW — Connection Gateway (Long-lived Connection Proxy)

CGW is purpose-built for stateful, long-lived connections. It manages persistent channels between clients and backend systems with full lifecycle awareness.

**Key features:**
- WebSocket proxying with connection lifecycle management
- Server-Sent Events (SSE) forwarding
- gRPC streaming (unary, server-streaming, client-streaming, bidirectional)
- TCP/UDP tunnel proxying
- Connection multiplexing and keep-alive management
- Backpressure handling and flow control
- Automatic reconnection and session resumption

---

### EGW — Egress Gateway (Forward Proxy)

EGW controls and proxies all outbound traffic originating from internal services. It provides visibility, control, and enforcement over egress connections to external destinations.

**Key features:**
- HTTP CONNECT tunneling
- HTTPS forward proxy with SNI inspection
- SOCKS5 proxy support
- Egress policy enforcement (allowlist / denylist)
- Traffic auditing and outbound request logging
- Bandwidth throttling per service or destination
- Transparent proxy mode

---

## Why UGW?

| Feature | Nginx | UGW |
|---|---|---|
| Language | C | Rust (memory-safe) |
| Long-lived connections | Limited | First-class (CGW) |
| Egress proxy | Not built-in | First-class (EGW) |
| Config format | Custom DSL | TOML / YAML |
| Dynamic reconfiguration | Reload required | Hot reload via API |
| Observability | Basic logs | Structured logs + metrics + traces |
| Binary size | Medium | Single static binary |

---

## Getting Started

### Prerequisites

- Rust `1.75+` (install via [rustup](https://rustup.rs/))

### Build from Source

```bash
git clone https://github.com/your-org/ugw.git
cd ugw
cargo build --release
```

### Run

```bash
# Start UGW with a config file
./target/release/ugw --config ugw.toml

# Start with a specific component only
./target/release/ugw --config ugw.toml --component agw
./target/release/ugw --config ugw.toml --component cgw
./target/release/ugw --config ugw.toml --component egw
```

---

## Configuration

UGW uses a single `ugw.toml` file. Each component has its own section.

```toml
[agw]
bind = "0.0.0.0:443"
tls  = { cert = "/etc/ugw/cert.pem", key = "/etc/ugw/key.pem" }

[[agw.routes]]
host    = "api.example.com"
path    = "/v1/"
upstream = "http://backend-service:8080"

[cgw]
bind          = "0.0.0.0:8443"
protocols     = ["websocket", "grpc-stream", "sse"]
idle_timeout  = "60s"

[egw]
bind     = "0.0.0.0:3128"
mode     = "forward"
policy   = "allowlist"
allowed  = ["api.stripe.com", "api.github.com"]

[observability]
log_level  = "info"
metrics    = { enabled = true, port = 9090 }
tracing    = { enabled = true, endpoint = "http://jaeger:4317" }
```

---

## Architecture

UGW is built on an async Rust runtime ([Tokio](https://tokio.rs/)) and leverages a shared core for connection management, TLS, and observability. Each gateway component (AGW, CGW, EGW) runs as an independent async task pool with a unified control plane.

```
┌────────────────────────────────────────────────────┐
│                    UGW Process                     │
│                                                    │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│   │   AGW    │  │   CGW    │  │   EGW    │        │
│   │  Task    │  │  Task    │  │  Task    │        │
│   │  Pool    │  │  Pool    │  │  Pool    │        │
│   └────┬─────┘  └────┬─────┘  └────┬─────┘        │
│        │              │              │              │
│   ┌────▼──────────────▼──────────────▼─────┐       │
│   │           Shared Core                  │       │
│   │  TLS Engine │ Router │ Observability   │       │
│   └─────────────────────────────────────────┘      │
│                                                    │
│   ┌────────────────────────────────────────┐       │
│   │         Admin API  (HTTP REST)         │       │
│   │   hot reload · health · metrics dump   │       │
│   └────────────────────────────────────────┘       │
└────────────────────────────────────────────────────┘
```

---

## Observability

UGW emits structured logs, Prometheus-compatible metrics, and OpenTelemetry traces out of the box.

```bash
# Prometheus metrics endpoint
curl http://localhost:9090/metrics

# Health check
curl http://localhost:9091/health

# Live config reload (no restart)
curl -X POST http://localhost:9091/admin/reload
```

---

## Roadmap

- [x] Project scaffold & core async runtime
- [ ] AGW: HTTP/1.1 + HTTP/2 reverse proxy
- [ ] AGW: TLS termination & ACME support
- [ ] CGW: WebSocket proxying
- [ ] CGW: gRPC streaming
- [ ] EGW: HTTP CONNECT forward proxy
- [ ] EGW: SOCKS5 support
- [ ] Unified admin API
- [ ] WASM-based plugin system
- [ ] Kubernetes-native control plane integration

---

## Contributing

Contributions are welcome. Please open an issue to discuss major changes before submitting a PR.

```bash
# Run tests
cargo test

# Run with verbose logging
RUST_LOG=debug cargo run -- --config ugw.toml

# Lint
cargo clippy -- -D warnings
```

---

## License

MIT License. See [LICENSE](LICENSE) for details.

