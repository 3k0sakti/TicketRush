# 01 — Project Brief

## 1. Context

Flash sales (concert tickets, limited sneakers, train tickets during holidays) are among the hardest workloads in practice. Contention for the same items is extreme, the traffic spike is huge, and the correctness requirements are strict. Selling one seat twice means a public scandal and a refund. Charging one fan twice means a complaint and lost trust.

The tempting shortcut is a single database server, which becomes a single point of failure at exactly the worst moment. The other tempting shortcut is eventually consistent replicas, which oversell. In this project you will build the alternative: a **strongly consistent, replicated ticketing core using Raft**, deployed across **two physical computers** so that failures are real.

## 2. Learning objectives

After the project, each student can:

1. **Implement and justify** a crash-fault-tolerant consensus protocol (Raft), including its safety properties.
2. **Map application invariants to consistency models.** Explain why "buy seat" needs linearizability while "show seat map" can tolerate staleness, and measure what each choice costs.
3. **Design exactly-once client interaction** over an unreliable network (retries, idempotency, deduplication).
4. **Reason about failure domains and quorums.** Explain why 2 machines cannot tolerate the loss of *either* machine, and design the node placement on purpose.
5. **Handle time correctly in a replicated system.** Implement seat-hold expiry deterministically, without trusting the local clocks of different machines.
6. **Deploy and operate** a multi-host containerized system, and inject real faults (crash, partition, latency).
7. **Evaluate experimentally**: hypotheses, controlled fault injection, repeated runs, and honest statistics.

## 3. Problem statement

Build **TicketRush**: a cluster of `N` Raft nodes (`N = 3` or `5`) spread over **two computers**. Together the nodes provide a ticketing service for one or more events. Each event has sections, rows, and seats.

A fan's purchase is a **two-step flow**, as on real ticketing sites:

```
1. Hold(seats)      -> holdId, expiresAt     // seats reserved for e.g. 60 s
2. Confirm(holdId)  -> tickets               // seats become SOLD
   (or the hold expires / is released -> seats become AVAILABLE again)
```

Business rules, which become your **safety invariants**:

- A seat is in exactly one state: `AVAILABLE`, `HELD(by hold h)`, or `SOLD(to user u)`.
- A seat is **never sold twice** and **never held by two holds at once**.
- A fan can own or hold at most **4 seats per event**.
- Holding several seats is **all-or-nothing**. Either all requested seats are held, or none.
- A retried request (network timeout) **never** creates a second hold or a second purchase.
- `available + held + sold = capacity` at all times.

## 4. Requirements

### 4.1 Core functional requirements (all mandatory)

| ID | Requirement |
|---|---|
| F1 | **Raft leader election**: randomized timeouts and terms. Works across the 2 computers. |
| F2 | **Raft log replication**: AppendEntries consistency check and conflict resolution. |
| F3 | **Commit rule**: majority replication, and only entries from the current term are committed by counting replicas (Raft §5.4.2). |
| F4 | **Persistence**: term, vote, and log are fsync'd to a Docker volume before the node replies. Containers can be killed and restarted without losing data. |
| F5 | **Ticketing state machine**: `CreateEvent`, `Hold`, `HoldBestAvailable`, `Confirm`, `Release`, `GetSeatMap`, `GetMyTickets`, with all business rules in §3. |
| F6 | **Exactly-once**: every mutating request carries `(clientId, seqNo)`. Duplicates return the cached result. |
| F7 | **Linearizable buys and seat reads**: `Hold`/`Confirm` are linearizable. `GetSeatMap` supports **two modes**: `linearizable` (ReadIndex or through the log) and `stale` (served locally by any node). |
| F8 | **Deterministic hold expiry**: expiry must **not** depend on the local clock of whichever node applies the entry (see spec §4.3). |
| F9 | **Live dashboard**: a web UI that shows the seat map, plus each node's role, term, commitIndex, and host (computer A or B). |
| F10 | **Bot army**: a containerized load generator that simulates many concurrent fans with configurable behavior (target popular seats, give up, retry, abandon holds). |
| F11 | **2-computer Docker deployment**: one documented command per computer starts its part of the cluster (see [03-deployment-docker.md](03-deployment-docker.md)). |

### 4.2 Extensions (implement **at least one**; two or one "hard" ⭐ for a top grade)

| ID | Extension | Textbook link |
|---|---|---|
| E1 | **Snapshots / log compaction** with `InstallSnapshot`. Needed in practice, because a big sale produces a huge log. | Ch. 8 (checkpointing, recovery) |
| E2 ⭐ | **Sharding by event/section**: several Raft groups, consistent hashing, and **two-phase commit** for a fan who buys seats in two sections owned by different groups, while the 4-ticket limit holds globally. | Ch. 6, Ch. 8.5 (distributed commit) |
| E3 ⭐ | **Third failure domain**: dynamic membership change that adds a *witness* node on a third device (Raspberry Pi, cloud VM, or a classmate's laptop), so the cluster survives the loss of *either* main computer. Replace a dead node while the sale is running. | Ch. 8 (group membership) |
| E4 | **Leader leases** for fast linearizable seat-map reads, with an analysis of clock drift between the 2 real computers. | Ch. 5 (physical clocks) |
| E5 | **Security**: mTLS between hosts, signed buyer tokens, and an anti-bot rate limit that is *replicated* (enforced consistently across failover). Includes a threat model. | Ch. 9 |
| E6 | **Fair virtual waiting room**: fans join a replicated FIFO queue and are admitted in batches. Measure fairness (admission order vs. arrival order) under failover. | Ch. 5 (ordering), Ch. 7 |
| E7 | **Flaky payment provider**: `Confirm` calls an external mock payment container that sometimes times out. Guarantee that no fan is charged twice and no seat is sold without payment (idempotency keys, outbox pattern). | Ch. 4, Ch. 8.3 |

### 4.3 Non-functional requirements

| ID | Requirement |
|---|---|
| N1 | Runs on **2 physical computers** on the same LAN. Each computer runs ≥ 1 Raft node. |
| N2 | After the leader crashes, selling resumes within **3 s** (LAN, default timeouts). |
| N3 | ≥ **500 successful holds/s** sustained on the 2-computer cluster (sanity floor, not a race). |
| N4 | `make test` runs the automated suite (single machine, simulated network) in under 10 minutes. |
| N5 | **Zero** oversells, double holds, or limit violations in every audited run. |
| N6 | Code is readable. Safety-critical paths cite the Raft paper rule they implement. |

## 5. Out of scope

- Byzantine faults (discussed in the report only).
- Real payments, real user accounts, or a polished fan-facing UI. The dashboard is an *ops/visualization* UI.
- A production storage engine. In-memory state plus a write-ahead log on a Docker volume is enough.

## 6. Deliverables

1. Git repository: source, `Dockerfile`s, compose/stack files, tests, scripts, and a README that lets another team reproduce the 2-computer setup.
2. Design document (Week 1, 2–3 pages).
3. Final report (max 8 pages, see [07-report-template.md](07-report-template.md)).
4. "The Big Sale" live demo (15 min) on 2 computers, plus Q&A.
5. `CONTRIBUTIONS.md`: half a page per student.
