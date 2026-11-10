# 04 — Four-Week Plan

Each week ends with a **10-minute milestone check-in** (demo plus a short written update). M1–M3 earn points for being on time and complete. They are formative, not graded in depth.

---

## Week 1 — Design, 2-computer setup, leader election

**Reading:** Textbook Ch. 4, Ch. 5.1–5.4, Ch. 8.1–8.2 · Raft paper §1–5

### Tasks
- [ ] Repository, build, CI (lint plus unit tests), `Dockerfile` for the node.
- [ ] **Deployment checklist from [03-deployment-docker.md](03-deployment-docker.md) §6.** Do this on day 1–2. Network problems are the #1 schedule risk.
- [ ] Define the RPC interfaces (ticketing API plus Raft peer RPCs).
- [ ] Transport layer **with a fault-injection hook** (drop/delay/partition), used later by the automated tests.
- [ ] Raft roles, terms, `RequestVote`, randomized timeouts, heartbeats.
- [ ] Minimal `/status` endpoint (role, term) on every node.
- [ ] **Design document** (2–3 pages): architecture, node placement over the 2 computers and its quorum consequences, concurrency model, persistence plan, how hold expiry stays deterministic, invariants I1–I10, and the chosen extension(s).

### M1 deliverable
- Design document.
- Demo **on 2 computers**: a leader is elected. `docker kill` the leader and a new one appears within 3 s. Pull the cable of computer B: A keeps a leader and B's nodes stay candidates/followers. Reconnect, and the cluster converges to one leader.
- Automated test: at most one leader per term over 100 randomized runs (simulated network).

### Pitfalls
- Resetting the election timer at the wrong moments.
- Holding a lock during a blocking RPC.
- Election timeouts tuned on localhost and too small for the real LAN.

---

## Week 2 — Log replication, persistence, ticketing state machine

**Reading:** Textbook Ch. 7.1–7.3, 7.5, Ch. 8.3 · Raft §5.3–5.4, §8 · Herlihy & Wing §1–3

### Tasks
- [ ] AppendEntries: consistency check, conflict truncation, `nextIndex`/`matchIndex`.
- [ ] Commit rule (current-term restriction) and in-order apply.
- [ ] Persistence: metadata plus WAL with fsync and CRC on a Docker volume. Restart recovery.
- [ ] Ticketing state machine: `CreateEvent`, `Hold`, `HoldBestAvailable`, `Confirm`, `Release`, `GetSeatMap` (stale mode first). Leader-stamped time and `Tick` entries for expiry.
- [ ] Per-index state hash for checking that nodes don't diverge (I4).
- [ ] Tests: replicate 1,000 ops, crash/restart nodes, and check that all nodes have equal state hashes.

### M2 deliverable
- Demo on 2 computers: a simple script buys seats continuously while you `docker kill` the leader. No confirmed ticket is lost. After restart, all nodes report the same state hash and the same seat map.
- Half a page on the WAL format, crash recovery, and how hold expiry stays deterministic across the 2 machines' clocks.

### Pitfalls
- Committing previous-term entries by counting replicas (Raft Figure 8).
- Replying before fsync.
- Using `time.Now()` inside `apply()`.

---

## Week 3 — Exactly-once, linearizability, dashboard, bot army, extension

**Reading:** Raft §6–7 · Ongaro dissertation §6 (client interaction) · extension papers ([08-reading-list.md](08-reading-list.md))

### Tasks
- [ ] Client library: leader discovery across hosts, retries with the same `(clientId, seqNo)`.
- [ ] Dedup table in the state machine (I9).
- [ ] Linearizable reads (ReadIndex or through the log). Both `GetSeatMap` modes.
- [ ] **Bot army** container with strategies, abandon rate, and history recording.
- [ ] **Dashboard**: seat map plus cluster panel grouped by computer.
- [ ] **Auditor**: checks I5–I9 on the final state and the history, and runs the linearizability checker for I10.
- [ ] `scripts/chaos.sh` for seeded fault schedules on the 2-computer deployment.
- [ ] Implement **one extension** end to end, with tests.

### M3 deliverable
- Demo: the bot army (≥ 1,000 bots) rushes a 500-seat event while `chaos.sh` kills nodes and partitions the computers. The dashboard shows it happening, and afterwards the auditor prints `0 oversold · 0 double holds · 0 limit violations · history linearizable`.
- Extension demo.
- Draft evaluation plan: hypotheses, experiments, and metrics ([05](05-testing-and-evaluation.md) §3).

### Pitfalls
- Stale `linearizable` reads from a leader that was partitioned away (on computer B!).
- A client retry that creates a new seqNo, which leads to a double hold.
- A dedup table that is not deterministic, or not included in snapshots.

---

## Week 4 — Evaluation, hardening, "The Big Sale"

### Tasks
- [ ] Run the required experiments X1–X6 on the **2-computer deployment**, with ≥ 5 repetitions each.
- [ ] Run the automated chaos suite ≥ 200 times. Fix and analyze every failure.
- [ ] *(Top grade)* second extension, or polish of a ⭐ extension.
- [ ] Final report (max 8 pages).
- [ ] Clean up the repository: reproduction steps for the 2-computer setup, and scripts that regenerate every figure.
- [ ] Rehearse the demo **on the classroom network**. Bring your own switch or hotspot.

### M4 — "The Big Sale" (15 min per team)

| Minute | What |
|---|---|
| 0–3 | Architecture and node placement: why it survives losing B but not A |
| 3–5 | Sale opens: the bot army rushes 5,000 seats, and the dashboard lights up |
| 5–10 | **Chaos on stage**: kill the leader → failover. The instructor pulls computer B's cable → A keeps selling. Pull A's cable instead → sales stop safely. Reconnect → recovery. |
| 10–12 | Auditor results live: zero violations, linearizable |
| 12–15 | Key evaluation results plus extension highlight |

Then Q&A, in which every member answers.

*Optional class event:* several teams point their bot armies at **one** team's cluster at the same time, and the dashboard shows which team's bots got the front row.

---

## Suggested split for a team of 3

| Member | Primary | Secondary |
|---|---|---|
| A | Raft core (election, replication, commit, ReadIndex) | Review of B's state machine |
| B | Ticketing state machine, persistence, dedup, client library | Extension |
| C | Docker 2-computer deployment, chaos scripts, bot army, dashboard, auditor, experiments | Report figures |

Everyone must understand the Raft core and the quorum-placement argument. Either can be asked of anyone in the Q&A.
