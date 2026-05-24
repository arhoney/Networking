# Omada Network Monitoring & Forensics

A two-phase diagnostic stack that transforms a stock TP-Link Omada home network into a high-fidelity monitoring environment — without exposing anything to the public internet.

---

## What Problem Does This Solve?

Two specific visibility gaps that are nearly impossible to debug without proper tooling:

1. **Sticky clients & signal drops** — A device clings to a far access point instead of roaming to the nearer one. You see slowness but can't prove why or when it started.
2. **Application-layer issues** — A Microsoft Teams call stutters. Is it the ISP? A specific hop? DNS latency? Packet retransmissions? Without deep-packet analysis you're guessing.

This project addresses both.

---

## Architecture at a Glance

```
AT&T BGW210-700 (IP Passthrough)
        │ 1 Gbps WAN
        ▼
   Omada Router  ──────────────────────────────────┐
        │ 2.5 Gbps backhaul                        │
        ▼                                          │
   2.5G Switch                             Omada Controller
   ├── Port 1 → Router uplink              (metrics API :443)
   ├── Port 2 → 1G Switch → Upstairs AP          │
   ├── Port 3 → Downstairs AP                    │
   ├── Port 4 → Omada Controller          ◄───────┘
   └── Port X → Mirror destination NIC
                    │
                    ▼
             Monitoring Server (Docker)
             ├── Phase A: Prometheus + Loki + Grafana
             └── Phase B: ntopng (passive forensics)
```

### Data Flow

| Layer | Tool | What it captures |
|-------|------|-----------------|
| Metrics & timeseries | Prometheus + Grafana | RSSI, retry rates, bandwidth, roaming history |
| Log streaming | Grafana Loki | Millisecond-accurate roaming events from Omada syslog |
| Packet forensics | ntopng | Layer 7 apps, jitter, DNS RTT, retransmissions |

---

## Project Structure

```
Networking/
├── README.md                   ← you are here
├── Draft.md                    ← original high-level design document
│
├── phase-a-metrics/            ← Phase A: Metrics & Event Pipeline
│   ├── GUIDE.md                ← beginner-friendly setup walkthrough
│   ├── docker-compose.yml      ← runs Prometheus, Loki, and Grafana
│   ├── prometheus/
│   │   └── prometheus.yml      ← what to scrape and how often
│   ├── loki/
│   │   └── loki-config.yml     ← receives Omada syslog events
│   └── grafana/
│       └── provisioning/
│           ├── datasources/
│           │   └── datasources.yml     ← auto-connects Prometheus & Loki
│           └── dashboards/
│               ├── dashboards.yml      ← tells Grafana where to load dashboards
│               └── omada-roaming.json  ← the main RSSI + roaming dashboard
│
└── phase-b-forensics/          ← Phase B: Passive Packet Forensics
    ├── GUIDE.md                ← beginner-friendly setup walkthrough
    ├── docker-compose.yml      ← runs ntopng (bound to localhost only)
    ├── ntopng/
    │   └── ntopng.conf         ← NIC binding and security config
    └── MIRROR-SETUP.md         ← how to configure port mirroring in Omada UI
```

---

## Quick Start

### Prerequisites

- A Linux server (or VM) on the same LAN as the Omada Controller
- Docker and Docker Compose installed (`docker --version`, `docker compose version`)
- At least two network interface cards on the forensics server (one for management, one passive for Phase B)
- Access to the Omada Controller web UI

### Step 1 — Set up the metrics stack (Phase A)

Read [`phase-a-metrics/GUIDE.md`](phase-a-metrics/GUIDE.md) for the full walkthrough.

```bash
cd phase-a-metrics
docker compose up -d
```

Grafana opens at **http://\<server-ip\>:3000** (admin / admin on first login).

### Step 2 — Set up port mirroring and forensics (Phase B)

Read [`phase-b-forensics/MIRROR-SETUP.md`](phase-b-forensics/MIRROR-SETUP.md) first to configure your switch, then read [`phase-b-forensics/GUIDE.md`](phase-b-forensics/GUIDE.md) for the container setup.

```bash
cd phase-b-forensics
docker compose up -d
```

ntopng is accessible **only via SSH tunnel** (see the guide for the exact command).

---

## Security Model

| Rule | Reason |
|------|--------|
| No port forwarding on the gateway | Grafana, Prometheus, and ntopng are LAN-only services |
| Passive NIC has no IP address | The forensics interface is invisible to network scans |
| ntopng binds to `127.0.0.1` only | Cannot be reached from any other host without an SSH tunnel |
| Omada read-only API account | The exporter cannot change any network configuration |

Remote access is handled exclusively through a private VPN mesh (WireGuard or Tailscale) or an SSH tunnel — never by opening ports on the public router.

---

## Hardware Controller Notes

If you are running an Omada **hardware controller** (OC200, OC300) rather than the software controller, two things differ from the standard guide:

| Item | Software Controller | Hardware Controller |
|------|--------------------|--------------------|
| Management API port | `8043` | `443` |
| Firmware upgrade port | `8043` | `8043` (upgrade only, not API) |
| Auth endpoint | `/api/v2/login` on `:8043` | `/api/v2/login` on `:443` |

Set `OMADA_HOST=https://<controller-ip>:443` in `.env` — not `:8043`.

### NordVPN / VPN users

If NordVPN (or any VPN using a `nordlynx`/WireGuard interface) is active on the monitoring server, it will intercept LAN traffic and route it through the VPN tunnel, making the controller unreachable. Fix:

```bash
nordvpn set lan-discovery enabled
# or
nordvpn whitelist add subnet 192.168.0.0/24
```

Confirm routing is via your LAN interface, not the VPN:

```bash
ip route get <controller-ip>   # should show dev eth0/enp*, not dev nordlynx
```

---

## Docker Reference

### Useful commands

**Stack status and inspection:**
```bash
docker compose ps                          # status of all containers in this stack
docker compose ls                          # all active stacks and their compose file paths
docker inspect omada-exporter             # full config of one container
docker inspect omada-exporter | grep -i "source\|bind\|volume"  # storage only
```

**Logs and live monitoring:**
```bash
docker compose logs prometheus             # recent output from a container
docker compose logs omada-exporter --follow  # live log stream (Ctrl+C to exit)
docker stats                               # real-time CPU/memory for all containers
```

**Get a shell inside a container:**
```bash
docker exec -it prometheus sh
```

**Volumes (persistent data):**
```bash
docker volume ls                           # list all volumes
docker volume inspect phase-a-metrics_prometheus-data   # exact host path
sudo du -sh /var/lib/docker/volumes/phase-a-metrics_prometheus-data   # disk usage
```

---

### Where data is stored

All data is stored **locally on this machine** — nothing goes to the cloud.

There are two types of storage Docker uses here:

**Named volumes** — managed by Docker, live under `/var/lib/docker/volumes/`:

| Volume | Contents | Retention |
|--------|----------|-----------|
| `phase-a-metrics_prometheus-data` | All scraped metrics (RSSI, retry rates, client counts) | 90 days |
| `phase-a-metrics_loki-data` | All Omada syslog events | Permanent |
| `phase-a-metrics_grafana-data` | Dashboard state, users, settings | Permanent |

**Bind mounts** — regular files in this repo, mounted directly into the containers:

| File | Mounted into |
|------|-------------|
| `phase-a-metrics/prometheus/prometheus.yml` | Prometheus config |
| `phase-a-metrics/loki/promtail-config.yml` | Promtail syslog config |
| `phase-a-metrics/grafana/provisioning/` | Grafana datasources and dashboards |

Editing a bind-mounted file takes effect after restarting the relevant container (`docker compose restart prometheus`). Volumes persist across restarts and survive `docker compose down` — only `docker compose down -v` deletes them.

---

## Reference

- Original design document: [`Draft.md`](Draft.md)
- Phase A detailed guide: [`phase-a-metrics/GUIDE.md`](phase-a-metrics/GUIDE.md)
- Phase B detailed guide: [`phase-b-forensics/GUIDE.md`](phase-b-forensics/GUIDE.md)
- Port mirror setup: [`phase-b-forensics/MIRROR-SETUP.md`](phase-b-forensics/MIRROR-SETUP.md)
