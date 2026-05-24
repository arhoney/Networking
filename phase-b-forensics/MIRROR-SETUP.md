# Omada Switch Port Mirroring — Setup Guide

Port mirroring (also called SPAN — Switched Port Analyser) tells your Omada switch to send a copy of all traffic passing through a chosen port to a separate "mirror destination" port. Your monitoring server connects to that destination port and captures everything silently.

This guide covers three mirroring scenarios so you can choose the right one for your diagnostic goal.

---

## Before You Start

You will need:
- Access to the Omada Controller web UI (`https://<controller-ip>:8043`)
- A spare port on the 2.5G switch (the mirror destination)
- A physical cable from that spare port to the passive NIC on your monitoring server
- The passive NIC must have **no IP address** (see the [Phase B Guide](GUIDE.md) for how to verify this)

---

## Understanding Your Switch Port Layout

Based on the network design, your 2.5G switch ports are:

| Port | Connected to | Speed |
|------|-------------|-------|
| Port 1 | Omada Router (WAN uplink) | 2.5 Gbps |
| Port 2 | 1G Switch (Upstairs AP uplink) | 1 Gbps |
| Port 3 | Downstairs Access Point | 1 Gbps |
| Port 4 | Omada Controller | 1 Gbps |
| Port 5+ | Available for mirror destination | varies |

---

## Choosing What to Mirror

| Diagnostic Goal | Mirror this port | What you will capture |
|----------------|-----------------|----------------------|
| All internet traffic (see everything to/from the WAN) | Port 1 (Router uplink) | Every packet entering or leaving your home network |
| Downstairs AP traffic only | Port 3 (Downstairs AP) | All Wi-Fi clients connected to the downstairs AP |
| Upstairs AP traffic only | Port 2 (1G Switch uplink) | All Wi-Fi clients connected to the upstairs AP |

> **Tip for diagnosing a specific device:** If you know which AP the device is connected to (check the Phase A Grafana dashboard), mirror just that AP's port. This reduces the volume of traffic ntopng has to process and makes the flows table easier to read.

---

## Step-by-Step: Configuring Port Mirroring in the Omada UI

### Step 1 — Log into the Omada Controller

Open your browser and navigate to:
```
https://<controller-ip>:8043
```

Log in with your administrator credentials.

### Step 2 — Navigate to the Switch Settings

1. In the left sidebar, click **Devices**
2. Find your **2.5G Switch** in the device list and click on it
3. A panel will slide in from the right showing the device details
4. At the top of that panel, click the **Settings** tab (gear icon)

### Step 3 — Open Port Mirroring

1. In the switch settings panel, scroll down to find **Port Mirroring**
2. Click **Port Mirroring** to expand it
3. Click **+ Add** or the **Create** button to start a new mirroring rule

### Step 4 — Configure the Mirror Rule

You will see a form with the following fields:

**Session** (or Rule Name):
- Enter a descriptive name such as `wan-monitor` or `downstairs-ap-monitor`

**Mirroring Mode:**
- Select **Both** (captures both incoming and outgoing traffic on the source port)
- *Ingress* only captures traffic arriving at the port; *Egress* only captures traffic leaving. For forensics you want both directions.

**Source Port(s):**
- Select the port(s) you want to copy traffic from.
- Example: Select **Port 1** to capture all internet traffic.
- You can select multiple source ports if needed.

**Destination Port:**
- Select the port connected to your monitoring server's passive NIC.
- Example: Select **Port 5** if your monitoring server is plugged into port 5.

> **Important:** The destination port is now exclusively a mirror output. It can no longer carry normal traffic — do not use it for anything else while mirroring is active.

### Step 5 — Apply the Configuration

1. Click **Apply** or **Save**
2. The controller will push the configuration to the switch (takes a few seconds)
3. You will see a confirmation message when complete

### Step 6 — Verify the Mirror is Working

On your monitoring server, run:

```bash
sudo tcpdump -i eth1 -c 20 --immediate-mode
```

Replace `eth1` with your passive NIC name. If port mirroring is working, you should immediately see packets scrolling by:

```
12:34:56.789012 IP 192.168.1.105.443 > 93.184.216.34.5678: ...
12:34:56.789045 IP 93.184.216.34.5678 > 192.168.1.105.443: ...
```

If you see `0 packets captured` after 5 seconds, the mirror is not active. Check the troubleshooting section below.

Press `Ctrl+C` to stop tcpdump when you are satisfied.

---

## Switching Between Mirror Targets

You can change which port is being mirrored at any time without affecting network traffic on other ports. Return to **Port Mirroring** in the switch settings, click the existing rule, change the source port(s), and click Apply.

Common scenario: You notice a device is struggling on the upstairs AP (from the Phase A Grafana dashboard). Switch the mirror source from Port 1 (all internet) to Port 2 (upstairs switch uplink) to see only that AP's traffic in ntopng.

---

## Disabling Port Mirroring

When you are done with a forensic investigation and no longer need the mirror:

1. Go to **Port Mirroring** in the switch settings
2. Click the rule you created
3. Click **Delete** or toggle it off

This frees the destination port for normal use and reduces the processing load on ntopng.

---

## How Much Traffic Will Be Mirrored?

| Mirror Source | Expected traffic volume |
|--------------|------------------------|
| Port 1 (Router uplink) | Up to 1 Gbps (your full WAN speed) — ntopng handles this easily |
| Port 3 (Downstairs AP) | Up to 1 Gbps (per-AP traffic only) |
| Port 2 (Upstairs switch uplink) | Up to 1 Gbps |

For a typical home network at rest, you will see 1–50 Mbps of traffic. During a 4K Netflix stream, expect 15–25 Mbps per stream. During a large file transfer, up to the WAN line speed.

The ntopng container has resource limits set in `docker-compose.yml` (2 CPU / 2 GB RAM) to prevent it from overwhelming the server even at peak traffic.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `tcpdump` shows 0 packets | Port mirror not applied or wrong destination port | Return to Omada UI → Port Mirroring and verify the rule shows the correct source and destination ports. Check the cable between the switch mirror port and the server passive NIC |
| `tcpdump` shows packets but ntopng shows nothing | Wrong interface in ntopng.conf | Check `ntopng/ntopng.conf` — the `-i=` line must exactly match the NIC name shown in `ip link show` |
| ntopng shows traffic but no application names | Traffic is encrypted or protocol is uncommon | This is expected for HTTPS/TLS traffic. ntopng uses JA3 fingerprinting for TLS — the **Application** column will show "TLS" with a detected category (e.g., "Web Browsing") rather than the specific site name |
| Omada UI does not show Port Mirroring option | Switch firmware version | Ensure the 2.5G switch is running the latest firmware. Port mirroring is available on all recent Omada-managed switch firmware versions. Update via **Devices → Switch → Upgrade** |
| Source port and destination port are the same | Configuration error | The destination port must be a different physical port than any source port. Use a spare port for the destination |

---

## Network Diagram with Mirror Active

```
2.5G Switch
┌─────────────────────────────────────────────┐
│                                             │
│  Port 1 ──── Router uplink (WAN traffic)   │
│      │                                      │
│      │ (mirrored copy)                      │
│      ▼                                      │
│  Port 5 ──── Monitoring server (passive NIC)│
│                                             │
│  Port 2 ──── 1G Switch ──── Upstairs AP    │
│  Port 3 ──── Downstairs AP                 │
│  Port 4 ──── Omada Controller              │
│                                             │
└─────────────────────────────────────────────┘
```

The mirrored copy is a read-only clone of the traffic. The original packets continue to their destinations completely unaffected.
