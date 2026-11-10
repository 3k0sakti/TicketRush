# 07 — Final Report Template

**Format:** PDF, IEEE or ACM two-column template, **max 8 pages** excluding references and appendix.
**File name:** `TicketRush-report-<team-name>.pdf`

---

## Title
*TicketRush: [subtitle, e.g., "Oversell-Free Flash Sales on a Two-Host Raft Cluster"]*
Authors, student IDs, emails.

## Abstract (≤ 200 words)
Problem, approach, key results with numbers (e.g., failover pause, throughput, 0 violations in N chaos runs), and the main conclusion.

## 1. Introduction (~0.5 page)
- Why flash sales are hard: contention, spikes, strict invariants.
- What you built, and your contributions as bullets.
- The main findings.

## 2. Background and system model (~0.75 page)
- Failure model: crash-recovery, asynchronous network, stable storage (Ch. 8).
- Consistency models: linearizability vs. sequential vs. eventual (Ch. 7), and which ticketing operations need which.
- Raft in brief. Do not restate the paper.

## 3. Design (~1.5 pages)
- Architecture and node placement on the 2 computers, with the **quorum and failure-domain analysis**.
- Ticketing state machine and invariants I5–I9.
- **Time:** deterministic hold expiry with leader-stamped time and `Tick` entries.
- Client protocol: leader discovery across hosts, retries, exactly-once.
- Read modes: linearizable vs. stale.
- Deployment: networking option, advertise addresses, volumes.
- **Design decisions table:** decision | alternatives | rationale.

## 4. Extension(s) (~1 page)

## 5. Correctness (~1 page)
- How each invariant I1–I10 is ensured, asserted, and audited.
- Fault-injection methodology (simulated network and real 2-host chaos).
- Chaos results: number of runs, pass rate, analysis of every failure (bug "war stories" are welcome).
- Mutation test: the bug you introduced and how it was caught.

## 6. Evaluation (~2 pages)
- Setup: both computers' hardware and OS, Docker version, network type, **measured RTT and bandwidth**, parameters, repetitions.
- X1–X6: **hypothesis → figure → interpretation**, tied to theory.

## 7. Discussion and limitations (~0.5 page)
- What two failure domains can and cannot tolerate.
- Threats to validity: two hosts only, LAN instead of WAN, synthetic bots.
- Unbounded dedup table, Byzantine faults, and what you would do differently.

## 8. Related work (~0.25 page)
Paxos/Multi-Paxos, Viewstamped Replication, ZooKeeper/ZAB, chain replication, and production systems (etcd, CockroachDB, TiKV). How real ticketing and flash-sale systems handle this (queues, inventory sharding).

## 9. Conclusion (~0.25 page)

## References

## Appendix (not counted toward the page limit)
- A. Reproduction: build, `make test`, 2-computer deployment commands, `chaos.sh`, figure scripts.
- B. **AI-assistance disclosure:** tools, what for, and how you checked the output.
- C. Individual contributions.
