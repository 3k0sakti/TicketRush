# 08 — Reading List

## Required

1. **M. van Steen & A.S. Tanenbaum.** *Distributed Systems*, 4th ed., 2023. <https://www.distributed-systems.net/>
   Ch. 4 Communication · Ch. 5 Coordination · Ch. 7 Consistency and Replication · Ch. 8 Fault Tolerance
2. **D. Ongaro & J. Ousterhout.** *In Search of an Understandable Consensus Algorithm.* USENIX ATC 2014. <https://raft.github.io/raft.pdf>. Figure 2 is your spec.
3. **M. Herlihy & J. Wing.** *Linearizability: A Correctness Condition for Concurrent Objects.* ACM TOPLAS, 1990.

## Strongly recommended

4. **D. Ongaro.** *Consensus: Bridging Theory and Practice.* PhD dissertation, Stanford, 2014. Covers membership (§4), compaction (§5), and client interaction / exactly-once / ReadIndex (§6).
5. **L. Lamport.** *Paxos Made Simple.* 2001. For related work.
6. **Fischer, Lynch, Paterson.** *Impossibility of Distributed Consensus with One Faulty Process.* JACM, 1985. Explains why Raft needs timeouts for liveness.
7. **E. Brewer.** *CAP Twelve Years Later: How the "Rules" Have Changed.* IEEE Computer, 2012 · **D. Abadi.** *Consistency Tradeoffs in Modern Distributed Database System Design* (PACELC), IEEE Computer, 2012.
8. Raft visualization: <https://raft.github.io/>

## Extension-specific

| Extension | Reading |
|---|---|
| E1 Snapshots | Ongaro dissertation §5 |
| E2 Sharding + 2PC | Karger et al., *Consistent Hashing and Random Trees*, STOC 1997 · Textbook Ch. 8.5 · Corbett et al., *Spanner*, OSDI 2012 (§4, 2PC over Paxos groups) |
| E3 Witness / membership | Ongaro dissertation §4 · Raft paper §6 |
| E4 Leases | Gray & Cheriton, *Leases*, SOSP 1989 · Textbook Ch. 5.1–5.2 (clock synchronization) |
| E5 Security | Textbook Ch. 9 · Lamport, Shostak, Pease, *The Byzantine Generals Problem*, TOPLAS 1982 |
| E6 Waiting room | Lamport, *Time, Clocks, and the Ordering of Events in a Distributed System*, CACM 1978 |
| E7 Flaky payment | Helland, *Idempotence Is Not a Medical Condition*, ACM Queue 2012 · Textbook Ch. 8.3 |

## Tools

- **Porcupine**: linearizability checker (Go). <https://github.com/anishathalye/porcupine>
- **Jepsen** analyses: <https://jepsen.io/analyses>. Excellent reading on how real databases fail.
- **Docker Compose** / **Docker Swarm** docs: <https://docs.docker.com/>
- **tc netem** (latency/loss) · **iptables** (partitions) · **iperf3** (bandwidth) · **Toxiproxy** (TCP fault proxy)
