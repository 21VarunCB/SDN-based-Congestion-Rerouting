# Congestion Detection and Dynamic Traffic Rerouting in SDN (Project 27)

An automated Software-Defined Networking (SDN) and low-level UDP socket programming project implemented in **Mininet** and **POX**. The system continuously monitors link utilization across a multi-path diamond topology, detects primary path congestion in real time, and dynamically modifies OpenFlow rules to reroute affected UDP traffic onto an underutilized backup path.

---

## Network Architecture & Topology

The testbed consists of a 4-switch OpenFlow 1.0 diamond topology (`s1`–`s4`) and 3 hosts (`h1`, `h2`, `h3`):

```text
                       [ s2 ]  (10 Mbps, 2 ms/hop — Primary Path)
                      /      \
[ h1 ] (10.0.0.1) --- [ s1 ]  [ s4 ] --- [ h2 ] (10.0.0.2, UDP Receiver)
                      \      /
[ h3 ] (10.0.0.3) -----+ [ s3 ]  (10 Mbps, 5 ms/hop — Backup Path)
```

* **Primary Path (`s1 → s2 → s4`):** `10 Mbps` bandwidth, `2 ms` delay per hop (`4 ms` one-way), `50`-packet queue limit.
* **Backup Path (`s1 → s3 → s4`):** `10 Mbps` bandwidth, `5 ms` delay per hop (`10 ms` one-way), `50`-packet queue limit.
* **Traffic Flows:**
  * **Flow A (`h1 → h2:9999`):** `4 Mbps` UDP stream initially routed via the primary path (`s1` port `2`).
  * **Flow B (`h1 → h2:9998`):** `3 Mbps` UDP stream routed via the backup path (`s1` port `3`).
  * **Cross Traffic (`h3 → h2:9997`):** `8 Mbps` UDP congestion stream injected onto the primary path from `t = 15 s` to `t = 45 s`.

---

## Repository Structure

```text
├── README.md                  # Documentation and reproduction guide
├── Report.pdf                 # Complete project report with architecture, logs, and analysis
├── topology.py                # Mininet DiamondTopo definition (s1-s4, h1-h3)
├── congestion_controller.py   # POX OpenFlow controller (port/flow stats polling & rerouting)
├── udp_sender.py              # Paced UDP client embedding sequence numbers & timestamps
├── udp_receiver.py            # Non-blocking multi-port UDP server measuring loss & delay
├── run_experiment.py          # Automated 60-second static/adaptive experiment orchestrator
├── analyze.py                 # Computes comparison metrics and plots throughput/loss/delay
└── results/                   # Experimental CSV telemetry, logs, and comparison graph
    ├── comparison.csv
    ├── comparison.png
    ├── static_receiver.log
    ├── adaptive_receiver.log
    ├── controller_static.csv
    └── controller_adaptive.csv
```

---

## Prerequisites & Installation

Run the following commands on an **Ubuntu** machine or VM:

```bash
# 1. Update packages and install Mininet, Open vSwitch, iperf3, and Python dependencies
sudo apt update
sudo apt install -y git python3 python3-pip mininet openvswitch-switch iperf3 python3-pandas python3-matplotlib

# 2. Start Open vSwitch
sudo service openvswitch-switch start

# 3. Clone POX controller
cd ~
git clone [https://github.com/noxrepo/pox.git](https://github.com/noxrepo/pox.git)

# 4. Copy custom controller into POX
cp congestion_controller.py ~/pox/pox/
```

---

## How to Reproduce the Experiments

Open **two terminals** (`Terminal 1` for POX, `Terminal 2` for Mininet).

### 1. Run Static Routing Baseline
* **Terminal 1 (Start POX in static mode):**
  ```bash
  cd ~/pox
  python3 pox.py log.level --DEBUG congestion_controller --mode=static --logfile="$HOME/Downloads/files/results/controller_static.csv"
  ```
* **Terminal 2 (Run Mininet static experiment):**
  ```bash
  cd ~/Downloads/files
  sudo mn -c
  sudo python3 run_experiment.py --tag static
  ```
  *(Once Terminal 2 finishes after ~65 seconds, press `Ctrl+C` in Terminal 1 to stop POX.)*

### 2. Run Adaptive SDN Rerouting
* **Terminal 1 (Start POX in adaptive mode):**
  ```bash
  cd ~/pox
  python3 pox.py log.level --DEBUG congestion_controller --mode=adaptive --threshold=7.0 --capacity=10.0 --interval=1 --logfile="$HOME/Downloads/files/results/controller_adaptive.csv"
  ```
* **Terminal 2 (Run Mininet adaptive experiment):**
  ```bash
  cd ~/Downloads/files
  sudo mn -c
  sudo python3 run_experiment.py --tag adaptive
  ```
  *(Once Terminal 2 finishes, press `Ctrl+C` in Terminal 1.)*

### 3. Generate Comparison Metrics & Plot
In **Terminal 2**:
```bash
cd ~/Downloads/files
python3 analyze.py
xdg-open results/comparison.png
```

---

## Summary of Experimental Results

| Metric | Static Routing | Adaptive SDN Rerouting |
| :--- | :---: | :---: |
| **Baseline Flow A Throughput** | `3.90 Mbps` | `3.90 Mbps` |
| **Flow A Mean Throughput (During Congestion)** | `2.22 Mbps` | `3.67 Mbps` |
| **Flow A Packet Loss (During Congestion)** | `43.96%` | `6.75%` |
| **Flow A Mean Delay (During Congestion)** | `151.81 ms` | `55.57 ms` |
| **Flow A Overall Packet Loss (Full Run)** | `22.09%` | `4.84%` |
| **Congestion Detection Delay** | `never` | `2.02 s` |
| **Reroute Execution Delay** | `never` | `2.03 s` |
| **Throughput Recovery Time (>= 90% baseline)** | `never` | `6.01 s` |
