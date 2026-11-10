# 06 — Grading Rubric

**Total: 100 points.** The project grade is a team grade. An individual adjustment of up to ±15% can be applied based on the Q&A and `CONTRIBUTIONS.md`.

## 1. Breakdown

| Component | Points |
|---|---|
| Milestones M1–M3 (on time, complete) | 10 |
| Core implementation (Raft + ticketing, F1–F10) | 25 |
| 2-computer Docker deployment (F11) | 10 |
| Extension(s) | 15 |
| Testing and correctness evidence | 15 |
| Experimental evaluation | 15 |
| Report | 5 |
| "The Big Sale" demo and Q&A | 5 |

## 2. Criteria

### Core implementation (25)
| Level | Description |
|---|---|
| 23–25 | All F1–F10 are correct. Expiry is deterministic, exactly-once holds across failover, and both read modes work. Clean code, with safety-critical paths citing the Raft paper. No data races (`-race` or equivalent). |
| 17–22 | Everything works, with minor robustness or liveness problems under heavy loss. |
| 12–16 | Raft works, but ticketing guarantees (exactly-once, limits, expiry) are incomplete. |
| < 12 | Safety violations in basic scenarios. |

### 2-computer deployment (10)
| Level | Description |
|---|---|
| 9–10 | Reproducible with documented commands and `.env`. Correct advertise addresses. Volumes persist. Placement is justified with a quorum analysis. Seeded `chaos.sh` works across hosts. |
| 6–8 | Runs on 2 computers, but setup is manual or fragile, or the placement is not justified. |
| 3–5 | Only partly distributed (e.g., all Raft nodes on one host). |
| 0–2 | Single machine only. |

### Extension(s) (15)
| Level | Description |
|---|---|
| 13–15 | Two extensions, or one ⭐ extension (E2 sharding + 2PC, E3 witness + membership change), fully tested and evaluated. |
| 9–12 | One extension, fully working, tested, and evaluated. |
| 5–8 | Partly working or not evaluated. |
| 0–4 | Missing or broken. |

### Testing and correctness evidence (15)
| Level | Description |
|---|---|
| 13–15 | Full suite. Chaos pass rate ≥ 99% over ≥ 200 runs. The auditor checks I4–I10. A mutation test proves that the checker catches bugs. Seeds are reproducible. |
| 9–12 | Most tests present. The checker is used, but runs are fewer or failures unanalyzed. |
| 5–8 | Mostly happy-path tests. |
| 0–4 | Little automated testing. |

### Experimental evaluation (15)
| Level | Description |
|---|---|
| 13–15 | X1–X6 run on the real 2-computer setup, with hypotheses, repetitions, CIs/percentiles, and interpretation tied to theory (quorum, RTT, fsync, failure domains). Reproducible from scripts. |
| 9–12 | All experiments done. Shallow interpretation or weak statistics. |
| 5–8 | Experiments missing, or numbers reported without analysis. |
| 0–4 | No meaningful evaluation. |

### Report (5)
Precise textbook terminology, justified design decisions with alternatives considered, honest limitations, and within the page limit.

### Demo and Q&A (5)
The live cable-pull demo works and is explained clearly. Each member answers technical questions.

## 3. Sample Q&A questions

1. Your placement is 3+2. Why does losing computer B not stop sales, but losing A does? Would 6 nodes as 3+3 be better?
2. Why can't the leader commit an entry from a previous term just by counting replicas? Walk through Raft Figure 8.
3. Node1 on A and node4 on B have clocks 2 s apart. Show that they still expire exactly the same holds.
4. A new leader's clock is behind the old leader's clock. What happens to `now`, and to holds that were about to expire?
5. A fan's `Confirm` times out and the bot retries on a different node after a failover. Prove that the fan gets exactly one set of tickets.
6. Why may the seat map be stale but `Hold` may not? Is there any risk in letting fans *browse* a stale map?
7. Where is TicketRush on the CAP / PACELC spectrum? What do fans on computer B experience during a cable pull?
8. If nodes could be Byzantine (e.g., a compromised node), what breaks? How many replicas would you need?
9. *(E2)* 2PC is known to block when the coordinator fails. Why is that less of a problem when every participant is a Raft group?
10. *(E3)* Why does a witness on a third device fix what adding more nodes on A or B cannot?

## 4. Penalties

| Issue | Penalty |
|---|---|
| Late submission | −10% per day (max 3 days) |
| Report over the page limit | −2 points |
| Results not reproducible from the repo | −5 points |
| Using an existing consensus library or service as the replication layer | Core implementation scored 0 |
| Copied implementation | Grade 0, and referral under academic integrity rules |
