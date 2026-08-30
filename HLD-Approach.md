# How to approach HLD Problems

> Requirement → Scale → Problem → Solution → Trade-off

## Approach
1. Clarify Requirements
2. Estimate Scale
3. Define APIs
4. Data Model
5. High-Level Architecture
6. Deep Dive
7. Bottlenecks & Trade-offs
8. Reliability & Observability

### 1. Clarify Requirements: "What are we building"?
#### Functional Requirements:
    What can the user/system do?
    What are the core use cases?
    What's explicitly out of scope?

#### Non-Functional Requirements:
    Scale
    Latency
    Availability
    Consistency
    Durability
    Reliability

### 2. Scale Estimation — "How big is this?"
    DAU
    Requests/day
    QPS
    Peak QPS
    Storage/day
    Storage/year
    Bandwidth

For example: 

    100M users
    10M requests/day
    ≈ 116 QPS average
    ≈ 1,000 QPS peak

### 3. APIs — "How do clients interact with it?"
    • request
    • response
    • parameters
    • errors
    • idempotency where relevant
For eg:

    POST /shorten
    GET /{shortCode}

### 4. Data Model — "What data do we need?"
    Entities
    relationship
    indexes
    sql/nosql
For URL shortener:

    URLMapping 

    short_code
    long_url
    created_at
    expires_at
    user_id
Then decide: SQL or NoSQL?

### 5. High-Level Architecture — "What components do we need?"

    Client
       ↓
    Load Balancer
       ↓
    Application Servers
       ↓
    Database
Then, based on the scale/requirements, you may introduce:
Every box should answer: "What problem does this solve?"

             ┌── Cache
             │
    Client → LB → App → DB
             │
             └── Queue → Workers   

### 6. Deep Dives — "What happens when we scale?"
"Your database is getting too many reads. What do you do?"

    Caching
    Read replicas
    Sharding
    Indexing

"Your application servers are overloaded."

    Load balancing
    Horizontal scaling
    Caching
    Async processing

"One dependency is down."

    Timeouts
    Retries
    Circuit breaker
    Fallback

### 7. Bottlenecks + Trade-offs — "Why this solution instead of another?"
    DB becomes bottleneck → Cache?
    Cache becomes bottleneck → Distributed cache?
    Single DB cannot scale writes → Sharding?

### 8. Reliability + Observability — "How do we know this system is healthy and what happens when things fail?"

    ├── Failure handling
    ├── Idempotency
    ├── Monitoring
    └── Recovery

    Retries
    Timeouts
    Circuit breaker
    Replication
    Failover
    Idempotency
    Logging
    Metrics
    Monitoring
    Alerting