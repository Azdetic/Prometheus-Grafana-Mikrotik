![MikroTik](https://img.shields.io/badge/MikroTik-293239?style=for-the-badge&logo=mikrotik&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?style=for-the-badge&logo=grafana&logoColor=white)
![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?style=for-the-badge&logo=prometheus&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![SNMP](https://img.shields.io/badge/SNMP-v2c-4B8BBE?style=for-the-badge&logo=cisco&logoColor=white)
![Network Monitoring](https://img.shields.io/badge/Network_Monitoring-open--source-22c55e?style=for-the-badge&logo=grafana&logoColor=white)

# WiFi Business Network Monitoring System

Hey everyone, welcome to my network engineering portfolio project
I built this because I noticed that most monitoring tools are either way too expensive or too complicated for small businesses like WiFi cafes or WISPs
So I decided to build my own monitoring stack using free open-source tools that anyone can run locally

## What Is This Project For

The main goal here is real-time network monitoring
It tracks bandwidth consumption, router CPU and memory usage, and uptime
The idea is simple: if you are a WiFi cafe owner and suddenly your internet feels slow, you should be able to open one dashboard and immediately see what is wrong
I wanted to build something that actually solves a real problem, not just a basic tutorial project

---

## Network Topology

This whole project runs on a single Windows machine, no extra hardware needed
The MikroTik router is virtualized inside VirtualBox, and the monitoring stack runs inside Docker containers

```
+-------------------------------------------------------------------------+
|                        WINDOWS HOST MACHINE                             |
|                                                                         |
|   +--------------------------------------------------------------------+|
|   |                  VirtualBox (Hypervisor)                           ||
|   |                                                                    ||
|   |    +---------------------------------------------------------+    ||
|   |    |           MikroTik CHR v7.23 (x86_64)                  |    ||
|   |    |                                                         |    ||
|   |    |   ether1 (WAN via NAT)        ether2 (Host-Only)        |    ||
|   |    |   [Dynamic DHCP IP]           [192.168.56.103]          |    ||
|   |    |                                       |                 |    ||
|   |    |   SNMP enabled, community: public     |                 |    ||
|   |    |   Listening on UDP port 161           |                 |    ||
|   |    +---------------------------------------------------------+    ||
|   |                                    |                              ||
|   |              VirtualBox Host-Only Network                         ||
|   |              Subnet: 192.168.56.0/24                              ||
|   |              Host Gateway: 192.168.56.1                           ||
|   +------------------------------------+-------------------------------+|
|                                        |                                |
|               SNMP UDP port 161        |                                |
|         +------------------------------+                                |
|         |                                                               |
|   +-----v--------------------------------------------------------------+|
|   |             Docker Desktop (WSL2 Backend)                          ||
|   |                                                                    ||
|   |   Internal Bridge Network: "monitoring" (172.18.0.0/16)           ||
|   |                                                                    ||
|   |  +-------------------+     +-------------------+                  ||
|   |  |   SNMP Exporter   |     |    Prometheus      |                  ||
|   |  |  prom/snmp-       +---->+  prom/prometheus   |                  ||
|   |  |  exporter:latest  |     |  :latest           |                  ||
|   |  |                   |     |                    |                  ||
|   |  |  Port: 9116       |     |  Port: 9090        |                  ||
|   |  |                   |     |                    |                  ||
|   |  |  Translates SNMP  |     |  Scrapes Exporter  |                  ||
|   |  |  to Prometheus    |     |  every 30 seconds  |                  ||
|   |  |  metrics format   |     |  Stores in TSDB    |                  ||
|   |  +-------------------+     +--------+-----------+                  ||
|   |                                     |                              ||
|   |                           PromQL queries                           ||
|   |                                     |                              ||
|   |                           +---------v----------+                  ||
|   |                           |      Grafana        |                  ||
|   |                           |  grafana/grafana    |                  ||
|   |                           |  :latest            |                  ||
|   |                           |                     |                  ||
|   |                           |  Port: 3000         |                  ||
|   |                           |                     |                  ||
|   |                           |  Dark NOC Dashboard |                  ||
|   |                           |  9 Monitoring Panels|                  ||
|   |                           +---------------------+                  ||
|   +--------------------------------------------------------------------+|
|                                                                         |
|                         +--------------------+                          |
|                         |   Web Browser      |                          |
|                         |   localhost:3000   |  <- you open this        |
|                         +--------------------+                          |
+-------------------------------------------------------------------------+
```

### How Data Actually Flows

Here is the exact path data takes from the router all the way to the browser screen:

```
Step 1 - SNMP Polling
MikroTik CHR (192.168.56.103:161)
       |
       |  SNMP v2c GET/WALK over UDP
       |  Community: public
       |  Pulls: ifHCInOctets, ifHCOutOctets, sysUpTime, hrProcessorLoad, etc.
       v
SNMP Exporter Container (snmp-exporter:9116)
       |
       |  Converts raw SNMP OID values into Prometheus metric format
       |  example: ifHCInOctets{ifDescr="ether1"} 1234567890
       v

Step 2 - Metric Scraping
Prometheus Container (prometheus:9090)
       |
       |  HTTP GET http://snmp-exporter:9116/snmp
       |       ?target=192.168.56.103
       |       &module=mikrotik
       |       &auth=public_v2
       |  Runs every 30 seconds
       |  Saves timestamped metrics in TSDB with 30-day retention
       v

Step 3 - Visualization
Grafana Container (grafana:3000)
       |
       |  Runs PromQL queries against Prometheus
       |  example: rate(ifHCInOctets{ifDescr="ether1"}[5m]) * 8 / 1000000
       |  Renders live graphs in the NOC dashboard
       v
Your Browser at http://localhost:3000
```

### Component Summary

| Component | Role | Image | Host Port | IP or Address |
|---|---|---|---|---|
| MikroTik CHR v7.23 | The router we are monitoring | VirtualBox VM | - | `192.168.56.103` |
| SNMP Exporter | Translates SNMP to Prometheus metrics | `prom/snmp-exporter` | `9116` | `snmp-exporter:9116` |
| Prometheus | Time-series database that stores metrics | `prom/prometheus` | `9090` | `prometheus:9090` |
| Grafana | Dashboard frontend for visualization | `grafana/grafana` | `3000` | `grafana:3000` |

---

## Detailed Breakdown of Each Component

To make this work without buying any hardware, I built a completely virtualized topology
Here is how each part connects together

- **The Router (MikroTik CHR v7.23 x86_64)**
  Running inside Oracle VirtualBox using two network adapters:
  - `ether1` - connected via NAT, gets a dynamic WAN IP from VirtualBox, this is the WAN uplink we are monitoring
  - `ether2` - connected via Host-Only adapter at `192.168.56.103`, this is how Docker containers reach the router
  SNMP is globally enabled using the default `public` community string on UDP port 161

- **The Windows Host Machine**
  My physical laptop runs both Docker Desktop and VirtualBox at the same time
  The Host-Only network creates a private subnet `192.168.56.0/24` between the host and the VM
  The host gateway is `192.168.56.1` so the router can ping back to the host machine

- **Docker Desktop running on WSL2**
  All three monitoring services run as containers on a dedicated bridge network called `monitoring`
  Containers talk to each other by service name, for example `prometheus` resolves to the Prometheus container's IP automatically

- **SNMP Exporter on port 9116**
  This is the translator between SNMP (the router's protocol) and Prometheus (the monitoring system's protocol)
  It sends SNMP GET and WALK requests to `192.168.56.103:161` every time Prometheus asks for a scrape

- **Prometheus on port 9090**
  The time-series database
  Every 30 seconds it calls the SNMP Exporter and saves the metrics with a timestamp
  Data is kept for 30 days which is more than enough to see traffic patterns

- **Grafana on port 3000**
  The dashboard frontend that makes everything visual
  It uses PromQL to query Prometheus and renders the results into 9 live panels
  Dark theme with color-coded thresholds so it is easy to read even at a glance

---

## Challenges Faced and Troubleshooting Journey

Building this environment locally was not smooth at all
Integrating VirtualBox with Docker on a Windows host with WSL2 backend introduced some nasty networking problems
Here are the three main issues I ran into and how I fixed them

**1 - The UDP and WSL2 Trap (connection refused)**

My first attempt used `host.docker.internal` with VirtualBox NAT Port Forwarding so the SNMP Exporter could reach the router
But Docker on WSL2 has a known issue where UDP packets (which SNMP uses) often get dropped or misrouted through that path
The exporter kept throwing `connection refused` even though the port forwarding rules were correct

- Fix: I dropped the NAT Port Forwarding idea completely and created a dedicated VirtualBox Host-Only Adapter instead
- This puts the MikroTik VM and the Windows host on the same `192.168.56.0/24` subnet so communication is direct with no middleman handling the UDP packets

**2 - Target String Misconfiguration in Prometheus (context deadline exceeded)**

After fixing the network route, Prometheus was still failing with `context deadline exceeded` on every scrape
I had set the scrape target as `192.168.56.103:161` thinking I needed to specify the SNMP port explicitly

- Fix: The SNMP Exporter already handles the port internally through the module configuration
- Having `:161` in the target string confused the exporter because it was being passed as part of the host address
- Removing it and leaving just `192.168.56.103` made the scrape succeed immediately

**3 - The VirtualBox Host-Only Adapter Going Silent (host unreachable)**

At one point the Windows host completely lost the ability to ping the router even though nothing changed in the config
The MikroTik VM was running, SNMP was enabled, but every ping timed out and Grafana showed no data

- Fix: The VirtualBox Host-Only network adapter on Windows had silently glitched and stopped passing traffic
- I went to Windows Network Connections (`ncpa.cpl`), disabled the VirtualBox Host-Only Ethernet Adapter, then re-enabled it
- The pings came back instantly and Grafana started populating data again

---

## Live Testing - Normal Traffic vs Network Down

I ran two tests to prove the stack actually works and responds correctly to real network events

### Test 1 - Generating Normal Traffic

To get the bandwidth graphs to show something useful, I needed to push real traffic through the router
I used the MikroTik terminal to send large ping packets to the host gateway at `192.168.56.1`
Each packet was 1400 bytes so the traffic would be noticeable on the graph
I ran a total of **7742 ping requests** and got 0% packet loss which confirms the connection was stable

```
/ping 192.168.56.1 size=1400 count=7742
```

![Normal Traffic Ping Test](normal-traffic-req.png)

After all those ICMP packets went through `ether1`, the bandwidth graph in Grafana immediately spiked up
The download and upload speeds were captured in real-time which confirms the full pipeline from SNMP to Grafana is working

### Test 2 - Simulating a Network Outage

After the normal traffic test, I wanted to see what happens when the network suddenly goes down
To simulate a real outage like a cable getting unplugged, I went to the MikroTik console and disabled the interface manually

```
/interface disable ether2
```

![Disabling Interface](disable%20ether2.png)

The dashboard reacted immediately
With the interface disabled, no more traffic could flow and the SNMP counters stopped updating
In Grafana the bandwidth graph dropped straight down to zero forming a steep vertical cliff

![Network Down Traffic Graph](down%20%20traffic-req.png)

This shows that the monitoring stack can detect a failure in real-time
If this was a real WiFi business setup, the owner would see that drop on their screen right away and know exactly which interface failed

---

## Tech Stack

| Technology | Version | What it does |
|---|---|---|
| MikroTik RouterOS | v7.23 x86_64 CHR | The network device being monitored |
| Oracle VirtualBox | Latest | Runs the MikroTik router as a VM |
| Docker Desktop | Latest with WSL2 backend | Runs the monitoring containers on Windows |
| SNMP Exporter | `prom/snmp-exporter:latest` | Translates SNMP data to Prometheus format |
| Prometheus | `prom/prometheus:latest` | Stores all the metrics with timestamps |
| Grafana | `grafana/grafana:latest` | Visualizes everything in a dashboard |
| SNMP Protocol | v2c | How we pull data from the router |

---

## How To Run It Yourself

If you want to try this on your own machine, here is what you need and how to set it up

**What you need first:**
- Docker Desktop installed and running
- Oracle VirtualBox with a MikroTik CHR VM set up
- SNMP must be enabled on the router: `/snmp set enabled=yes`
- The MikroTik VM needs to be reachable from the host using Host-Only adapter

**Steps to run it:**

1. Clone or download this repo to your machine
2. Open `prometheus/prometheus.yml` and change the target IP to match your router:
   ```yaml
   - "192.168.56.103"   # change this to your MikroTik's IP
   ```
3. Open a terminal inside the project folder
4. Start everything with:
   ```bash
   docker compose up -d
   ```
5. Wait around 30 to 60 seconds for all three containers to finish starting
6. Open `http://localhost:3000` in your browser
7. Login using `admin` for both username and password
8. Go to Dashboards and open MikroTik Network Monitor
9. You should see live graphs pulling data from your router

**Quick health checks:**
- SNMP Exporter: `http://localhost:9116`
- Prometheus targets: `http://localhost:9090/targets` - check that mikrotik_snmp shows UP
- Grafana: `http://localhost:3000`

---

## Project Files

```
wifi-monitor-project/
+-- docker-compose.yml            # Starts all 3 containers together
+-- README.md                     # This file
|
+-- prometheus/
|   +-- prometheus.yml            # Scrape config and relabeling rules
|
+-- snmp-exporter/
|   +-- snmp.yml                  # Defines which OIDs to pull from MikroTik
|
+-- grafana/
    +-- provisioning/
        +-- datasources/
        |   +-- datasource.yml    # Auto-connects Prometheus as data source on startup
        +-- dashboards/
            +-- dashboard.yml     # Tells Grafana where to find the dashboard files
            +-- mikrotik.json     # The actual 9-panel NOC dashboard
```

---

Thanks for reading through this
If anything is unclear or you run into problems, feel free to open an issue
