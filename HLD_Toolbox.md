# HLD Toolbox --- SDE-2 Interview Revision

A practical quick-reference for common High-Level Design components.

## 1. Caching

**Problem it solves:** Repeated or popular reads hit the database again and again, increasing latency and DB load.

**Basic cache-aside flow:**

``` text
Request → Cache
           ├─ HIT  → Return cached value
           └─ MISS → Read DB → Store in cache → Return
```

- **Cache-aside:** Application checks the cache first; on a miss, it reads the DB and populates the cache.
- **TTL (time to live):** How long an entry may remain in the cache before it expires.
- **Eviction:** Removing entries when space is needed (e.g., LRU).
- **Invalidation:** Removing or updating cached data when the source changes.
- **Cache hit rate:** Fraction of requests served by the cache.
- **Hot key:** One key receives a disproportionately large amount of traffic.
- **Cache stampede:** Many requests miss/expire together and overwhelm the DB.

**Use when:** The same data is read frequently and can tolerate the chosen freshness policy.

**Trade-offs / risks:**

- Lower latency and DB read load, but cached data can be stale.
- Adds memory cost and invalidation/consistency complexity.
- Cache failure or mass expiry can send a sudden traffic spike to the DB.

**Choose the right tool:**

- Repeated popular data → Cache
- Slow filtering/search over rows → Database index
- Many different DB reads → Read replicas
- Static content delivered globally → CDN
- Expensive repeated computation → Precomputation

**Interview takeaway:** A cache is a fast layer in front of the source of truth; explain freshness, invalidation, and what happens on a cache miss/failure.

## 2. Load Balancer

**Problem it solves:** A single application server can become overloaded or unavailable.

**Basic flow:**

``` text
Clients → Load Balancer → App Server 1
                       → App Server 2
                       → App Server 3
```

- **Load balancer (LB):** Distributes incoming requests/connections across healthy backend instances.
- **Horizontal scaling:** Adding more instances instead of only making one server bigger.
- **Health check:** Periodically checks whether an instance can receive traffic; unhealthy instances are removed from rotation.
- **Round Robin:** Sends requests to instances in rotation.
- **Least Connections:** Prefers the instance with fewer active connections.
- **Sticky session / session affinity:** Routes a user's requests to the same instance. Can create uneven load; stateless services are generally easier to scale.
- **L4 LB:** Routes using transport-level information (e.g., TCP/UDP).
- **L7 LB:** Understands application protocols such as HTTP and can route using host/path/headers.
- **Connection draining:** Stops sending new requests to an instance while allowing existing requests to finish before shutdown.

**Use when:** Traffic must be distributed across multiple service instances for scale and availability.

**Trade-offs / risks:**

- Adds infrastructure and a network hop.
- The LB layer must itself be highly available; otherwise it can become a single point of failure.
- Bad health checks can remove healthy instances or keep unhealthy ones in rotation.
- Sticky sessions reduce flexibility and may cause uneven load.

**Interview takeaway:** The LB distributes traffic; it does not reduce the amount of incoming traffic. Mention health checks, redundancy, and stateless application servers.

## 3. Database Indexes

**Problem it solves:** A query that scans a large table row by row can be slow.

**Without an index:** The DB may perform a full table scan.
**With an index:** The DB can use an auxiliary data structure to find matching rows more efficiently.

- **Index:** A data structure mapping indexed column values to row locations/keys.
- **B-tree / B+ tree:** Common balanced tree structures used by database indexes; they support efficient lookups and ordered/range access.
- **Selectivity:** How uniquely a column identifies rows. Email is usually more selective than gender.
- **Composite index:** An index over multiple columns, e.g. `(user_id, status)`.
- **Column order matters:** `(user_id, status)` is generally useful for queries filtering by `user_id`, or by both columns; it may not help a query filtering only by `status`.
- **Query optimizer:** Chooses a query execution plan and decides whether an index is worthwhile.

**Use when:** Columns are frequently used in `WHERE`, `JOIN`, or relevant `ORDER BY` patterns.

**Trade-offs / risks:**

- Faster reads, but indexes consume storage.
- Inserts, updates, and deletes must maintain affected indexes, adding write overhead.
- Too many indexes can hurt write-heavy workloads.
- An index is not guaranteed to be used; for a small table or low-selectivity query, a full scan may be cheaper.

**Index vs cache vs replica:**

- Expensive query over rows → Index
- Repeated access to the same/popular values → Cache
- Overall read volume is too high even after query optimization → Read replicas

**Interview takeaway:** Don't say "index every column." Tie index choice to actual query patterns, selectivity, composite-column order, and write cost.

## 4. Database Replication

**Problem it solves:** One database may not be able to serve all read traffic or provide the required availability.

### Read replicas

``` text
                 ┌→ Read Replica 1 → Reads
Writes → Primary ├→ Read Replica 2 → Reads
                 └→ Read Replica 3 → Reads
```

- **Primary/leader:** Accepts writes.
- **Read replica/follower:** Receives copies of primary changes and can serve reads.
- Replicas reduce the primary's **read** load; writes still go to the primary in a typical single-leader setup.
- **Replication lag:** A replica has not yet applied the latest primary changes, so a read may be stale.
- For a critical read that must reflect a just-completed write, route to the primary or use a consistency-aware strategy.

**Cache vs read replica:**

- Cache: stores selected/frequently accessed data; very fast, often has TTL/eviction.
- Read replica: a replicated copy of database data; distributes DB read workload.
- They can be used together.

### Synchronous vs asynchronous replication

- **Asynchronous:** Primary acknowledges the write without waiting for replica acknowledgement. Lower write latency, but replicas may lag. If the primary fails before a write reaches a surviving replica, an acknowledged write may be lost during failover.
- **Synchronous:** Primary waits for the configured replica acknowledgement(s) before confirming the write. Stronger replication/durability protection, but adds latency and can be affected by replica/network unavailability.
- "Synchronous" does not automatically mean every replica mustacknowledge; the configured acknowledgement policy matters.

**Read-your-writes:** A user sees their own latest successful write on subsequent reads. One approach is to route that user's reads to the primary until replicas catch up. This is narrower than global strong consistency.

### Replication Architectures

- **Single-leader:** One node accepts writes; followers replicate from it. Simple and common, but the leader can be a write bottleneck and needs failover.

- **Multi-leader:** Multiple nodes/regions accept writes. Useful for local writes across regions, but concurrent updates can conflict and need resolution (e.g., application merge or last-write-wins).

- **Leaderless:** No fixed leader; clients read/write across replicas. Often uses quorum-style acknowledgements and version/conflict handling.

### Quorum basics

- `N` = total replicas
- `W` = replicas that must acknowledge a write
- `R` = replicas consulted for a read
- A commonly used overlap condition is `R + W > N`. For example, with `N=3`, `W=2`, `R=2`, the read and write sets overlap under the usual quorum assumptions.
- Overlap alone does not guarantee the latest value in every system; versioning, conflict resolution, and failure semantics also matter.

### Failover and Failure Risks

**Failover:** Promote a healthy replica when the primary fails, then redirect writes and reconfigure replication.

- **Data Loss:** An acknowledged async write may not have reached the promoted replica.
- **Split Brain:** Old and new primary both believe they are leader and accept writes.
- **Leader Election:** Nodes agree on a new leader; **fencing/epochs** help prevent an old leader from continuing to write after losing leadership.
- **Durability:** A committed, acknowledged write should survive failures according to the system's stated guarantees.
- **Eventual Consistency:** Replicas may temporarily disagree but converge after updates stop and replication succeeds.
- **Idempotency:** Retrying the same logical operation should not create a duplicate effect. A unique key/constraint and atomic persistence of the operation result help; retries with the same key should return the existing result, not repeat the operation.

**Interview Takeaway:** Explain the read-scaling benefit and then discuss replication lag, consistency needs, failover, and the possibility of losing unreplicated acknowledged writes.

## Quick Comparison

| Component | Main Problem Addressed |
| --- | --- |
| Cache | Repeated or popular reads are slow or expensive. |
| Load Balancer | Traffic needs to be distributed across multiple service instances. |
| DB Index | Queries scan too many rows, resulting in slow database lookups. |
| Read Replica | The primary database is overloaded with read requests. |
| Sharding | A single database cannot handle the data volume or write workload, so data is split across multiple nodes. *(Next topic)* |

## 5. Database Sharding

**Problem it solves:** A single database may reach its storage, CPU, I/O, or write-throughput limits even after query optimization and read replicas. Sharding splits the dataset across multiple database nodes so capacity and workload can scale horizontally.

### Basic Architecture

```text
                  Application / Shard Router
                    /        |        \
                   v         v         v
                Shard 1    Shard 2    Shard 3
              (subset A) (subset B) (subset C)
```

- **Shard:** An independent database holding a subset of the overall data.
- **Shard Key:** The field(s) used to determine where a record belongs. Choose it around both distribution and common query patterns.
- **Shard Routing:** Mapping a request/key to the correct shard.
- **Horizontal Partitioning:** Splitting rows across databases. This is different from splitting columns vertically.

### Common Sharding Strategies

- **Range-based:** Assign key ranges to shards, e.g. user IDs 1–1000 to Shard 1 and 1001–2000 to Shard 2. Simple and useful for range queries, but sequential inserts or uneven key ranges can create a hot shard.
- **Hash-based:** Route using a hash of the shard key, e.g. `hash(user_id) % N`. Usually spreads sequential keys more evenly, but range queries become scatter-gather operations. Adding/removing shards with simple modulo hashing can remap many keys.
- **Consistent hashing:** Place shard nodes and keys on a hash ring; adding/removing a node typically moves only a portion of keys. Virtual nodes improve balance. It reduces data movement, but does not remove the need for migration or careful routing.
- **Directory-based:** A lookup table maps keys/tenants to shards. It allows flexible placement and moving tenants, but the directory must scale, remain available, and handle stale mappings safely.

### Query Patterns and Hotspots

- **Scatter-gather:** If a query cannot be routed to one shard (e.g., filter by a non-shard-key field), send it to multiple shards and merge results. This increases latency, network/DB load, and complexity of sorting and pagination. Prefer shard keys aligned with frequent query patterns; use search/analytics systems for broad reporting where appropriate.
- **Hot shard:** One shard receives disproportionate traffic or data. A hot key (e.g., a celebrity user) can cause this even if the total dataset is evenly distributed.
- **Mitigations:** Revisit the shard key; split a hot tenant/user using a composite key such as `(user_id, post_id)` where access patterns allow; cache read-heavy popular data; isolate very large tenants. A composite key can make fetching all records for one user require multiple partitions. Caching helps read hotspots, not a write hotspot by itself.

### Rebalancing and migration

**Rebalancing** redistributes data when shards are added/removed or capacity/workload becomes uneven.

```text
Identify data to move → Copy to destination → Capture concurrent writes → Catch up changes → Switch routing → Verify and clean up
```

Migration must account for concurrent writes and routing changes so requests do not lose updates or read from the wrong  location. It consumes network and database resources, so it should be throttled and monitored.

### Cross-Shard Transactions: 2PC vs Saga

A local ACID transaction normally covers one database. If a transfer debits an account on Shard 1 and credits an account on Shard 2, partial success must be handled.

- **Two-Phase Commit (2PC):** A coordinator asks participants to prepare, then commits if all are ready (otherwise aborts). It provides atomic commit semantics under its protocol assumptions, but adds coordination/latency and can block while participants or the coordinator recover.
- **Saga:** A workflow is split into local transactions. If a later step fails, compensating actions undo the business effect where possible (e.g. refund the debit). It avoids a global transaction coordinator holding a distributed transaction, but provides eventual consistency; compensation is a new action, not an instantaneous rollback, and may itself fail or need retries.

**Important:** Do not confuse a **cross-shard query** (scatter-gather and merge) with a **cross-shard transaction** (coordinating writes/atomicity).

### Availability: Sharding + Replication

Sharding splits different subsets of data; replication keeps copies of each shard.

A shard can have its own primary and replicas.

- If its primary fails, a replica may be promoted and routing updated.
- With asynchronous replication, lag can mean some acknowledged writes are absent on the promoted replica.
- Replication improves availability but does not remove failover or consistency risks.

### Common SDE-2 Follow-ups

1. **How do you choose a shard key?** Prefer a key with high cardinality and even distribution that also supports common access patterns; avoid keys that create hot partitions. Consider growth and tenant skew.
2. **Why not hash every key?** Hashing helps distribution but destroys key-range locality, making range queries fan out across shards.
3. **Does adding a shard solve a hot shard?** Not automatically. If the hot key still maps to one shard, the hotspot remains; change the partitioning strategy/key or isolate/cache the hot workload.
4. **What is the cost of cross-shard joins/queries?** They require scatter-gather and result merging, increasing network traffic, latency, resource use, and pagination complexity.
5. **How do you scale without downtime during rebalancing?** Copy data, capture/catch up concurrent writes, switch routing safely, verify, then remove the old copy; use throttling and a rollback plan.

**Interview Takeaway:** Sharding is a scale-out decision, not a default optimization. Explain why one DB is insufficient, how the shard key balances distribution against query locality, and how the design handles hot shards, cross-shard operations, rebalancing, and shard failure.
