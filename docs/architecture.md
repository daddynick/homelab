# Homelab Architecture

All five services are running as of 24 Nov 2026 (Day 3 was run a day early). Each node records its container, host port and volume type.

```mermaid
flowchart TB
    C["curl / browser<br/>sends Host: whoami.localhost"]

    subgraph Host["Host — Linux Mint · every published port bound to 127.0.0.1"]
        subgraph Docker["Docker engine"]
            subgraph ProxyNet["network: proxy (172.18.0.0/16) — ingress"]
                T["traefik · traefik:v3.7<br/>127.0.0.1:8000 → :80 web entrypoint<br/>127.0.0.1:8080 → :8080 api + dashboard<br/>bind: /var/run/docker.sock (ro)<br/>restart: unless-stopped"]
                W["whoami · traefik/whoami<br/>no host port · :80 internal<br/>volumes: none<br/>restart: unless-stopped"]
            end
            subgraph MonNet["network: monitoring (172.19.0.0/16) — scrape + query"]
                P["prometheus · prom/prometheus:v3.16.0<br/>127.0.0.1:9090 → :9090<br/>bind: monitoring/prometheus.yml (ro)<br/>no data volume — TSDB lost on docker rm"]
                G["grafana · grafana/grafana:13.2.6<br/>127.0.0.1:3002 → :3000<br/>named volume: grafana-data"]
            end
            N["node-exporter · quay.io/prometheus/node-exporter:v1.13.1<br/>host network · :9100 on every host interface<br/>bind: / → /host (ro, rslave)"]
        end
        S[("/var/run/docker.sock")]
    end

    C -->|"HTTP :8000"| T
    T -->|"router whoami · rule Host(whoami.localhost)<br/>service whoami · lb port 80 → 172.18.0.2:80"| W
    S -.->|"read-only bind mount — Traefik reads labels via the Docker API"| T
    P -->|"scrape host.docker.internal:9100"| N
    G -->|"datasource http://prometheus:9090"| P

    classDef running fill:#e1f5fe,stroke:#0288d1
    classDef planned fill:#f5f5f5,stroke:#9e9e9e,stroke-dasharray: 5 5
    class T,W running
    class P,G,N running
```

## Service inventory

| Service | Container | Host bind | Volume type |
|---|---|---|---|
| Traefik | `traefik` | `127.0.0.1:8080→8080`, `127.0.0.1:8000→80` | bind `/var/run/docker.sock` (ro) — no data volume |
| whoami | `whoami` | none (internal `:80`) | none |
| node-exporter | `node-exporter` | none published — host net, `:9100` on every host interface | bind `/ → /host` (ro, rslave) |
| Prometheus | `prometheus` | `127.0.0.1:9090→9090` | bind `prometheus.yml` (ro); **no data volume — TSDB lost on `docker rm`** |
| Grafana | `grafana` | `127.0.0.1:3002→3000` | named volume `grafana-data` |

## The two edges that matter

- **Traefik → whoami** — the route is created by container labels, not by config: Traefik reads `traefik.enable=true`, the router rule `Host(whoami.localhost)` and the service port off the Docker API, so both containers must share the `proxy` network or the dial has nowhere to go (504).
- **Prometheus → node-exporter** — node-exporter runs on the host network, so it listens on `:9100` on every host interface; Prometheus reaches it at `host.docker.internal:9100`. A port published to `127.0.0.1` is *not* reachable this way — measured from inside a container, `172.17.0.1:7352` (listener on `0.0.0.0`) connects while `172.17.0.1:8000` (published to loopback only) is connection refused.
