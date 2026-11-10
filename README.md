# Course Project — **TicketRush**: A Fault-Tolerant Flash-Sale Ticketing Platform

**Course:** Distributed Systems (Master's level)
**Textbook:** M. van Steen & A.S. Tanenbaum, *Distributed Systems*, 4th ed. — <https://www.distributed-systems.net/>
**Duration:** 4 weeks · **Team size:** 2–3 students
**Deployment:** Docker, on **2 physical computers** connected over a LAN
**Recommended language:** Go (Rust, Java, Kotlin, or Python are allowed; see [05-testing-and-evaluation.md](05-testing-and-evaluation.md))

---

## The story

A huge concert goes on sale at 10:00:00 sharp. There are **5,000 seats** and **50,000 fans** (bots) pressing "Buy" at the same second. Anyone who has lived through a *ticket war* knows how it goes wrong: the site goes down, a seat is sold to two people, payments are charged twice, and some fans see "available" for a seat that is already gone.

Your team builds **TicketRush**, a ticketing backend that **never sells a seat twice**, **never charges a fan twice**, and **keeps selling while servers crash and network cables are pulled out**. The cluster runs as Docker containers spread over **two real computers**. During the final demo the instructor will pull the network cable of one of them in the middle of a sale.

Underneath, TicketRush is a **replicated state machine built on Raft**, implemented by you. The ticketing logic gives every consistency guarantee a visible consequence. A linearizability bug here appears on the screen as two fans holding the same seat.

## What makes it fun (and hard)

- **Live seat map dashboard.** Seats flip from green to yellow (held) to red (sold) in real time. Next to the map, every Raft node is shown with its role, term, and commit index. Kill the leader and you can watch the sale pause, a new leader get elected, and the sale resume.
- **Bot army.** A load-generator container simulates thousands of competing buyers. For the final demo, teams' bots can compete in the same sale ("who got the front row?").
- **Real failures.** These are actual crashes, cable pulls, and Wi-Fi drops between two machines, not a simulation.
- **The 0-oversell audit.** After every chaos run, an auditor checks that no seat was sold twice, no user exceeded the ticket limit, and the history is linearizable. One violation fails the run.

## Textbook coverage

| Chapter | Where it shows up in TicketRush |
|---|---|
| Ch. 1 Introduction | Failure and replication transparency for fans. The *fallacies of distributed computing* come up when you deploy over a real LAN. |
| Ch. 2 Architectures | Replicated state machine, stateless clients, dashboard as an observer |
| Ch. 3 Processes | Containers, multithreaded servers, crash and restart of processes |
| Ch. 4 Communication | RPC between nodes on different hosts, timeouts, retries, exactly-once |
| Ch. 5 Coordination | Leader election, terms as logical clocks, hold expiry without trusting local clocks, clock skew between the 2 computers |
| Ch. 6 Naming | Leader discovery across hosts. With the extension: consistent hashing of events and sections to Raft groups. |
| Ch. 7 Consistency & Replication | "Buy" must be linearizable. The seat map may be stale, and you measure the trade-off. |
| Ch. 8 Fault Tolerance | Quorums, failure domains, stable storage, recovery. With the extension: 2PC for cross-shard purchases. |
| Ch. 9 Security | (extension) mTLS between hosts, authenticated buyers, anti-bot measures |

## Documents

| File | Contents |
|---|---|
| [01-project-brief.md](01-project-brief.md) | Learning objectives, requirements, scope |
| [02-system-specification.md](02-system-specification.md) | Architecture, ticketing API, state machine, Raft RPCs, invariants, extensions |
| [03-deployment-docker.md](03-deployment-docker.md) | **Running on 2 computers with Docker**: topology, networking, quorum placement, fault injection |
| [04-weekly-plan.md](04-weekly-plan.md) | Week-by-week milestones and deliverables |
| [05-testing-and-evaluation.md](05-testing-and-evaluation.md) | Test suite, the oversell audit, linearizability checking, required experiments |
| [06-grading-rubric.md](06-grading-rubric.md) | Assessment criteria and sample Q&A questions |
| [07-report-template.md](07-report-template.md) | Structure of the final report (paper style, max 8 pages) |
| [08-reading-list.md](08-reading-list.md) | Papers and tools |

## Milestones

| When | Milestone |
|---|---|
| End of Week 1 | **M1** — Design doc. Containers on 2 computers can reach each other. Leader election works across machines. |
| End of Week 2 | **M2** — Log replication plus persistence. The ticketing state machine sells seats. |
| End of Week 3 | **M3** — Exactly-once, linearizable buys, dashboard, bot army, one extension |
| End of Week 4 | **M4** — "The Big Sale": live demo on 2 computers with a cable pull, plus report and code |

## Academic integrity

You may read the Raft paper, the textbook, and public documentation, and you may discuss ideas with other teams. You **may not** copy a Raft implementation (e.g., `etcd/raft`, `hashicorp/raft`, MIT 6.5840 solutions), and you may not use an existing consensus or coordination service (etcd, ZooKeeper, Consul) as your replication layer. **Raft must be your own code.** Off-the-shelf libraries are fine for RPC, serialization, HTTP, and the dashboard UI. Every member must be able to explain any part of the system. Disclose AI-assistant use in the report.
