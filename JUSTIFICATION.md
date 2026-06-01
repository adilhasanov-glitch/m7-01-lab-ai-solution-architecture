# Architectural Justification: Real-Time Fraud Scoring

## 1. Serving Pattern Selection
We selected an **Online Serving Pattern with a Streaming Feature Pipeline (Hybrid Architecture)**. 

### Why Online Serving?
The business requirements explicitly dictate that the system must score every transaction for fraud risk *before* the user sees a confirmation, operating at a peak throughput of 300 transactions/second with a strict p95 latency budget of 80 ms end-to-end. This synchronous requirement completely eliminates batch processing. 

### Why Streaming Features?
To accurately score fraud, the model requires historical context (e.g., account history) combined with fresh transaction velocities (e.g., "number of transactions in the last 10 minutes"). An Online API cannot calculate these aggregations on the fly within an 80 ms window without crippling database performance. Therefore, a streaming engine (Apache Flink) processes asynchronous event streams out-of-band to maintain pre-computed, rolling sliding-window aggregations in a low-latency cache.

## 2. Inference Infrastructure Deployment
Inference will run entirely in the **Cloud (Centralized, Multi-AZ Cluster)** rather than on the edge.

### Justification:
Fraud detection relies heavily on cross-referencing global account histories, regional blocklists, and shared behavioral patterns that cannot be stored locally on a user's mobile device due to storage constraints and data security risks. Edge deployment would expose our proprietary model weights to reverse engineering and prevent the system from recognizing cross-account coordinated attacks in real time. Running a lightweight, highly optimized framework (such as XGBoost or a compiled TensorRT model) on cloud-managed container clusters easily meets the 300 requests/second demand.

## 3. Optimization Matrix

| Metric | Status | Target / Constraint Strategy |
| :--- | :--- | :--- |
| **Latency** | **Optimize** | **Target: p95 < 80 ms.** The runtime budget allocated to the model inference itself is capped at **25 ms**, leaving 55 ms for networking overhead, API gateway routing, and downstream database lookups. |
| **Throughput** | **Optimize** | **Target: 300 requests/second at peak.** Managed container auto-scaling and an in-memory feature cache ensure scale resiliency during peak transaction windows. |
| **Cost** | **Budget Constraint** | Cost is a secondary consideration compared to security and user experience. To respect the budget, we will utilize structured, tabular models (XGBoost/LightGBM) hosted on CPU instances, entirely bypassing expensive GPU infrastructure. |

## 4. Resiliency & Fallback Strategy
When the ML model is unavailable, slow, or times out (>25 ms execution window), the API Gateway instantly drops back to a deterministic, **Rule-Based Fallback Engine**. 

- **Fallback Execution:** A hardcoded matrix evaluating basic constraints (e.g., transaction amount within typical limits, country matches billing profile) will process the request.
- **Default Action:** If the transaction passes the basic rule constraints, it defaults to **Allow** but gets flagged for an asynchronous offline human audit. If it violates baseline safety thresholds, it triggers a **Step-Up Authentication (MFA)** path rather than a hard block, protecting revenue while mitigating systemic payment risk.