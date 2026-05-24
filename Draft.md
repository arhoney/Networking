

# **Network Monitoring & Forensics High-Level Design (HLD)**

## **1\. Executive Summary & Goals**

The objective of this design is to transition a stock TP-Link Omada residential network from a basic configuration topology into a high-fidelity diagnostic environment. The system must address two core visibility blind spots without introducing external security vulnerabilities:

* **Macro-Triage:** Correlate real-time device connection quality, RF signal degradation ($RSSI/SNR$), and wireless roaming behavior over a continuous historical timeline to fix "sticky clients" and intermittent connection drops.  
* **Micro-Forensics:** Enable deep-packet analysis and real-time application identification (e.g., diagnosing packet loss/jitter on Microsoft Teams calls) via passive monitoring.

## **2\. Existing Infrastructure & Topology**

Based on your network topology map and hardware inventory:

### **Upstream Gateway**

* **ISP Gateway:** AT\&T BGW210-700 configured in **IP Passthrough Mode** (allocating the public WAN IP directly to the downstream Omada Router).  
* **WAN Link:** Operating at $1000\\text{ Mbps}$ Full Duplex (1000FDX) to the router.

### **Core Architecture**

* **Omada Router:** Acts as the Layer 3 boundary.  
* **Inter-Switch Backhaul:** Connected via a high-speed $2.5\\text{ Gbps}$ Full Duplex (2500FDX) link (LAN2 on Router $\\leftrightarrow$ Port \#1 on the 2.5G Switch) to maximize internal LAN headroom.  
* **Core Distribution Node ("2.5 Switch"):** Serves as the central backbone connecting:  
  * **Omada Hardware Controller:** Connected to Port \#4 (1000FDX).  
  * **Downstairs Access Point (AP):** Connected to Port \#3 (1000FDX).  
  * **Edge Switch ("1g Switch"):** Connected via Port \#2 (1000FDX) to Port \#5 on the edge unit.  
* **Edge Distribution Node ("1g Switch"):** Feeds the **Upstairs Access Point (AP)** over Port \#1 (1000FDX).

## **3\. Available Diagnostic Toolset Matrix**

To build your telemetry pipeline, you have three distinct data aggregation tiers available:

| Telemetry Layer | Primary Software | Extraction Method | Diagnostic Value |
| :---- | :---- | :---- | :---- |
| **Metrics & Timeseries** | Prometheus & Grafana | Hybrid local-polling Prometheus Exporter | Tracks long-term rssi, tx\_retry\_rate, interface packet drops, and bandwidth trends. |
| **Instant Log Streaming** | Grafana Loki | Native Omada Controller Syslog Engine | Captures millisecond-accurate client roaming events (Device X roamed from Downstairs AP to Upstairs AP). |
| **Passive Packet Forensics** | ntopng (Dockerized) | Layer 2 Hardware Port Mirroring | Resolves Layer 7 application details (SIP/STUN jitter, DNS round-trip latency, packet retransmissions). |

## **4\. Implementation Design Architecture**

### **Phase A: The Metrics & Event Pipeline (Triage Layer)**

To monitor roaming paths alongside signal health over the course of the day, deploy a hybrid container stack.

1. **High-Frequency Scraping:** Configure your Prometheus exporter container to connect locally to the Omada Controller via its local web API (:8043) using a dedicated, local read-only account. This side-steps cloud rate-limiting, allowing safe 10-to-15-second metric updates.  
2. **Syslog Streaming:** Enable **Remote Logging** within the Omada Controller dashboard. Route **Client Info Logs** instantly to your Grafana Loki container to capture roaming transitions exactly when they occur.  
3. **Unified Visualization:** In Grafana, construct a dual-panel troubleshooting dashboard. Stack a **Time Series** panel tracking omada\_client\_rssi\_dbm vertically above a **State Timeline** panel mapping the active access point connection.

### **Phase B: The Passive Forensics Engine (Deep-Dive Layer)**

To track down why a specific client drops a video call or encounters high latency without altering the packet stream:

1. **Physical Sniffing Node:** Provision a server running Docker equipped with a secondary physical Network Interface Card (NIC) that has **no local IP address assigned** (Interface running strictly in Promiscuous Mode via ip link set \<interface\> promisc on).  
2. **Omada Mirror Configuration:** \* *To capture all internet traffic:* Configure Port \#1 on the **2.5 Switch** (the Router uplink) as a **Mirror Source**.  
   * *To capture a specific AP's traffic:* Configure Port \#3 (Downstairs AP) or Port \#2 (Upstairs Switch Uplink) as a **Mirror Source**.  
   * *Destination:* Map the cloned traffic out of a vacant port on the switch connected directly to your sniffing node's passive NIC.  
3. **ntopng Container Containment:** Deploy ntopng via Docker Compose, forcing its web interface to bind solely to local loopback (127.0.0.1:3000). Access the data securely via an encrypted SSH tunnel or private local VPN to ensure your raw packet forensics are protected from other network devices.

## **5\. Security & Risk Mitigation Guidelines**

* **Zero-Trust Capture Plane:** The network interface on the server receiving the mirrored port data must never have an IP address bound to it. This renders the host invisible to network scans and impervious to traditional Layer 3 attacks over that line.  
* **No Port Forwarding:** Under no circumstances should port forwarding be used on your AT\&T or Omada gateway to expose Grafana, Prometheus, or ntopng to the public WAN. Remote access must be handled exclusively via a private VPN mesh overlay (e.g., WireGuard or Tailscale).  
* **Resource Containment:** Because deep packet analysis is highly resource-intensive, cap the ntopng container limits inside your Docker configuration file (e.g., maximum 2 CPUs, 2GB RAM) to prevent a network broadcast event from starving adjacent infrastructure components.

