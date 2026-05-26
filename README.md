# WiFi Business Network Monitoring System

Hey everyone, welcome to my network engineering portfolio project
I built this project to solve a real problem for small Internet Service Providers (WISPs) and WiFi cafe owners
Most monitoring tools out there are super expensive and complex for small businesses
So I decided to build a cheap, automated, and easy-to-use monitoring stack that runs locally using open-source tools

## What Is This Project For
The main goal of this project is to monitor a network's health in real-time
It tracks how much bandwidth is being consumed, the router's CPU and Memory usage, and system uptime
With this, a business owner can easily see if they are hitting their ISP bandwidth limit or if their router is overloaded
I wanted to make something production-grade but accessible for entry-level setups

---

## Network Topology

This entire project runs on a single Windows machine — no extra hardware needed at all
The MikroTik router is virtualized inside VirtualBox, and the monitoring stack runs inside Docker containers

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        WINDOWS HOST MACHINE                             │
│                                                                         │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │                  VirtualBox (Hypervisor)                        │   │
│   │                                                                  │   │
│   │    ┌─────────────────────────────────────────────┐             │   │
│   │    │         MikroTik CHR v7.23 (x86_64)         │             │   │
│   │    │                                              │             │   │
│   │    │   ether1 (WAN / NAT)    ether2 (Host-Only)  │             │   │
│   │    │   [Dynamic DHCP IP]     [192.168.56.103]     │             │   │
│   │    │                              │               │             │   │
│   │    │   SNMP Enabled (community: public)           │             │   │
│   │    │   Listening on UDP Port 161                  │             │   │
│   │    └─────────────────────────────────────────────┘             │   │
│   │                               │                                  │   │
│   │              VirtualBox Host-Only Network                        │   │
│   │              Subnet: 192.168.56.0/24                             │   │
│   │              Host Gateway: 192.168.56.1                          │   │
│   └───────────────────────────────┼──────────────────────────────────┘   │
│                                   │                                       │
│               SNMP UDP :161       │                                       │
│         ┌─────────────────────────┘                                       │
│         │                                                                 │
│   ┌─────▼──────────────────────────────────────────────────────────┐    │
│   │             Docker Desktop (WSL2 Backend)                        │    │
│   │                                                                  │    │
│   │   Internal Bridge Network: "monitoring" (172.18.0.0/16)         │    │
│   │                                                                  │    │
│   │  ┌───────────────────┐     ┌───────────────────┐               │    │
│   │  │   SNMP Exporter   │     │    Prometheus      │               │    │
│   │  │  prom/snmp-       │────▶│  prom/prometheus   │               │    │
│   │  │  exporter:latest  │     │  :latest           │               │    │
│   │  │                   │     │                    │               │    │
│   │  │  Container Port:  │     │  Container Port:   │               │    │
│   │  │     9116          │     │     9090           │               │    │
│   │  │  Host Port: 9116  │     │  Host Port: 9090   │               │    │
│   │  │                   │     │                    │               │    │
│   │  │  Translates SNMP  │     │  Scrapes Exporter  │               │    │
│   │  │  → Prometheus     │     │  every 30 seconds  │               │    │
│   │  │  metrics format   │     │  Stores in TSDB    │               │    │
│   │  └───────────────────┘     └────────┬───────────┘               │    │
│   │                                     │                            │    │
│   │                           PromQL queries                         │    │
│   │                                     │                            │    │
│   │                           ┌─────────▼──────────┐               │    │
│   │                           │      Grafana        │               │    │
│   │                           │  grafana/grafana    │               │    │
│   │                           │  :latest            │               │    │
│   │                           │                     │               │    │
│   │                           │  Container Port:    │               │    │
│   │                           │     3000            │               │    │
│   │                           │  Host Port: 3000    │               │    │
│   │                           │                     │               │    │
│   │                           │  Dark NOC Dashboard │               │    │
│   │                           │  9 Monitoring Panels│               │    │
│   │                           └─────────────────────┘               │    │
│   └──────────────────────────────────────────────────────────────────┘    │
│                                                                           │
│                         ┌────────────────────┐                           │
│                         │   Web Browser      │                           │
│                         │   localhost:3000   │  ◀── You open this        │
│                         └────────────────────┘                           │
└─────────────────────────────────────────────────────────────────────────┘
```

### Data Flow Explanation

Here is the exact journey of data from the router to your screen, step by step:

```
Step 1 — SNMP Polling
MikroTik CHR (192.168.56.103:161)
       │
       │  SNMP v2c GET/WALK (UDP)
       │  Community: public
       │  Pulls: ifHCInOctets, ifHCOutOctets, sysUpTime, hrProcessorLoad, etc.
       ▼
SNMP Exporter Container (snmp-exporter:9116)
       │
       │  Translates raw SNMP OID values into Prometheus metric format
       │  e.g: ifHCInOctets{ifDescr="ether1"} 1234567890
       ▼

Step 2 — Metric Scraping
Prometheus Container (prometheus:9090)
       │
       │  HTTP GET http://snmp-exporter:9116/snmp
       │       ?target=192.168.56.103
       │       &module=mikrotik
       │       &auth=public_v2
       │  Interval: every 30 seconds
       │  Stores timestamped metrics in TSDB (30-day retention)
       ▼

Step 3 — Visualization
Grafana Container (grafana:3000)
       │
       │  Runs PromQL queries against Prometheus
       │  e.g: rate(ifHCInOctets{ifDescr="ether1"}[5m]) * 8 / 1000000
       │  Renders live graphs in the NOC dashboard
       ▼
Your Browser → http://localhost:3000
```

### Component Summary Table

| Component | Role | Image | Host Port | IP/Address |
|---|---|---|---|---|
| MikroTik CHR v7.23 | Target Router (the device we monitor) | VirtualBox VM | — | `192.168.56.103` |
| SNMP Exporter | Translator (SNMP → Prometheus metrics) | `prom/snmp-exporter` | `9116` | `snmp-exporter:9116` |
| Prometheus | Time-Series Database (stores metrics) | `prom/prometheus` | `9090` | `prometheus:9090` |
| Grafana | Dashboard & Visualization Frontend | `grafana/grafana` | `3000` | `grafana:3000` |

---

## Detailed Network Topology Breakdown

To make this work without buying expensive hardware, I built a completely virtualized topology
Here is the detailed breakdown of how all the components are connected

- **The Router (MikroTik CHR v7.23 x86_64)**
  Running inside Oracle VirtualBox with two network adapters:
  - `ether1` — connected via NAT, gets a dynamic WAN IP from VirtualBox, this is the WAN uplink we monitor
  - `ether2` — connected via Host-Only adapter at IP `192.168.56.103`, this is the management interface
  SNMP is globally enabled with the default `public` community string, listening on UDP port 161

- **The Windows Host Machine**
  My physical laptop acts as the Docker host and the VirtualBox hypervisor at the same time
  The Host-Only network adapter creates a private subnet `192.168.56.0/24` between the host and the VM
  The host itself is reachable from inside the VM at `192.168.56.1` (the gateway)

- **Docker Desktop (WSL2 Backend)**
  All three monitoring services run as Docker containers on a dedicated internal bridge network named `monitoring`
  Containers talk to each other using their service names (DNS auto-resolves, e.g., `prometheus` → container IP)

- **SNMP Exporter (Port 9116)**
  This is the translator between the router's language (SNMP) and Prometheus's language (metrics)
  It fires SNMP GET/WALK requests to `192.168.56.103:161` whenever Prometheus asks it to scrape

- **Prometheus (Port 9090)**
  This is the time-series database engine
  Every 30 seconds, Prometheus sends a scrape request to the SNMP Exporter and stores the result with a timestamp
  Data is retained for 30 days, enough to see meaningful traffic trends

- **Grafana (Port 3000)**
  The beautiful frontend NOC-style dashboard
  It runs PromQL queries against Prometheus and renders them into 9 real-time panels
  The dark theme and color-coded thresholds (green/yellow/red) make it easy for anyone to read

---

## Live Testing: Normal Traffic vs Network Down Simulation

To prove that this monitoring stack is accurate and reactive, I ran two simulation tests directly on the router

### Scenario 1: Normal Traffic Generation

First, I needed to generate active traffic so the bandwidth graphs would actually show data
I used the MikroTik terminal to ping the host gateway `192.168.56.1` continuously with large packets
I configured the ping with a size of 1400 bytes to ensure it generates noticeable bandwidth
As you can see in the screenshot below, I successfully sent exactly **7742 requests** with 0% packet loss

```
/ping 192.168.56.1 size=1400 count=7742
```

![Normal Traffic Ping Test](normal-traffic-req.png)

Because of these 7742 ICMP requests flowing through `ether1`, the WAN bandwidth graph in Grafana spiked up
The dashboard accurately captured and displayed the active download and upload speeds in real-time
This validates that the entire SNMP → Prometheus → Grafana pipeline is working correctly end-to-end

### Scenario 2: Simulating a Network Outage

After confirming normal traffic was being monitored, I wanted to test how the system reacts to a real failure
To simulate a physical cable being unplugged or an ISP outage, I went back to the MikroTik console
I intentionally ran the command below to completely shut down the active interface:

```
/interface disable ether2
```

![Disabling Interface](disable%20ether2.png)

The reaction on the monitoring dashboard was instant
Since the interface was disabled, all traffic flow stopped and SNMP could no longer collect interface counters
In Grafana, the WAN bandwidth graph plummeted straight to zero, forming a steep vertical drop in the graph

![Network Down Traffic Graph](down%20%20traffic-req.png)

This proves that the SNMP Exporter successfully detects interface state changes in real-time
If this was a real WiFi business, the network owner would see this drop immediately on their Grafana screen
They would know exactly which interface went down without having to log into the router manually

---

## Tech Stack

| Technology | Version | Purpose |
|---|---|---|
| MikroTik RouterOS | v7.23 (x86_64 CHR) | The monitored network device |
| Oracle VirtualBox | Latest | Hypervisor to run MikroTik CHR |
| Docker Desktop | Latest (WSL2 backend) | Container orchestration on Windows |
| SNMP Exporter | `prom/snmp-exporter:latest` | SNMP-to-Prometheus translator |
| Prometheus | `prom/prometheus:latest` | Time-series metrics database |
| Grafana | `grafana/grafana:latest` | Dashboard and visualization |
| SNMP Protocol | v2c | Router data collection protocol |

---

## How To Run It Yourself

If you want to test my project on your own machine, just follow these steps

**Prerequisites:**
- Docker Desktop installed and running
- Oracle VirtualBox with a MikroTik CHR VM
- MikroTik CHR must have SNMP enabled (`/snmp set enabled=yes`)
- The VM must be reachable from the Docker host (Host-Only adapter recommended)

**Steps:**
1. Clone or download this repository to your local machine
2. Open `prometheus/prometheus.yml` and update this line to match your MikroTik's IP:
   ```yaml
   - "192.168.56.103"   # ← change this to your router's IP
   ```
3. Open a terminal inside the project folder
4. Run the stack:
   ```bash
   docker compose up -d
   ```
5. Wait about 30–60 seconds for all containers to fully start
6. Open your browser and go to `http://localhost:3000`
7. Log in with username `admin` and password `admin`
8. Navigate to **Dashboards → MikroTik Network Monitor**
9. The dashboard will auto-load with live data from your router

**To verify the stack is healthy:**
- SNMP Exporter health check: `http://localhost:9116`
- Prometheus target status: `http://localhost:9090/targets` (should show `UP`)
- Grafana: `http://localhost:3000`

---

## Project File Structure

```
wifi-monitor-project/
├── docker-compose.yml                          # Orchestrates all 3 containers
├── README.md                                   # You are reading this
│
├── prometheus/
│   └── prometheus.yml                          # Scrape config + relabeling rules
│
├── snmp-exporter/
│   └── snmp.yml                                # OID definitions for MikroTik
│
└── grafana/
    └── provisioning/
        ├── datasources/
        │   └── datasource.yml                  # Auto-connects Prometheus as data source
        └── dashboards/
            ├── dashboard.yml                   # Tells Grafana where to find dashboard files
            └── mikrotik.json                   # The 9-panel NOC dashboard definition
```

---

Thanks for checking out my project
If you have any questions, ideas, or run into issues, feel free to open an issue or reach out
