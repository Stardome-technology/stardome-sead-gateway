# Stardome SEAD Gateway

**Single public HTTPS entry point for a SEAD node.** Terminates TLS, enforces
auth, and routes to the internal C++ services (sead-core, edge-service,
storage-gateway, source-data-service) over the shared `sead-network` bridge.
Runs as a sidecar alongside each SEAD stack.

## Architecture

```mermaid
graph LR
    subgraph "Public Internet"
        Client[Third-party / cross-org clients]
    end

    subgraph "SEAD Node"
        GW[Go Gateway :30080<br/>TLS · auth · validate · proxy · metrics]
        SC[sead-core :50051 gRPC]
        ED[edge-service :50055 gRPC]
        ST[storage-gateway :50052 gRPC]
        SD[source-data-service :50053 gRPC]
        GW -->|gRPC| SC
        GW -->|gRPC| ED
        GW -->|gRPC| ST
        GW -->|gRPC| SD
    end

    Client -->|HTTPS| GW
```

The gateway is the **only** public surface of a SEAD node. The C++ services are
**gRPC-only** and publish no public HTTP ports — they are reachable only on the
internal `sead-network` bridge. Cross-node sync fetch is gateway↔gateway HTTPS,
with the gateway's gRPC Sync server (`/sead_rpc.Sync`) on `GATEWAY_GRPC_PORT`
(50054) serving `sead-sync`'s cross-node fetch requests.

## Deploy

### Standalone gateway vs the combined SEAD stack

The gateway is **already bundled** inside the `stardome-sead`
[`docker-compose.remote.yml`](https://github.com/Stardome-technology/stardome-sead/blob/main/docker-compose.remote.yml)
as a `gateway` service. That combined stack is the normal path for a full SEAD
node — you should **not** deploy this repo's gateway alongside it, or you'll get
two gateways fighting over port `30080`.

In the combined stack the cert mount is `./secrets:/etc/gateway/certs:ro`, so
`server.crt` + `server.key` go in `~/stardome-sead/secrets/`. In this
standalone repo the mount is `./certs:/etc/gateway/certs:ro`, so they go in
`~/stardome-sead-gateway/certs/`. The deployed file names are the same either
way.

The **standalone** gateway repo is for cases where you want the gateway as the
single public HTTPS entry point **without** running the full C++ services, or
where the gateway runs on a separate host/VM from its backend:

- **IPFS-auth / minimal stack** — a minimal `sead-core` + gateway node that only
  exposes IPFS pinning (see
  [stardome-ipfs](https://github.com/Stardome-technology/stardome-ipfs)).
- **Gateway on its own host** — you run `stardome-sead` on node A and put this
  gateway on node B, pointing `SVC_*_GRPC` at A's services. Both must be on the
  same `sead-network` bridge (or the target must otherwise be DNS-resolvable
  from the gateway container).

In every case, at least one `sead-core` is required so the gateway can resolve
keys; the gateway cannot serve events/pins on its own.

### Prerequisites

- Docker + Docker Compose plugin
- A running SEAD stack (see [stardome-sead](https://github.com/Stardome-technology/stardome-sead))
- The gateway image is pre-built at `ghcr.io/stardome-technology/stardome-sead-gateway:latest` (multi-arch amd64 + arm64)

### Quick start

```bash
# 1. Configure runtime env
cp .env.example .env
#    edit .env — set GATEWAY_AUTH_SECRET, TLS, and service targets

# 2. Pull and run (joins the existing sead-network bridge)
docker compose -f docker-compose.remote.yml pull
docker compose -f docker-compose.remote.yml up -d

# 3. Verify (TLS on by default; -k for self-signed)
curl -k https://localhost:30080/health
```

> **Network:** the gateway joins the external `sead-network` bridge that the
> SEAD stack creates. It resolves the internal services by their compose
> service names (`sead-core`, `edge-service`, `storage-gateway`,
> `source-data-service`). Start the SEAD stack first so the network exists.

### TLS: public cert (production) vs self-signed (isolated/own-party)

The gateway serves whatever cert you point `GATEWAY_TLS_CERT`/`GATEWAY_TLS_KEY`
at. Choose based on who connects to `:30080`:

- **Production / cross-org (default posture) — public cert.** Use a
  Let's Encrypt (or other CA) certificate for the gateway's public hostname.
  Third-party orgs' brokers/nodes then trust it through the standard PKI with
  no manual CA distribution. Obtain one with `certbot` (see
  `stardome-sead/docs/setup-vps.md` for the Nginx+certbot flow) and set the
  cert/key paths in `.env`.
- **Isolated / own-party only — self-signed.** For isolated deployments where
  every node is under your control, a self-signed cert avoids public cert
  issuance. Mount `./certs/server.{crt,key}` and have your own clients trust
  the CA (`SEAD_CA_CERT` in the broker/explorer). Do **not** use this for a
  gateway reachable by genuine third parties.

> **Why not a CA "over the SEAD"?** SEAD already has its own trust model (the
> DAG: org genesis → edge authorization → XMSS signatures). TLS is only
> *transport* encryption — it must not become a second authority layered on
> top of the DAG. Keep transport trust (public PKI at the edge, private CA/mTLS
> internally) separate from SEAD identity.

## Public ports to open

For the gateway to be reachable by third-party / cross-org clients, open on the
host firewall (and any cloud security group):

- **`30080/tcp`** — HTTPS public API (all endpoints)

The gateway is the cross-org entry point (Go Gateway migration phase 4.4, replaces sead-core's former public port).
See the [public port contract](https://github.com/Stardome-technology/stardome-sead/blob/main/docs/public-port-contract.md)
for the full list of standardized federation ports.

The internal gRPC ports (`50051`–`50055`) are **private** and must NOT be
exposed publicly — they are reachable only on the `sead-network` bridge.

## Configuration

All configuration via environment variables (see `.env.example`):

| Variable | Default | Description |
|----------|---------|-------------|
| `GATEWAY_LISTEN_ADDR` | `:30080` | Address to listen on |
| `GATEWAY_TLS_ENABLED` | `true` | Enable TLS termination (integrator mounts a self-signed cert in `./certs`) |
| `GATEWAY_TLS_CERT` | `/etc/gateway/certs/server.crt` | Path to TLS certificate |
| `GATEWAY_TLS_KEY` | `/etc/gateway/certs/server.key` | Path to TLS private key |
| `GATEWAY_AUTH_ENABLED` | `true` | Enable Bearer token auth |
| `GATEWAY_AUTH_SECRET` | | Shared secret for token validation |
| `GATEWAY_EDGE_TOKENS` | | Comma-separated edge token secrets |
| `GATEWAY_AUTH_CBOR_ENABLED` | `true` | CBOR auth-token verification (key resolution via SVC_SEAD_CORE_GRPC) |
| `GATEWAY_AUTH_CBOR_CACHE_TTL` | `60` | Key cache TTL (s) |
| `GATEWAY_AUTH_CBOR_SKEW_TOLERANCE` | `30` | Token skew tolerance (s) |
| `GATEWAY_METRICS_ENABLED` | `true` | Enable `/metrics` endpoint |
| `GATEWAY_LOG_LEVEL` | `info` | Log level |
| `GATEWAY_READ_TIMEOUT` | `30s` | HTTP read timeout |
| `GATEWAY_WRITE_TIMEOUT` | `30s` | HTTP write timeout |
| `GATEWAY_IDLE_TIMEOUT` | `90s` | HTTP idle timeout |
| `GATEWAY_PROXY_TIMEOUT` | `30s` | Upstream proxy timeout |
| `SVC_SEAD_CORE_GRPC` | `sead-core:50051` | sead-core gRPC target |
| `SVC_EDGE_SERVICE_GRPC` | `edge-service:50055` | edge-service gRPC target |
| `SVC_STORAGE_GRPC` | `storage-gateway:50052` | storage-gateway gRPC target |
| `SVC_SOURCE_DATA_GRPC` | `source-data-service:50053` | source-data-service gRPC target |
| `GATEWAY_GRPC_PORT` | `50054` | Gateway Sync gRPC port |

## Endpoints

### Public API (authenticated)

| Path | Method | Backend |
|------|--------|---------|
| `/ingest` | POST | edge-service |
| `/receipt/{id}` | GET | edge-service |
| `/chain-of-custody/{hash}` | GET | edge-service |
| `/verification-package/{hash}` | GET | edge-service |
| `/events/{id}` | GET | sead-core |
| `/events/batch-resolve` | POST | sead-core |
| `/peers` | GET | sead-core |
| `/frontier-summary` | POST | sead-core |
| `/frontier` | GET | sead-core |
| `/orgs/{id}` | GET | sead-core |
| `/edges/{id}` | GET | sead-core |
| `/events/by-org/{org}/genesis` | GET | sead-core |
| `/events/by-edge/{org}/{edge}/authorizations` | GET | sead-core |
| `/events/by-org/{org}/revocations` | GET | sead-core |
| `/events/by-edge/{org}/{edge}/revocations` | GET | sead-core |
| `/events/{id}/dependencies` | GET | sead-core |
| `/pin` | POST | gateway (native Go IPFS client) |
| `/cid/{hash}` | GET | gateway (native Go IPFS client) |
| `/verify` | POST | gateway (collapsed verifier) |
| `/disclosure/request` | POST | source-data-service |
| `/source-data/{hash}` | GET | source-data-service |
| `/source-data/{hash}/audit` | GET | source-data-service |

### Internal (no auth)

| Path | Method | Description |
|------|--------|-------------|
| `/health` | GET | Health check |
| `/metrics` | GET | Prometheus metrics |

## Development

The gateway source lives in the private repo
(`stardome-sead-gateway-private`). This public repo is the deployment surface:
it references the published `ghcr.io` image and documents the runtime
configuration. See the private repo's `Makefile` for the multi-arch build/push
workflow.

## License

Apache License 2.0 — see [LICENSE](LICENSE).