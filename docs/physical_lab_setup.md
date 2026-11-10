# Mini-Shark Physical Lab Deployment & Demonstration Guide (8 to 50 Physical PCs)

This guide details how to deploy Mini-Shark in a physical computer lab across 8 to 50 student PCs, an intermediate Linux router laptop, and a central server hub.

---

## 1. Network Topology & Roles

```
[Student PC 1 (Probe 11)] ──┐
[Student PC 2 (Probe 12)] ──┼──> [Intermediate Linux Router] ──> [Server Hub (Probe 2)]
[Student PC 3 (Probe 13)] ──┤          (Probe 10)                 (:9000 E3 Telemetry)
[Student PC N (Probe N )] ──┘         (L3 Forwarding)             (:8001 Web UI + API)
          Link 1 (Ingress)                       Link 2 (Egress)
```

### Roles:
1. **Central Server Hub (1 PC or Laptop)**:
   * Runs the E3 correlation server (`:9000`), FastAPI backend (`:8001`), and React Wireshark UI (`:5173` or bundled).
   * Runs **Probe 2** (`--role server`) to monitor egress arrival.
2. **Intermediate Linux Router (1 Laptop)**:
   * Two network interfaces (e.g., `eth0` connected to Client switch, `eth1` connected to Server).
   * Pure Layer-3 IP forwarding enabled (`sysctl net.ipv4.ip_forward=1`).
   * Runs **Probe 10** (`--role router`) to detect packets in transit between Link 1 and Link 2.
3. **Student Client Machines (8 to 50 PCs)**:
   * Run the zero-install probe launcher: `./run_probe.sh --server <SERVER_IP>:9000`.
   * Automatically derives unique probe IDs from each PC's IP address (e.g., IP `192.168.1.42` becomes **Probe 42**).

---

## 2. Step-by-Step Setup Instructions

### Step 1: Start Central Server Hub
On the Server machine (e.g. IP `192.168.20.2`):
```bash
# 1. Start backend server (port 8001 for API, port 9000 for E3 telemetry)
./run.sh

# 2. Start local Probe 2 on the server
sudo python3 run_probe.py --server 127.0.0.1:9000 --role server
```

Open the dashboard in any browser at `http://localhost:5173` (or `http://<SERVER_IP>:5173`).

---

### Step 2: Configure Intermediate Router Laptop
On the Linux Router laptop:
```bash
# 1. Enable pure Layer-3 routing (NO NAT, preserving BLAKE2b hashes)
sudo ./scripts/setup_router.sh enable-routing

# 2. Start Router Probe (Probe 10) to observe transit packets
sudo ./run_probe.sh --server 192.168.20.2:9000 --role router
```

---

### Step 3: Start Student Client PCs
On each student PC (Linux or Windows):
```bash
# Linux / macOS:
sudo ./run_probe.sh --server 192.168.20.2:9000

# Windows:
run_probe.bat --server 192.168.20.2:9000
```
*Note: Students do NOT need to specify `--probe-id`. The probe automatically detects the host's IP address and assigns a unique probe ID without collisions.*

---

## 3. Running Real-World Workloads (Beyond Ping)

From any student client PC, generate multi-protocol workloads:

```bash
# 1. Mixed Real-World Workload (Concurrent HTTP REST + UDP streaming):
python3 scripts/traffic_generator.py --target 192.168.20.2 --mode mixed --rate 10

# 2. HTTP REST Queries only:
python3 scripts/traffic_generator.py --target 192.168.20.2 --mode http --rate 5

# 3. High-throughput UDP Media / Audio Streaming (1024B frames):
python3 scripts/traffic_generator.py --target 192.168.20.2 --mode udp-stream --rate 25
```

---

## 4. Live Demonstration: Silent Drop Localization

During evaluation, you can demonstrate hop-by-hop silent drop detection:

### Scenario A: Inject Drop on Link 1 (Client ──✖──> Router)
On the Router laptop:
```bash
sudo ./scripts/setup_router.sh drop-link1 20
```
* **Dashboard Output**: Packets are marked in red. The note clearly indicates:
  `Dropped on Link 1: Client PC (Probe 11) ──✖──> Router`
* **Evidence**: Probe 11 observed the packet, but Probe 10 (Router) never saw it.

---

### Scenario B: Inject Drop on Link 2 (Router ──✖──> Server)
On the Router laptop:
```bash
sudo ./scripts/setup_router.sh drop-link2 20
```
* **Dashboard Output**: Packets are marked in red. The note indicates:
  `Dropped on Link 2: Router (Probe 10) ──✖──> Server Hub (Probe 2)`
* **Evidence**: Probe 11 observed the packet, Probe 10 (Router) confirmed transit, but Probe 2 (Server) never received it.

---

### Scenario C: Clear All Drops
On the Router laptop:
```bash
sudo ./scripts/setup_router.sh clear-drops
```
* All packets immediately return to green `matched` status with sub-millisecond transit latency.
