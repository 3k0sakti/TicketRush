# 05 — Testing and Evaluation

A ticketing system that "seems to work" in a demo is worth little. What counts is **evidence** that it never oversells under failures. This document defines the minimum test suite, the auditor, and the required experiments.

Testing happens at two levels:

| Level | Where | Network | Purpose |
|---|---|---|---|
| **Automated suite** | One machine, `make test` | In-process simulated network (deterministic, seeded) | Fast, reproducible correctness testing. Hundreds of runs. |
| **Deployment chaos** | 2 computers, Docker | Real LAN plus `docker kill`, `iptables`, `tc`, cable pulls | Shows that it works for real. All experiments run here. |

Language note: Go is the easiest choice (goroutines, gRPC, and Porcupine is native Go). With Python, the GIL and per-request overhead are real problems. Lower N3 to 200 holds/s and discuss the reason in your report.

---

## 1. The auditor

The auditor is a separate container or tool that runs after every chaos run and **fails the run** on any violation.

| Check | How |
|---|---|
| I4 State machine safety | All nodes report the same state hash at the same `lastApplied` |
| I5 No oversell | Every seat has ≤ 1 ticket. Cross-check that `SOLD` seats on all nodes agree. |
| I6 No double hold | Every bot-observed successful `Hold` maps to a distinct holdId. No two overlapping active holds share a seat in the history. |
| I7 Per-user limit | For every user, held + sold ≤ 4 at every log index (replay the log) |
| I8 Conservation | available + held + sold = capacity, on every node |
| I9 Exactly-once | For every `(clientId, seqNo)` with retries, there is at most one effect in the log |
| I10 Linearizability | Run the checker on the recorded history (§4) |

Output example:

```
run=chaos-seed-8812  ops=48211  holds_ok=7310  confirms_ok=4987
oversold=0  double_holds=0  limit_violations=0  conservation=OK
state_hash_agree=5/5  linearizable=YES (checked 5000 seats, 3.2s)
RESULT: PASS
```

---

## 2. Automated test suite (`make test`)

All randomized tests log their seed. Failed seeds must be replayable.

### 2.1 Raft election
| Test | Pass condition |
|---|---|
| `TestInitialElection` | One leader within 3 s. The term is stable when there are no faults. |
| `TestReElection` | A disconnected leader is replaced, and steps down when it reconnects |
| `TestNoQuorumNoLeader` | No commit without a majority |
| `TestElectionSafetyRandom` | 100 random partition schedules: never two leaders in one term |
| `TestTwoDomainPlacement` | Simulate the 3+2 placement: cutting off the "B" nodes keeps progress, cutting off the "A" nodes stops progress |

### 2.2 Raft replication
| Test | Pass condition |
|---|---|
| `TestBasicAgree` | 100 commands applied in the same order everywhere |
| `TestFailAgree` / `TestFailNoAgree` | Progress with a minority down, and none without a majority |
| `TestRejoinConflictingLeader` | Uncommitted conflicting entries of an old leader are overwritten |
| `TestFigure8` | Raft Figure 8 scenario: no committed entry is lost |
| `TestPersistCrashAll` | Crash all nodes, restart them, and the committed state is intact |
| `TestUnreliableNetwork` | 10% loss plus random delay: convergence |

### 2.3 Ticketing semantics
| Test | Pass condition |
|---|---|
| `TestHoldAllOrNothing` | A multi-seat hold with one seat taken holds nothing |
| `TestUserLimit` | The 5th seat for a user is rejected, including across concurrent clients of the same user |
| `TestHoldExpiryDeterministic` | Nodes with skewed clocks (±2 s) expire exactly the same holds at the same index |
| `TestExpiryWithoutTraffic` | `Tick` entries expire holds even with no client traffic |
| `TestRetryNoDoubleHold` | Drop every first reply. Retries must not create a second hold. |
| `TestConfirmAcrossFailover` | Leader crashes during `Confirm`. The client retries on the new leader and gets exactly one set of tickets. |
| `TestNoStaleLinearizableRead` | A partitioned old leader does not serve stale `linearizable` seat maps |
| `TestFlashSaleChaos` | 200 bots, 100 seats, `popular-first`, random crashes/partitions/loss for 30 s → **auditor PASS**. Run ≥ 200 times. Report the pass rate. |

### 2.4 Extension tests
At least two targeted tests per extension. Examples: a lagging follower catches up via snapshot (E1); a cross-section purchase during a coordinator-group failover stays atomic (E2); the witness is added during a sale and then computer A is cut off, and sales continue (E3); a skewed lease clock produces a stale read that the checker catches (E4).

---

## 3. Required experiments (on the 2-computer deployment)

For each experiment, give: **hypothesis → setup → result (figure with CI/percentiles) → interpretation tied to theory**. Use ≥ 5 repetitions per configuration. Report the measured LAN RTT in the setup.

| # | Experiment | Vary | Measure | Example hypothesis |
|---|---|---|---|---|
| X1 | **Sale pause on leader failure** | Election timeout range (e.g., 150–300, 300–600, 800–1600 ms); leader on A vs. B | Time from `docker kill` to the first successful hold; split votes | "Pause ≈ timeout + one RTT. Below ~5× the LAN jitter, split votes become frequent." |
| X2 | **Placement and cluster size** | 3 nodes (2+1), 5 nodes (3+2), and 5 nodes all on A (baseline) | Holds/s, p50/p99 latency | "Spreading over 2 hosts adds roughly one LAN RTT to commit latency, because the majority needs a cross-host ack *only when* the leader's host lacks a local majority." Check whether this holds. |
| X3 | **Seat-map read cost vs. freshness** | `linearizable` (ReadIndex or log) vs. `stale` (vs. lease, E4) | Read latency p50/p99, throughput at 95/5 browse/hold, *staleness* of stale reads (index lag, ms) | "Stale reads are an order of magnitude cheaper, and their staleness is bounded by roughly the heartbeat interval in steady state." |
| X4 | **Cable pull (CAP in practice)** | Partition A⟷B with the leader on A, then with the leader on B | Successful ops/s on each side over time, and the error types bots see | "The side with the majority keeps selling (after an election if the leader was on B). The minority side's error rate goes to 100% with 0 oversells." |
| X5 | **Contention** | Bot strategy `popular-first` vs. `random`, number of bots | Success rate, `SEAT_TAKEN` rate, latency | "Contention does not reduce Raft throughput (everything is serialized anyway), but it lowers fans' success rate." |
| X6 | **Extension-specific** | E.g., snapshot threshold; vnodes; witness yes/no; lease margin; waiting-room batch size; payment failure rate | Your choice | Your own |

Plot rules:
- Labeled axes with units, and legends.
- Time-series plots (X1, X4) mark fault-injection moments, using timestamps from the `chaos.sh` log.
- Every figure can be regenerated from `eval/` by one script.

---

## 4. Linearizability checking

- Record for each operation: `clientId, op, args, result, invokeTs, returnTs`, using the **bot-army machine's monotonic clock**.
- Model the history **per seat** as a register whose state is AVAILABLE, HELD(h), or SOLD(u). `Hold` of several seats touches several registers. Either check them together, or argue why checking per seat is sufficient for I5/I6.
- A **timed-out** operation has an unknown outcome. Model it as possibly taking effect at any time after its invoke (Porcupine: return = ∞).
- Keep per-partition histories small (≤ a few thousand ops per seat). Linearizability checking is NP-complete in general.
- **Mutation test (required).** Break the system on purpose, for example by serving `linearizable` reads locally on followers, removing the current-term commit restriction, or using `time.Now()` in `apply()`. Show that the auditor or checker catches the bug. Put it in the report.
- If bots run on both computers, see [03-deployment-docker.md](03-deployment-docker.md) §5 on merging histories across clocks.

---

## 5. Definition of done (minimum to pass)

- All tests in §2.1–2.3 pass.
- `TestFlashSaleChaos` passes ≥ 99% of 200 runs, and every failure is explained.
- The 2-computer deployment works, and the cable-pull demo (X4) shows zero violations.
- X1–X5 are complete. At least one extension is implemented and evaluated (X6).
