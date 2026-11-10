# 03 — Deployment: Docker on 2 Computers

The final system **must** run on two physical computers. This document covers the topology, the networking options, quorum placement (the most important design question), fault injection, and the common practical problems.

---

## 1. Hardware and network setup

| Item | Recommendation |
|---|---|
| Computers | 2 laptops/PCs. **Linux is easiest.** macOS/Windows with Docker Desktop also work with Option B below. |
| Network | **Wired Ethernet through a small switch/router** (best), or a dedicated Wi-Fi hotspot. |
| ⚠ Campus Wi-Fi | Often has *client isolation*, which blocks laptop-to-laptop traffic. Test this in Week 1. Fall back to a phone hotspot or your own router. |
| IP addresses | Static, or DHCP reservations. Write them in `.env` (e.g., `HOST_A=192.168.1.10`, `HOST_B=192.168.1.20`). |
| Clocks | Turn on NTP on both machines, but **do not rely on synchronized clocks for correctness** (see spec §4.3). |

Measure first: run `ping` and an `iperf3` test between the computers, and record RTT (mean and p99) and bandwidth. You'll need these numbers to tune timeouts and to interpret your results.

---

## 2. Quorum placement — the central design question

Two computers mean **two failure domains**. Raft needs a strict majority of nodes to make progress.

| Placement | Lose computer A | Lose computer B | Partition A ⟷ B |
|---|---|---|---|
| 3 nodes: 2 (A) + 1 (B) | ❌ stops (1/3) | ✅ continues (2/3) | A side sells, B side stops |
| **5 nodes: 3 (A) + 2 (B)** (default) | ❌ stops (2/5) | ✅ continues (3/5) | A side sells, B side stops |
| 4 nodes: 2 (A) + 2 (B) | ❌ stops | ❌ stops | ❌ **both sides stop** |
| 5 nodes + witness on a 3rd device (E3) | ✅ | ✅ | the side with the witness continues |

Key insight that students must explain in the report: **with only two failure domains, no placement survives the loss of either domain.** Adding more nodes on the same machines does not help, and an even split is worse. The "fix" is a third failure domain (extension E3). This is also why real deployments use 3 availability zones.

In every case, **safety is preserved**. The minority side refuses to sell instead of overselling. That is CAP in practice: during a partition, TicketRush chooses C over A.

---

## 3. Networking options

You need Raft nodes on different computers to reach each other through a stable address. Pick **one** option and document it.

### Option A — Plain Docker Compose per computer (recommended baseline, works everywhere)

Each computer runs its own `docker compose` file. Containers **publish** their peer and client ports on the host. Peers are addressed as `HOST_IP:PORT`.

`deploy/compose.host-a.yml` (sketch):

```yaml
services:
  node1:
    image: ticketrush/node:latest
    command: ["--id=node1", "--listen=0.0.0.0:7001", "--advertise=${HOST_A}:7001",
              "--peers=node1=${HOST_A}:7001,node2=${HOST_A}:7002,node3=${HOST_A}:7003,node4=${HOST_B}:7004,node5=${HOST_B}:7005",
              "--data=/data"]
    ports: ["7001:7001"]
    volumes: ["node1-data:/data"]
    cap_add: ["NET_ADMIN"]        # allows tc/iptables fault injection inside the container
    restart: "no"                 # we want crashes to stay crashed during tests
  node2: { ... port 7002 ... }
  node3: { ... port 7003 ... }
  dashboard:
    image: ticketrush/dashboard:latest
    ports: ["8080:8080"]
    environment: ["NODES=${HOST_A}:7001,${HOST_A}:7002,${HOST_A}:7003,${HOST_B}:7004,${HOST_B}:7005"]
volumes: { node1-data: {}, node2-data: {}, node3-data: {} }
```

`deploy/compose.host-b.yml` defines `node4`, `node5`, `bot-army`, and `auditor` in the same way.

```bash
# on computer A
docker compose --env-file .env -f deploy/compose.host-a.yml up -d
# on computer B
docker compose --env-file .env -f deploy/compose.host-b.yml up -d
```

Watch out for the **advertise address**. A node must tell others (and clients, in `leaderHint`) the address that is reachable *from the other computer*, which is the host IP, not the container's internal IP.

### Option B — Docker Swarm with an overlay network (more elegant, Linux recommended)

```bash
# computer A
docker swarm init --advertise-addr 192.168.1.10
docker node update --label-add host=A <nodeA>
# computer B
docker swarm join --token <token> 192.168.1.10:2377
# computer A (manager)
docker node update --label-add host=B <nodeB>
docker stack deploy -c deploy/stack.yml ticketrush
```

`stack.yml` puts nodes on machines with `deploy.placement.constraints: [node.labels.host == A]`, and they find each other by service name (`node1:7001`) over the overlay network. That ties in nicely with Ch. 6 (Naming).

- Open ports: `2377/tcp`, `7946/tcp+udp`, `4789/udp`.
- Set `restart_policy: condition: none` for the Raft nodes, or Swarm will "help" by restarting killed nodes.
- Swarm itself uses Raft among its managers. In the report, explain why you must not *rely* on it, and what would happen to a Swarm with 2 managers.
- Overlay networking across hosts on Docker Desktop (Mac/Windows) can be unreliable. Use Option A if you hit problems.

---

## 4. Fault injection on real hardware

The automated test suite ([05](05-testing-and-evaluation.md) §2) uses an **in-process simulated network**, which makes it deterministic and fast. On the 2-computer deployment, faults are real:

| Fault | Command / action |
|---|---|
| Crash a node | `docker kill ticketrush-node2-1` |
| Restart a node | `docker start ticketrush-node2-1` (state comes from its volume) |
| Crash a whole computer | `docker compose -f deploy/compose.host-b.yml kill` |
| **Partition A ⟷ B** | Pull the Ethernet cable / turn off Wi-Fi on B · or `iptables -A INPUT -s $HOST_A -j DROP` on B |
| Isolate one node | Inside the container: `iptables -A INPUT -p tcp --dport 7002 -j DROP` (needs `NET_ADMIN`) |
| Add latency/loss on the LAN | Inside a container: `tc qdisc add dev eth0 root netem delay 100ms 30ms loss 5%` |
| Asymmetric partition | `iptables` on one side only (A→B works, B→A dropped) |
| Clock skew (E4) | Start a node with `--clock-offset=+800ms` (your clock abstraction) |

Provide a script, `scripts/chaos.sh`, that runs a random schedule of these faults from a seed and logs every action with a timestamp, so that results can be reproduced and plotted.

---

## 5. Measuring across two machines

- **Put the bot army and the history recorder on one machine** (default: B, or a third laptop if available). Linearizability checking needs one monotonic clock for invoke/return timestamps.
- If bots run on both computers, their clocks differ. Either check each machine's history separately, or widen every operation's interval by the measured clock-offset bound ε. Explain why widening is sound. (Ch. 5)
- When B is partitioned from A, bots on B can only reach the minority and must see failures, not wrong answers. Plot this.

---

## 6. Week-1 checklist for the deployment

- [ ] Both computers run Docker. The node image builds on both, or is pushed to a registry or `docker save`/`load`ed.
- [ ] `ping` and `iperf3` between the machines work. RTT and bandwidth are recorded.
- [ ] A "hello" container on A can reach one on B through the chosen option.
- [ ] Peer config and advertise addresses come from `.env`, with no hardcoded IPs in the code.
- [ ] `docker kill` + `docker start` keeps the data in the volume.
- [ ] A one-page `deploy/README.md` lets another team reproduce the setup.

## 7. Common problems

| Symptom | Likely cause |
|---|---|
| Nodes on A and B never see each other | Client isolation on Wi-Fi, host firewall (macOS/Windows Defender), wrong advertise address |
| Constant re-elections | Election timeout too close to the RTT/jitter of Wi-Fi. Measure and increase it. |
| `leaderHint` points at `172.17.x.x` | The node advertises its container IP instead of the host IP |
| Data lost after restart | `docker compose down -v` (deletes volumes), or the volume is not mounted at the data path |
| Killed node comes back by itself | `restart: always` or a Swarm restart policy |
| Mac/Windows performance much lower | Docker Desktop VM + file sharing. Use named volumes, not bind mounts, for `/data`. |
