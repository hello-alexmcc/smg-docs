---
title: Hardening Defaults
---

# Hardening Defaults

What a gateway started with default flags does, and the flag that changes it. SMG keeps these defaults so a local start stays one command; a production deployment closes what it does not need. The table describes current `main`; a note marks what landed after the v1.12.0 release.

<div class="prerequisites" markdown>

#### Before you begin

- A gateway you can start and whose first log lines you can read

</div>

---

## What a default start leaves open

| Surface | Default | Flag that changes it |
|---------|---------|----------------------|
| Control-plane routes (`/workers`, `/flush_cache`, `/start_profile`, `/stop_profile`, `/parse/*`, `/wasm`, `/v1/tokenizers`, `/get_loads`) | Open to anyone who can reach the port. The first log lines carry one `WARN` record whose message starts with `SECURITY POSTURE: this gateway runs without authentication`, with a field per open surface naming its routes and the flag that closes them. | `--control-plane-api-keys id:name:admin:<key>`, or `--jwt-issuer` with `--jwt-audience`; the shared `--api-key` gates them too. See [Default posture](control-plane-auth.md#default-posture). |
| Serving routes (`/v1/chat/completions`, `/v1/completions`, `/v1/responses`, `/v1/embeddings`, `/v1/messages` and the rest) | Open. | `--api-key <key>`, or `--tenant-api-key tenant:<key>` per tenant. |
| Metrics listener | `0.0.0.0:29000` (`::` when `--host` is an IPv6 address), without authentication in any configuration. | `--prometheus-host` to a private address, and a network policy that admits only the scraper. See [Monitoring](monitoring.md#enable-metrics). |
| Mesh listener (`--enable-mesh`) | Gossips membership and state in clear text with any peer that connects; named in the posture `WARN`. | `--mesh-tls-ca-cert`, `--mesh-tls-cert` and `--mesh-tls-key` (peers then need a certificate from that CA). |
| Cross-origin requests | Any origin, method and header is allowed (`Access-Control-Allow-Origin: *`). A change that makes the default no cross-origin allowance is in review (smg-project/smg#3315). | `--cors-allowed-origins https://app.example` (space-separated origins). |
| Request body size | 512 MiB (`--max-payload-size 536870912`). | `--max-payload-size <bytes>`. |
| Output length | No `max_tokens` ceiling at the gateway: a request's `max_tokens` is bounded by the engine's context length only. | None at the gateway; bound it at the engine. |
| Concurrent requests | v1.12.0: no bound unless set. `main` after v1.12.0 (smg-project/smg#2927): the default `-1` derives the bound from the host (1024 per available core, at least 4096) and sheds with `429` or `503` beyond it. | `--max-concurrent-requests <n>` (`0` disables the bound on `main`). |
| Stored history (`--history-backend memory`, the default) | v1.12.0: stored responses and conversation items are kept without a cap. `main` after v1.12.0 (smg-project/smg#3244): at most 10,000 responses and 100,000 items, oldest first, and a 24-hour retention. | `--history-backend none` when history is not needed; on `main`, `--history-memory-max-responses`, `--history-memory-max-items` and `--history-memory-retention-secs`. |
| Keys on the command line | The values of `--api-key`, `--tenant-api-key` and `--control-plane-api-keys` are visible in the process list (`ps`, `/proc/<pid>/cmdline`). | `CONTROL_PLANE_API_KEYS` carries the control-plane keys as an environment variable. The data-plane keys have no environment form: inject the flag from a secret at start and keep the process list private. |

## Routes that stay public

The probe and discovery routes answer without a credential in every configuration: `/health`, `/readiness`, `/liveness`, `/health_generate`, `/v1/models`, `/get_model_info`, `/get_server_info`, `/engine_metrics` and `/loads` (which names each backend by its address). Nothing closes them but a network policy on the gateway's port; a dedicated probe listener (`--health-check-port`) serves `/health`, `/readiness` and `/liveness` on its own port, but the same routes stay on the main port too.

## Ports below 1024

The published image runs as uid 65532 and the Helm chart drops every capability, so a gateway in that image cannot bind a port below 1024: `--port 443` fails there. Serve on a high port behind a Service, an ingress or a load balancer, or grant the container `CAP_NET_BIND_SERVICE`.

## Configurations the gateway refuses at start-up

Each of these ends the start with an error that names the flag; nothing is served:

- a `--control-plane-api-keys` entry that is not `id:name:role:key`
- `--jwt-issuer` without `--jwt-audience`, or the reverse (JWT authentication would otherwise be silently off)
- two listeners on one port: `--port`, `--prometheus-port`, `--health-check-port` and `--mesh-port` each need their own
- an empty `--api-key` (authentication would be on with a key nothing can present)
- `--tls-cert-path` without `--tls-key-path`, or the reverse, or a file the gateway cannot read or use
- a `--cors-allowed-origins` value that is not exactly an origin (`scheme://host[:port]`, lower-case, no path)
- a `--mesh-port` another process holds (`main` after v1.12.0, smg-project/smg#3328)

## Start-up warnings worth acting on

- The posture `WARN` above: every surface it names answers any caller that can reach it.
- On `main` after v1.12.0 (smg-project/smg#3310), a TLS private key file that its group or others can read (a mode other than `0600` or `0400`) logs a `WARN` naming the file and the flag (`--tls-key-path`, `--client-key-path`, `--mesh-tls-key`); the start continues.

## What the access log carries

Each request leaves two records under a span with the method, the path, the HTTP version, the request id and, when tracing is on, the trace id; the response record adds the status and the latency. No headers, no bodies, no prompts. On `main` after v1.12.0 (smg-project/smg#3306) the span carries the path only, never the query string, so a credential a client mistakenly puts in `?api_key=` does not reach the log; v1.12.0 logs the full URI. See [Request and Response Logs](logging.md#request-and-response-logs).

---

## What's Next?

- [Control Plane Auth](control-plane-auth.md): keys and JWT for the control plane
- [Configure TLS](tls.md): HTTPS on the gateway and its fixed TLS policy
- [Configuration Reference](../reference/configuration.md): every flag with its default
