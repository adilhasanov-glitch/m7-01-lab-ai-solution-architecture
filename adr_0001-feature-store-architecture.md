# ADR 0001: Separation of Online Cache and Offline Data Warehouse via a Managed Feature Store

## Context
Our fraud detection model requires low-latency feature retrieval (e.g., account profile data, device fingerprint status) during the synchronous inference path, which has a strict 80 ms end-to-end budget. Concurrently, data scientists require access to months of historical, raw transaction logs to train updated iterations of the model. 

Querying our analytical data warehouse directly during a live payment transaction introduces untenable latency (often >1 second), while storing terabytes of historical analytical data in an operational database like Redis is financially and operationally impractical. This friction forces a structural decision regarding how features are stored, updated, and served consistently between training and inference environments.

## Decision
We will implement an automated **Dual-Database Feature Store architecture managed by Feast**. 

This system splits data into two distinct operational tiers under a unified API schema framework:
1. **An Online Tier (Redis):** A high-performance, in-memory, key-value store optimized exclusively for point-in-time, ultra-low latency (<5 ms) feature lookups during live transaction scoring.
2. **An Offline Tier (Snowflake):** A columnar analytical data warehouse storing historical data, used exclusively for model training, batch evaluation, and feature backtesting.

The synchronization and feature definition consistency between these two tiers will be managed explicitly by the Feast registry to avoid train/serve skew.

## Alternatives Rejected

- **Alternative 1: Direct Warehouse Querying (Snowflake only)**
  - *Reason for Rejection:* Snowflake is optimized for analytical throughput, not operational latency. Point queries under heavy load cannot consistently meet our p95 80 ms SLA and would result in massive transaction drop-offs.
- **Alternative 2: Unified In-Memory Storage (Redis only)**
  - *Reason for Rejection:* Storing hundreds of millions of historical rows with hundreds of columns in-memory would incur catastrophic cloud infrastructure costs. Redis also lacks the analytical querying capabilities needed to join large-scale datasets for model training.

## Consequences

- **The Good:**
  - Guarantees ultra-low latency feature lookups (<5 ms) on the critical payment path.
  - Eliminates train/serve data skew because both the online production API and offline training pipelines share identical feature definitions from a single registry.
  - Keeps operational costs under control by keeping the Redis footprint minimal (storing only active keys and rolling real-time aggregations).
- **The Bad:**
  - Introduces structural complexity by requiring a sync pipeline to keep the online cache updated with offline database changes.
  - Adds operational overhead to manage, monitor, and scale two entirely separate database technologies.

## Revisit if
We will reopen this decision if cloud providers release unified operational/analytical hybrid databases (HTAP architectures) that can guarantee sub-5 ms point lookups at a scale of 300+ queries/second, while simultaneously running massive analytical queries over terabytes of training data without resource contention.