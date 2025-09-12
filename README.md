# floQast_fintech_project


# Test Strategy Document — Staff Quality Engineer (Fintech microservices)

**System context (assumed):**  
User Service, Transaction Service, Notification Service, each with their own datastore (MongoDB, Redis), behind an API Gateway. Microservices talk REST over internal network; services deploy via CI/CD to Kubernetes (or similar). 



<img width="500" height="500" alt="fintech_arch" src="https://github.com/user-attachments/assets/e093b3c7-3b3a-43d4-8403-c133903abd13" />


---

## 1. Quality Vision & Objectives

- **Accuracy and Integrity:** Financial transactions must be processed accurately (no silent loss, duplication, or corruption). ACID-compliant database and building idempotency into its application layer.  
- **Availability & reliability:** System uptime to meet SLA's.  
- **Performance & latency:** End-user transaction flows must complete within business SLOs under expected load.  
- **Security & compliance:** Rate limiting, DDOS attacks, Captcha,OTPs enhance secuity. Encrypting PII and transaction data and complying with regulatory requirements (logging, audit).  
- **Deployability & observability:** Pushes to production are safe, automated rollbacks possible, and issues are quickly diagnosable.  
- **Maintainability & velocity:** Tests and automation should enable rapid safe releases with clear ownership.

**Key quality metrics / KPIs**
- **Transaction correctness rate:** % of transactions processed without error (target ≥ 99.999% daily).  
- **End-to-end success rate:** % of API calls that return expected success codes (target ≥ 99.9%).  
- **Latency metrics:** p50/p95/p99 latency for key endpoints (see thresholds below).  
- **Error rate:** 5xx error rate across services (target < 0.1% per hour).  
- **Availability:** System uptime (target 99.95% monthly).  
- **Mean Time to Detect (MTTD)** and **Mean Time to Recover (MTTR)** (targets: MTTD < 1 minute for critical incidents; MTTR < 15 minutes for common failure modes).  
- **Test coverage:** Unit coverage target ≥ 80% (functional-critical modules may require higher).  
- **Deployment failure rate / rollback frequency:** < 1% of deploys require rollback.  
- **Security metrics:** vulnerability count (critical/ high) = 0 in prod dependencies; SAST issues triaged within 48 hours.
---
> NOTE:
- P50 (Median) Latency: 50% of requests are faster than this value, representing the "typical" or middle experience for users.
- P95 (95th Percentile) Latency: 95% of requests are faster than this value. This metric helps to understand how the system performs for the majority of users, especially under higher load, as it starts to expose some performance outliers.
- P99 (99th Percentile) Latency: Only 1% of requests take longer than this value. This is crucial for identifying and addressing the worst-case scenarios and significant bottlenecks that affect the top 1% of users, even if they are a small group.

---
