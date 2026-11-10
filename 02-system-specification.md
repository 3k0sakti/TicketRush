# 02 — System Specification

## 1. Architecture overview

```
        COMPUTER A (e.g. 192.168.1.10)                 COMPUTER B (e.g. 192.168.1.20)
  +------------------------------------------+   +------------------------------------------+
  |  docker                                  |   |  docker                                  |
  |                                          |   |                                          |
  |  +--------+  +--------+  +--------+      |   |      +--------+  +--------+              |
  |  | node1  |  | node2  |  | node3  |<-----+---+----->| node4  |  | node5  |              |
  |  | Raft   |  | Raft   |  | Raft   |      |   |      | Raft   |  | Raft   |              |
  |  +---+----+  +---+----+  +---+----+      |   |      +---+----+  +---+----+              |
  |    [vol]       [vol]       [vol]         |   |        [vol]       [vol]                 |
  |                                          |   |                                          |
  |  +-------------+                         |   |   +-------------+   +-----------------+  |
  |  | dashboard   |  (web UI, :8080)        |   |   | bot-army    |   | auditor/checker |  |
  |  +-------------+                         |   |   +-------------+   +-----------------+  |
  +------------------------------------------+   +------------------------------------------+
                    \__________________ LAN (wired preferred) __________________/
```

The default placement is 5 nodes as **3 on A + 2 on B**. [03-deployment-docker.md](03-deployment-docker.md) explains why, and what happens with other placements.

Layers inside each Raft node container:

```
+-------------------------------------------------+
| HTTP/gRPC API (ticketing ops, /status, /seatmap)|
+-------------------------------------------------+
| Ticketing state machine                         |
|   seats, holds, tickets, per-user counts,       |
|   dedup table, logical "now"                    |
+-------------------------------------------------+
| Raft module (election, replication, commit,     |
|   ReadIndex, snapshots*)                        |
+-------------------------------------------------+
| Transport (peer RPC over the LAN) + fault hook  |
+-------------------------------------------------+
| Persistence: WAL + metadata on a Docker volume  |
+-------------------------------------------------+
```

**Design rule:** the Raft module knows nothing about tickets. It replicates opaque `[]byte` commands and delivers committed entries to the state machine **in log order**.

## 2. Ticketing API (client-facing)

Use gRPC or JSON over HTTP. Every mutating request carries a header:

```json
{ "clientId": "bot-17-3f9a", "seqNo": 42 }
```

| Operation | Input | Output | Notes |
|---|---|---|---|
| `CreateEvent` | eventId, sections[{name, rows, seatsPerRow}] | ok | Admin only |
| `Hold` | eventId, userId, seatIds[] (1–4) | holdId, expiresAt \| `SEAT_TAKEN` \| `LIMIT_EXCEEDED` | All-or-nothing |
| `HoldBestAvailable` | eventId, userId, section, count | holdId, seatIds[], expiresAt \| `SOLD_OUT` | Deterministic choice (e.g., lowest row, then lowest seat number) |
| `Confirm` | holdId | ticketIds[] \| `HOLD_EXPIRED` \| `UNKNOWN_HOLD` | Turns HELD into SOLD |
| `Release` | holdId | ok | Gives seats back |
| `GetSeatMap` | eventId, mode = `linearizable` \| `stale` | seat states + `asOfIndex` | `stale` may be served by any node |
| `GetMyTickets` | eventId, userId | ticketIds[] | Linearizable |

Every reply includes `status`, and for `WRONG_LEADER` a `leaderHint` (host:port reachable *from the other computer*).

### Client library behavior (used by bots and the dashboard)
1. Remember the last known leader and try it first. Follow `leaderHint` on `WRONG_LEADER`. Otherwise try the next node round-robin.
2. On timeout or connection error, **retry with the same `(clientId, seqNo)`**.
3. Keep at most one outstanding mutating request per clientId. Each bot is one client.

## 3. Peer RPCs (Raft)

Exactly as in Figure 2 of the Raft paper:

| RPC | Arguments | Results |
|---|---|---|
| `RequestVote` | term, candidateId, lastLogIndex, lastLogTerm | term, voteGranted |
| `AppendEntries` | term, leaderId, prevLogIndex, prevLogTerm, entries[], leaderCommit | term, success, (opt.) conflictTerm, conflictIndex |
| `InstallSnapshot` (E1) | term, leaderId, lastIncludedIndex, lastIncludedTerm, data | term |

Default timings. You may tune them, but justify your values with the RTT you **measure** between your 2 computers:

| Parameter | Default |
|---|---|
| Heartbeat interval | 50 ms |
| Election timeout | random in [300, 600] ms (higher than in the paper, because Wi-Fi/LAN jitter is real) |
| Client request timeout | 1 s |
| Hold duration | 60 s (use 5 s in tests) |

## 4. State machine

### 4.1 State

```
events   : eventId -> { capacity, sections }
seats    : (eventId, seatId) -> AVAILABLE | HELD(holdId) | SOLD(userId, ticketId)
holds    : holdId -> { eventId, userId, seatIds, expiresAt }
owned    : (eventId, userId) -> count of held + sold seats     // for the 4-seat limit
dedup    : clientId -> (lastSeqNo, cachedReply)
now      : logical time of the state machine (see 4.3)
```

### 4.2 Apply logic (sketch)

```
apply(entry):
  now = max(now, entry.leaderTimestamp)         // monotonic, replicated time
  expireHolds(now)                              // deterministic: same on every node
  if entry.seqNo <= dedup[entry.clientId].seqNo:
      return dedup[entry.clientId].reply        // duplicate -> exactly-once
  reply = execute(entry.op)                     // Hold / Confirm / Release / ...
  dedup[entry.clientId] = (entry.seqNo, reply)
  return reply
```

### 4.3 Time and hold expiry — the subtle part

Computers A and B have **different clocks**. If each node checked `time.Now() > expiresAt` while applying an entry, node1 on A could treat a hold as expired while node4 on B does not. The replicas would diverge, and divergence leads to oversell.

**Required approach:** the leader stamps every log entry with its wall-clock time (`leaderTimestamp`). The state machine uses only `now = max(now, entry.leaderTimestamp)`, which is part of the replicated state. Every node therefore expires the same holds at the same log index. The leader also appends a periodic `Tick` entry (e.g., every 1 s) so that holds expire even when no other traffic arrives.

In the report, discuss what happens when a new leader's clock is *behind* the old leader's clock, and why `max(...)` keeps `now` monotonic.

### 4.4 Read modes

| Mode | Served by | Guarantee | Use |
|---|---|---|---|
| `linearizable` | Leader, after ReadIndex confirmation (or through the log) | Reflects every committed write | `GetMyTickets`, pre-checks |
| `stale` | Any node, from its local applied state | Monotonic per node, may lag. Returns `asOfIndex`. | Dashboard seat map, "browse seats" |

## 5. Invariants

State all of these in the design document. Assert them in debug builds. The auditor checks them after every run.

| # | Invariant |
|---|---|
| I1 | **Election Safety**: at most one leader per term |
| I2 | **Log Matching**: same index and term implies identical prefix |
| I3 | **Leader Completeness**: committed entries appear in all later leaders' logs |
| I4 | **State Machine Safety**: all nodes apply the same command at each index (check with a per-index state hash) |
| I5 | **No oversell**: each seat is sold at most once |
| I6 | **No double hold**: a seat belongs to at most one active hold |
| I7 | **Per-user limit**: `owned(event, user) ≤ 4` at all times |
| I8 | **Conservation**: `available + held + sold = capacity` |
| I9 | **Exactly-once**: each `(clientId, seqNo)` changes the state at most once |
| I10 | **Linearizability**: the client-observed history of Hold/Confirm/linearizable reads is linearizable |

## 6. Persistence

```
/data (Docker named volume, one per node)
  meta.json          { "currentTerm": 7, "votedFor": "node2" }   write-temp + fsync + rename
  wal.log            length-prefixed, CRC32-checked entries      fsync before replying
  snapshot-<i>.bin   (E1)
```

The data must survive `docker kill` + `docker start`, and a full `docker compose down` + `up` (without `-v`). In the report, explain how you deal with a torn last write in `wal.log`.

## 7. Dashboard (F9)

A single page, with any framework you like (plain HTML plus JS polling is fine):

- **Seat map** grid, colored by state, refreshed every 200–500 ms with a `stale` read. Show `asOfIndex` and which node served it.
- **Cluster panel**: one card per node, grouped by computer (A or B). Each card shows role (L/F/C), term, commitIndex, lastApplied, and whether it is reachable from the dashboard.
- **Sales ticker**: holds/s, confirms/s, failed requests/s, current leader.
- *(Nice to have)* a timeline that marks elections and fault-injection events.

## 8. Bot army (F10)

A container started with parameters such as:

```
bot-army --event concert-01 --bots 2000 --duration 60s \
         --strategy popular-first --abandon-rate 0.2 --retry-on-timeout \
         --history-out /out/history.jsonl
```

- Strategies: `popular-first` (everyone rushes the front rows, which gives maximum contention), `random`, and `best-available`.
- `abandon-rate`: the fraction of holds that are never confirmed. This exercises expiry.
- Records every operation's invoke and return (see [05-testing-and-evaluation.md](05-testing-and-evaluation.md)).

## 9. Extension notes

- **E1 Snapshots**: snapshot = seats + holds + owned + dedup + `now` + lastIncludedIndex/Term. Trigger when the log has more than `S` entries.
- **E2 Sharding + 2PC**: events/sections map to Raft groups via consistent hashing with virtual nodes. A fan buying in two sections owned by different groups uses 2PC. Each participant is a Raft group, so participants don't "crash" in the classic 2PC sense. Explain what this means for the blocking problem of 2PC. The global 4-ticket limit needs a per-user owner group, or a third 2PC participant.
- **E3 Witness on a 3rd device**: add the node through a membership change (single-server change or joint consensus) while the sale is running. Then show that pulling the cable of *either* computer no longer stops sales.
- **E4 Leases**: measure the actual clock offset and drift between A and B (e.g., with NTP stats or your own ping-based estimate). Choose a lease with a safety margin. Show the unsafe case with a skewed clock.
- **E5 Security**: mTLS with your own CA, signed buyer tokens checked by the leader, and replicated rate-limit counters.
- **E6 Waiting room**: a replicated FIFO queue whose admission entries go through the log. Fairness metric: Kendall's τ between arrival order and admission order.
- **E7 Flaky payment**: a mock payment container fails or times out with probability *p*. `holdId` is the idempotency key. Show that no double charge happens under leader failover in the middle of `Confirm`.
