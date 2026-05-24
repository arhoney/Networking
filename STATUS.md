# Deployment Status

Last updated: 2026-05-24

---

## Phase A — Metrics & Event Pipeline

### Infrastructure

| Component | Status | Notes |
|-----------|--------|-------|
| Docker | Installed | v29.5.2, Docker Compose v5.1.4 |
| All 5 containers | Running | `docker compose ps` shows all Up |
| Prometheus | Up | Scraping at 10s interval, port 9090 |
| Loki | Up | Receiving syslog on port 3100 |
| Promtail | Up | Listening UDP 514 → forwarding to Loki |
| Grafana | Up | Accessible at `http://localhost:3000` |
| omada-exporter | Up | `chhaley/omada_exporter`, port 9202 |

### Omada Controller Connection

| Item | Value |
|------|-------|
| Controller type | Hardware (OC200/OC300, type=17) |
| Firmware | 6.2.10.18 |
| Management API port | **443** (not 8043 — see note below) |
| Auth method | Username/password via `/api/v2/login` |
| Exporter account | `network` (Local User, Viewer role, All Sites) |
| Site name | `home` |
| Syslog | Enabled → 192.168.0.69:514 |

> **Port 8043 is the firmware upgrade port only.** The management API (`/api/v2/login`, all data endpoints) runs on port 443 on hardware controllers. The GUIDE.md and original docker-compose.yml referenced 8043 — this has been corrected.

### Dashboards

| Dashboard | Source | Status |
|-----------|--------|--------|
| Wireless Client Roaming & Signal Health | Original (charlie-haley metrics) | Pending data confirmation |
| Access Point | StanislawHorna/omada-exporter-go | Provisioned |
| Site Overview | StanislawHorna/omada-exporter-go | Provisioned |

### Known Issues & Resolved

| Issue | Resolution |
|-------|-----------|
| `chhaley/omada-exporter` image not found | Image uses underscore: `chhaley/omada_exporter` |
| Controller unreachable (100% packet loss) | NordVPN was routing LAN traffic through VPN tunnel. Fixed: `nordvpn set lan-discovery enabled` |
| Auth returning 500 on all endpoints | Wrong port — hardware controller uses 443, not 8043 |
| Docker permission denied | User added to `docker` group; activate with `newgrp docker` |
| Grafana password not accepted | Grafana persists password to volume on first start. Fix: `docker volume rm phase-a-metrics_grafana-data` then restart |

---

## Outstanding Items

- [ ] **Confirm per-client RSSI data in Grafana** — charlie-haley exporter is running on port 443; verify the "Wireless Client Roaming & Signal Health" dashboard is populating
- [ ] **Confirm Loki syslog data** — Promtail is listening; trigger a roaming event and verify `{job="omada-logs"}` returns results in Grafana Explore
- [ ] **Enable Client Info syslog** — In Omada → Settings → Log Settings → Advanced, confirm "Client Info" events are included in Remote Logging output (required for roaming event timeline)
- [ ] **Phase B — Passive Packet Forensics** — ntopng setup not yet started

---

## Environment

| Item | Value |
|------|-------|
| Monitoring server | `pop-os` at `192.168.0.69` |
| OS | Pop!_OS 24.04 LTS |
| Omada Controller IP | `192.168.0.101` (static) |
| Upstairs AP | EAP770(US) v2.0 at `192.168.0.100` |
| Downstairs AP | EAP245(US) v3.0 at `192.168.0.109` |
| Gateway | ER707-M2 v1.30 at `192.168.0.1` |
