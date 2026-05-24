# Phase A — Metrics & Event Pipeline: Beginner Guide

This guide walks you through deploying the monitoring stack that captures the exact telemetry needed to diagnose AP flapping — the condition where a client device (like an iPhone) bounces rapidly between access points on different channels (e.g., Channel 100 upstairs and Channel 36 downstairs) within a single minute.

The stack uses a two-tier approach so you can see both the signal drop profile (Prometheus) and the exact millisecond a roaming event fired (Loki), stacked in the same Grafana time window.

---

## What Each Tool Does (Plain English)

| Tool | Think of it as... | What it captures |
|------|-------------------|-----------------|
| **chhaley/omada-exporter** | A translator | Logs into the Omada Controller's `/api/v2` interface, reads Wi-Fi stats, and converts them to Prometheus metrics every 10 seconds |
| **Prometheus** | A time-stamped notebook | Stores every metric with a timestamp — 10-second resolution captures a client mid-flap |
| **Promtail** | A syslog post box | Receives UDP syslog from the Omada Controller on port 514 and forwards every line to Loki |
| **Loki** | A permanent log store | Stores Omada syslog events forever — bypassing the controller's ~7,650 entry internal log ceiling |
| **Grafana** | A diagnostic display | Renders the two-tier dashboard: RSSI drop profile on top, roaming event timeline below |

---

## Prerequisites

- [ ] A Linux server (or VM) on the same LAN as the Omada Controller
- [ ] Docker installed: `docker --version`
- [ ] Docker Compose installed: `docker compose version`
- [ ] You know the local IP of your Omada Controller (accessible at `https://<ip>:8043`)
- [ ] You can log into the Omada Controller web UI as an administrator

---

## Step 1 — Create a Read-Only Omada Account

The exporter needs credentials. Use a dedicated viewer account — if credentials ever leak, the account cannot change any network configuration.

1. Open `https://<controller-ip>:8043`
2. Go to **Settings → Administrators → Add New Administrator**
3. Username: `local_monitoring_user` (or any name you choose)
4. Password: something strong
5. Role: **Viewer**
6. Save

---

## Step 2 — Enable Syslog on the Omada Controller

The Omada Controller pushes real-time events — including roaming transitions — over UDP syslog. This is how Loki captures the millisecond a device actually roamed.

1. Go to **Settings → Log Settings → Remote Logging**
2. Enable **Remote Logging**
3. Set:
   - **Server IP:** IP address of your monitoring server
   - **Port:** `514`
   - **Protocol:** `UDP`
4. Under **Log Type**, enable **Client Info**
5. Save

> **Why port 514?**
> Port 514 is the standard syslog port — it is what the Omada Controller uses by default. The Docker Compose file maps host port 514 to port 1514 inside the Promtail container. You do not need to change anything in the Omada UI.

---

## Step 3 — Create the .env File

Sensitive credentials go in a `.env` file, not in `docker-compose.yml`. Create this file in the `phase-a-metrics/` directory:

```bash
# Omada Controller connection
OMADA_HOST=https://192.168.1.10:8043
OMADA_USER=local_monitoring_user
OMADA_PASS=your-strong-password-here
OMADA_SITE=Default

# Grafana admin password — change this before deploying
GF_ADMIN_PASSWORD=choose-a-strong-password
```

> **Important:** Never commit `.env` to git. Add it to `.gitignore` if you version this directory.

> **What is `OMADA_SITE`?**
> The site name shown in the top-left of the Omada Controller web UI. If you have never created multiple sites, it is `Default`.

---

## Step 4 — Check the Port 514 Privilege Requirement

On Linux, binding to any port below 1024 typically requires root. Docker itself runs as root by default, so `docker compose up` usually works without any changes. If you see a `permission denied` error on the Promtail container during startup:

```bash
# Option A: Run the compose command with sudo (only needed once, Docker is root after)
sudo docker compose up -d

# Option B: Lower the privileged port threshold permanently (survives reboots)
sudo sysctl -w net.ipv4.ip_unprivileged_port_start=514
echo "net.ipv4.ip_unprivileged_port_start=514" | sudo tee -a /etc/sysctl.conf

# Option C: Change the host port to 5514 (non-privileged) in docker-compose.yml
# Then update the Omada Controller syslog port to 5514 to match.
```

---

## Step 5 — Start the Stack

```bash
cd phase-a-metrics
docker compose up -d
```

First run downloads images — this takes 2-5 minutes. Subsequent starts are instant.

---

## Step 6 — Access Grafana via SSH Tunnel

Grafana is bound to `127.0.0.1:3000` — it is not directly reachable from your laptop. This is intentional: the dashboard is a diagnostic tool, not a public service.

**From your laptop, open an SSH tunnel:**

```bash
ssh -L 3000:127.0.0.1:3000 user@<server-ip>
```

Keep this terminal open, then navigate to `http://localhost:3000` in your browser.

- Username: `admin`
- Password: whatever you set in `GF_ADMIN_PASSWORD` in `.env` (default: `admin`)

> **Alternatively:** If you have Tailscale or WireGuard running, you can access Grafana directly at `http://<vpn-ip>:3000` — but only because the VPN overlay routes through the server's loopback, not because Grafana is exposed to the LAN.

---

## Step 7 — Verify the Stack

### All containers up

```bash
docker compose ps
```

Expected output — all five containers should show `Up`:

```
NAME              STATUS
omada-exporter    Up
prometheus        Up
loki              Up
promtail          Up
grafana           Up
```

### Prometheus is scraping the exporter

Open `http://localhost:9090/targets` via another SSH tunnel (`-L 9090:127.0.0.1:9090`) or from the server itself. The `omada` job should show **State: UP**.

### Loki is receiving syslog

In Grafana, click **Explore** → select the **Loki** datasource → run this query:

```
{job="omada-logs"}
```

If log lines appear, the syslog pipeline is working. If you see nothing, check the troubleshooting table below.

---

## Step 8 — Open the AP Flapping Dashboard

In Grafana, click **Dashboards → Omada → Wireless Client Roaming & Signal Health**.

The dashboard has two tiers:

### Tier 1 — RSSI Drop Profile (Prometheus)

The top panel shows signal strength in dBm for each wireless client, updated every 10 seconds. Use the **Device** dropdown at the top of the dashboard to focus on a specific device (e.g., your iPhone).

**Reading the RSSI chart:**

| RSSI | Signal quality | What it means |
|------|---------------|---------------|
| -30 to -55 dBm | Excellent | Device is very close to an AP |
| -55 to -70 dBm | Good | Normal operating range |
| -70 to -75 dBm | Fair | Starting to degrade |
| -75 to -80 dBm | Poor | Should have roamed already |
| Below -80 dBm | Very poor | Active packet loss likely |

**What AP flapping looks like:** The RSSI line drops sharply (client moving away from an AP), briefly recovers (roams to the other AP), then drops again — repeating within a short window. Combine with the roaming event panel below to confirm.

### Tier 2 — Roaming Event Timeline (Loki)

The middle panel shows a horizontal state timeline of every `is roaming` syslog event. Each bar represents a detected roaming transition from the Omada Controller.

**The diagnostic method:** Line up both panels to the same time window. If the RSSI drops below -75 dBm but no roaming event bar appears — the client is sticky. If roaming events fire but RSSI immediately drops back — the client is flapping back to the original AP.

**Grafana query powering this panel:**
```
{job="omada-logs"} |= "is roaming"
```

The bottom panel shows the full syslog stream. Search for a specific MAC address or AP name to narrow down events.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `omada-exporter` container restarts immediately | Wrong credentials or controller URL | Run `docker compose logs omada-exporter` — check for auth error. Verify `OMADA_HOST`, `OMADA_USER`, `OMADA_PASS` in `.env` |
| `promtail` fails with "permission denied on port 514" | Port 514 is privileged | See Step 4 — use `sudo docker compose up -d` or lower `ip_unprivileged_port_start` |
| No data in RSSI panel after 60 seconds | Exporter not being scraped | Visit `http://localhost:9090/targets` — the omada job must show State: UP |
| Loki query `{job="omada-logs"}` returns nothing | Controller not sending syslog | Confirm Remote Logging is enabled in Omada UI with the server IP and port 514. Run `docker compose logs promtail` and look for "listening on UDP :1514" |
| Grafana shows "Bad gateway" or unreachable | Wrong SSH tunnel or not running | Confirm the tunnel: `ssh -L 3000:127.0.0.1:3000 user@<server-ip>`. Check `docker compose ps` — grafana must be Up |
| Roaming timeline shows no bars despite drops | Omada Client Info logs not enabled | In Omada → Settings → Log Settings, confirm "Client Info" is checked under the Remote Logging section |

---

## File Reference

| File | Purpose |
|------|---------|
| `docker-compose.yml` | Defines all five containers: omada-exporter, prometheus, loki, promtail, grafana |
| `.env` | Your credentials (never commit to git) |
| `prometheus/prometheus.yml` | Scrape config: 10-second interval, port 9202 |
| `loki/promtail-config.yml` | Syslog receiver on port 1514 (mapped from host 514), `job="omada-logs"` label |
| `grafana/provisioning/datasources/datasources.yml` | Auto-connects Grafana to Prometheus and Loki |
| `grafana/provisioning/dashboards/omada-roaming.json` | The two-tier AP flapping dashboard |

---

## Next Steps

Once the metrics stack is working and you can see roaming events in the timeline, move on to the deep-packet forensics layer to diagnose application-level impact (jitter, retransmissions on Teams calls, DNS latency):
→ [`../phase-b-forensics/GUIDE.md`](../phase-b-forensics/GUIDE.md)
