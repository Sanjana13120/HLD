# How to Approach HLD Problems

> **Requirement → Scale → Problem → Solution → Trade-off**

## Approach

1. Clarify Requirements
2. Estimate Scale
3. Define APIs
4. Data Model
5. High-Level Architecture
6. Deep Dive
7. Bottlenecks & Trade-offs
8. Reliability & Observability

---

### 1. Clarify Requirements — "What are we building?"

#### Functional Requirements

- What can the user/system do?
- What are the core use cases?
- What's explicitly out of scope?

#### Non-Functional Requirements

- Scale
- Latency
- Availability
- Consistency
- Durability
- Reliability

---

### 2. Scale Estimation — "How big is this?"

Consider:

- DAU (Daily Active Users)
- Requests/day
- QPS (Queries Per Second)
- Peak QPS
- Storage/day
- Storage/year
- Bandwidth

**Example:**

```text
100M users
10M requests/day
≈ 116 QPS average
≈ 1,000 QPS peak
```

The goal is not to get perfectly accurate numbers.

The goal is to understand the scale well enough to make the right architectural decisions.

---

### 3. APIs — "How do clients interact with it?"

For each important API, consider:

- Request
- Response
- Parameters
- Errors
- Authentication/authorization where relevant
- Idempotency where relevant

**Example:**

```text
POST /shorten
GET /{shortCode}
```

---

### 4. Data Model — "What data do we need?"

Consider:

- Entities
- Relationships
- Important fields
- Indexes
- SQL vs NoSQL
- Data access patterns

**Example — URL Shortener:**

```text
URLMapping

short_code
long_url
created_at
expires_at
user_id
```

Then ask:

> **SQL or NoSQL? Why?**

The choice should be based on the system's requirements, scale, and access patterns.

---

### 5. High-Level Architecture — "What components do we need?"

Start simple.

```text
Client
   ↓
Load Balancer
   ↓
Application Servers
   ↓
Database
```

Then, based on the requirements and scale, introduce additional components only when they solve a real problem.

For example:

```text
                  ┌──→ Cache
                  │
Client → LB → App ───→ Database
                  │
                  └──→ Queue → Workers
```

For every component, ask:

> **"What problem does this solve?"**

Don't add components just because they are commonly used in system design.

---

### 6. Deep Dives — "What happens when we scale?"

#### Database is getting too many reads

Possible approaches:

- Caching
- Read replicas
- Indexing
- Sharding

#### Application servers are overloaded

Possible approaches:

- Load balancing
- Horizontal scaling
- Caching
- Async processing

#### A dependency is slow or unavailable

Possible approaches:

- Timeouts
- Retries
- Circuit breaker
- Fallback

The important question is:

> **Which solution fits this particular problem, and why?**

---

### 7. Bottlenecks & Trade-offs — "What can break and why?"

Identify potential bottlenecks.

```text
Database becomes a bottleneck
        ↓
Caching?
Read replicas?
Sharding?

Cache becomes a bottleneck
        ↓
Distributed cache?
More cache nodes?

Single database cannot handle write scale
        ↓
Sharding?
Partitioning?
```

For every major decision, understand the trade-off.

Examples:

- SQL vs NoSQL
- Strong consistency vs eventual consistency
- Synchronous vs asynchronous processing
- Read replicas vs sharding
- Cache vs database
- Simplicity vs scalability

The goal is **not** to find a perfect architecture.

The goal is to choose an architecture that satisfies the requirements and scale without unnecessary complexity.

---

### 8. Reliability & Observability — "How do we know the system is healthy and what happens when things fail?"

#### Reliability

- Failure handling
- Timeouts
- Retries
- Circuit breaker
- Replication
- Failover
- Idempotency
- Recovery

#### Observability

- Logging
- Metrics
- Monitoring
- Alerting
- Tracing where relevant

Ask:

> **"What happens if this component fails?"**

and

> **"How would we detect the problem?"**

---

## Core HLD Thinking

Don't start with technologies.

Start with the problem.

```text
Requirement
     ↓
Estimate Scale
     ↓
Identify Problem
     ↓
Choose Solution
     ↓
Understand Trade-offs
     ↓
Design for Failure
```

> **The goal is not to use every HLD concept.**
>
> **The goal is to design the simplest system that satisfies the requirements and scale.**
