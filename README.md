# relay-lb

**A small Layer-4 TCP load balancer in Go.** It accepts client connections, picks a healthy backend and pipes bytes both ways. It does health checks, retries, graceful draining and Prometheus metrics, in a codebase small enough to read in one sitting.

> **Status: in active development.** This README describes the planned design. Benchmarks and failover results are published only from real runs.

## Features
- **Algorithms:** round-robin and least-connections, healthy backends only
- **Active health checks** with up/down hysteresis (no flapping), plus passive failure counting
- **Retry on dial failure** onto a different backend
- **Correct TCP handling:** half-close (`CloseWrite`) propagation, idle timeouts, a max-connections guard that protects file descriptors
- **Graceful drain on SIGTERM:** fail `/healthz` first, stop accepting, let in-flight connections finish, force-close after a timeout
- **Observability:** Prometheus metrics (active connections, bytes, dial errors, backend up, connection-duration histogram), a `/status` JSON and a provisioned Grafana dashboard

## How it works
```
client ──TCP──▶ relay :8080 ──pick healthy backend (rr | least_conn)──▶ dial (retry on failure)
                   │                                                        │
                   └──── io.Copy client→backend ──▶  ◀── io.Copy backend→client ────┘
                         (EOF on one side → CloseWrite on the other: TCP half-close)

health checker: TCP dial every 2s per backend → down after 3 fails, up after 2 successes
admin :9100 → /metrics  /status  /healthz
```

## Planned usage
```bash
relay -config config.yaml
cd deploy && docker compose up    # relay + 3 nginx backends + Prometheus + Grafana (:3000)
```

## Stack
Go (standard library `net`, `io`, `sync/atomic`, `log/slog`), Prometheus client, Docker (distroless), Prometheus + Grafana, wrk for benchmarks, GitHub Actions CI with `go test -race`.

## License
MIT
