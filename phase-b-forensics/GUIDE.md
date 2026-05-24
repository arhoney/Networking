# Phase B — Passive Forensics Engine: Beginner Guide

This guide walks you through setting up ntopng for deep-packet analysis of your network traffic. By the end, you will be able to identify exactly which applications are consuming bandwidth, see per-flow statistics, and diagnose issues like VoIP jitter or DNS latency on a per-device basis.

**Important:** Complete [Phase A](../phase-a-metrics/GUIDE.md) first. Phase B adds depth, but Phase A is what you will use every day for triage.

---

## What This Does (Plain English)

In Phase A, you monitor *signal quality* — how well your devices are connected to the Wi-Fi.

In Phase B, you monitor *what those devices are actually doing on the network* — every DNS lookup, every video stream, every retransmission.

The key technique is **port mirroring** (sometimes called SPAN — Switched Port Analyser). Your Omada switch is configured to silently copy every packet that passes through a chosen port and send that copy out of a dedicated "mirror" port. A server connected to the mirror port captures everything without interfering with any traffic — it is a completely passive observer.

```
Normal traffic flow:
  Device → AP → Switch → Router → Internet

What port mirroring adds:
  Device → AP → Switch → Router → Internet
                    │
                    └── (silent copy) → Mirror port → Your server (ntopng)
```

The server running ntopng never sends any packets on the capture interface. It is invisible to the network.

---

## What ntopng Shows You

| Feature | What it means for you |
|---------|----------------------|
| **Top Talkers** | Which devices are using the most bandwidth right now |
| **Flow Table** | Every active connection: source, destination, bytes, duration |
| **Application Detection** | "This traffic is Microsoft Teams" or "This is Netflix" |
| **Host Details** | All connections for a specific device, with RTT and jitter |
| **DNS Analysis** | Which domains are being resolved, and how long DNS takes |
| **Alert History** | Unusual traffic patterns like a device scanning the network |

---

## Prerequisites

Before you start, confirm all of the following:

- [ ] Phase A (Prometheus + Grafana) is running
- [ ] Your monitoring server has **two network interfaces**:
  - `eth0` (or similar): your management NIC — has an IP, used for SSH and Grafana
  - `eth1` (or similar): your passive capture NIC — will have **no IP address** assigned
- [ ] You know the name of the passive NIC. Run `ip link show` and look for the second interface
- [ ] A physical cable runs from the mirror port on your Omada 2.5G switch directly to the passive NIC on the server
- [ ] You have configured port mirroring on the switch (see [MIRROR-SETUP.md](MIRROR-SETUP.md))

---

## Step 1 — Find Your Passive NIC Name

Run this command on your monitoring server:

```bash
ip link show
```

You will see output like:

```
1: lo: <LOOPBACK,UP> ...
2: eth0: <BROADCAST,MULTICAST,UP> ...
   link/ether aa:bb:cc:dd:ee:01
3: eth1: <BROADCAST,MULTICAST> ...
   link/ether aa:bb:cc:dd:ee:02
```

The passive NIC is the one that is **not** currently `UP` with an assigned IP — it has no traffic on it yet. In this example it would be `eth1`.

Note this name down — you will need it in Step 3.

---

## Step 2 — Put the Passive NIC into Promiscuous Mode

Promiscuous mode means the NIC accepts all packets it sees, not just those addressed to itself. Without this, the NIC will silently discard the mirrored traffic.

Run these commands, replacing `eth1` with your actual NIC name:

```bash
# Bring the interface up (no IP assigned — just activates the hardware)
sudo ip link set eth1 up

# Enable promiscuous mode
sudo ip link set eth1 promisc on
```

Verify it worked:

```bash
ip link show eth1
```

You should see `PROMISC` and `UP` in the flags:

```
3: eth1: <BROADCAST,MULTICAST,PROMISC,UP> mtu 1500 ...
```

### Make this permanent across reboots

The commands above are temporary — they reset if the server reboots. To make them permanent, create a systemd service:

```bash
sudo tee /etc/systemd/system/promisc-eth1.service > /dev/null <<EOF
[Unit]
Description=Set eth1 to promiscuous mode for ntopng capture
After=network.target

[Service]
Type=oneshot
ExecStart=/sbin/ip link set eth1 up
ExecStart=/sbin/ip link set eth1 promisc on
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable promisc-eth1
```

> Replace `eth1` with your actual NIC name throughout.

---

## Step 3 — Set the Capture Interface in the Config

Open `ntopng/ntopng.conf` and find the line:

```
-i=eth1
```

Replace `eth1` with the name of your passive NIC from Step 1.

---

## Step 4 — Start ntopng

From the `phase-b-forensics/` directory:

```bash
docker compose up -d
```

---

## Step 5 — Access the ntopng Web Interface

ntopng's web interface is bound to `127.0.0.1` only — it cannot be reached directly from your laptop or any other device on the network. This is intentional.

To access it securely, create an SSH tunnel from your laptop:

```bash
ssh -L 3001:127.0.0.1:3001 user@<server-ip>
```

Then open a browser on your laptop and go to:

```
http://localhost:3001
```

- Default username: `admin`
- Default password: `admin`

Change the password immediately after first login.

> **What is an SSH tunnel?**
> The `-L 3001:127.0.0.1:3001` flag tells SSH: "Forward port 3001 on my laptop to port 3001 on localhost of the remote server." Your browser connects to your local port 3001, SSH encrypts the traffic and sends it through the secure connection, and the server delivers it to ntopng. No firewall rule needed. No exposure to the internet.

---

## Step 6 — What You Should See

Once the port mirror is active and ntopng is capturing:

1. The **Dashboard** page shows real-time throughput, top talkers, and recent alerts
2. Click **Flows** to see every active connection with source IP, destination IP, application type, and bytes transferred
3. Click **Hosts** and find your laptop or phone to see all of its active connections
4. Under **Applications**, you can see how much traffic is Teams, Netflix, DNS, etc.

### Diagnosing a Bad Teams Call

1. During the call, click **Hosts** and find your device
2. Click on the device, then click **Flows**
3. Filter by application **Microsoft Teams** or **STUN** (Teams uses STUN for voice)
4. Look at the **RTT** (round-trip time) column — values above 150ms will cause noticeable voice degradation
5. Look at the **Retransmissions** column — non-zero values mean packets are being lost

---

## Understanding the Security Model

The passive NIC has no IP address. This means:

- The server cannot send packets through that interface
- The interface does not respond to ARP requests
- Network scanners cannot detect or reach the server through the capture interface
- Even if ntopng has a vulnerability, an attacker on the network cannot reach it through the capture interface

The ntopng web interface is bound to `127.0.0.1` (localhost). This means:

- Only processes running on the same server can connect to port 3001
- Any device on the LAN attempting to connect to `<server-ip>:3001` will be refused
- Access requires an SSH tunnel, which means you need valid SSH credentials to the server

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| ntopng dashboard shows no traffic | Port mirror not configured or wrong NIC | Verify port mirror in Omada UI (see MIRROR-SETUP.md). Check that the passive NIC is `UP` and `PROMISC` (`ip link show eth1`). Confirm the cable goes from the mirror destination port to the passive NIC |
| ntopng web UI is unreachable | SSH tunnel not running | Start the tunnel: `ssh -L 3001:127.0.0.1:3001 user@<server-ip>` |
| "No data" in ntopng but interface is up | Wrong interface in ntopng.conf | Check `ntopng/ntopng.conf` — the `-i=` line must match the passive NIC name exactly |
| Container keeps restarting | Permission error accessing the NIC | The container needs host networking to access the NIC directly. Verify `network_mode: host` is set in `docker-compose.yml` |
| Traffic visible but no application labels | ntopng community edition limitations | Application detection via nDPI works for most protocols. For encrypted traffic, ntopng uses JA3/JA4 fingerprinting — this is expected behaviour |

---

## Resource Limits

Deep packet analysis is CPU and memory intensive. The `docker-compose.yml` caps ntopng at:

- **2 CPUs** maximum
- **2 GB RAM** maximum

This prevents a network broadcast storm or large file transfer from starving other containers on the server. If you see ntopng performing poorly, you can raise these limits in `docker-compose.yml` — but monitor the impact on the Grafana/Prometheus stack.

---

## File Reference

| File | Purpose |
|------|---------|
| `docker-compose.yml` | Runs the ntopng container with resource limits and host networking |
| `ntopng/ntopng.conf` | Configures the capture interface, port binding, and community edition settings |
| `MIRROR-SETUP.md` | Step-by-step instructions for configuring port mirroring in the Omada UI |
