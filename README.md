# floQast_fintech_project


# Test Strategy Document — Staff Quality Engineer (Fintech microservices)

**System context (assumed):**  
User Service, Transaction Service, Notification Service, each with their own datastore (MongoDB, Redis), behind an API Gateway. Microservices talk REST over internal network; services deploy via CI/CD to Kubernetes (or similar). 



<img width="500" height="500" alt="fintech_arch" src="https://github.com/user-attachments/assets/e093b3c7-3b3a-43d4-8403-c133903abd13" />


---

## 1. Quality Vision & Objectives

- **Accuracy and Integrity:** Financial transactions must be processed consistently accurately (no silent loss, duplication, or corruption). ACID-compliant database and building idempotency into its application layer.  
- **Availability & reliability:** System uptime to meet SLA's.  
- **Performance & latency:** End-user transaction flows must complete within business SLOs under expected load.  
- **Security & compliance:** Rate limiting, DDOS attacks, Captcha,OTPs enhance secuity. Encrypting PII and transaction data and complying with regulatory requirements (logging, audit).  
- **Deployability & observability:** Pushes to production are safe, automated rollbacks possible, and issues are quickly diagnosable.  
- **Maintainability & velocity:** Tests and automation should enable rapid safe releases with clear ownership.
- **Disaster Recovery:** if one of the Datacenters goes down how are we going to handle it and also reroute the traffic. 

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


## 2. Test Pyramid Strategy (distribution & justification)
- **Unit tests — 60%**  
  run on pre-commit/devloper workstation and catches any issues with internal logic. A robust business logic for fintect is of utmost importance.

- **Integration & Contract tests — 25%**  
  Mainly to test the interfaces and the contracts with DBs, and other micro services so that we dont see/catch integration issues while deploying into various environments. (Dev, feature, staging, pre-prod and prod also performance).

- **End-to-end (E2E) & UI/API flows — 10%**  
  Complete flow testing like how a user would interact. User onboarding, account details, account summary and transactions testing.

- **Non-functional tests (performance/scale/security) — 5%**  
  Performance, load, chaos/resilence and security tests will help in determining the CPU an memory usages of the machines being deployed on and also for calculating how disaster recovery scenarios will be mitigated.
---

## 3. CI/CD Quality Gates — pipeline design & pass/fail criteria

> NOTE:Environments; Dev(local), Feature(devlopers test their individual featutre on this pool/pod), staging (merged code to main branch), pre-prod(smoke tests, lnp) and prod
> Prod and Preprod will also have PCI and Retricted Non PCI zones. PCI zones will have heavy firewall restrictions and needs to be monitored and audited for ingress and egress and which microservices can access them.

### Pipeline stages & gates
1. **Pre-commit / Local**
   - Gate: **100% unit tests pass locally**; formatting, static analysis can be done here using IDE plugins.
   - Failure → block commit.

2. **Pull Request (PR) Validation**
   - Runs: unit tests, static analysis, code coverage(cobertura,jacoco and sonar), integration tests and SAST/DAST quick scan.
   - Gate criteria:
     - Unit tests: **pass 100%** (no failing tests).
     - Test coverage on new/changed files: >= **80%** (or at least not decreasing global coverage).
     - Maven dependencies added needs to be reviewed for cylic dependndancies and vulnerabilities.
     - This stage is crucial and at least 2 peers should be reviewing the code.
   - Failure → block merge.

3. **Merge / Build/ staging**
   - Runs: full unit suite, full integration tests (spinning ephemeral DB and Redis containers), contract tests, mutation/contract checks.
   - Gate criteria:
     - All tests pass.
     - Contract tests: consumer expectations satisfied (PACT/Dredd/Swagger).
     - Image build success and image scanning: zero critical vulnerabilities.
   - Failure → block deployment to staging.

4. **Pre-Production/LnP**
   - Runs: E2E smoke tests, data migration tests (if applicable), DAST, performance sanity tests, chaos tests (light).
   - Gate criteria:
     - E2E smoke: all critical flows succeed.
     - Performance: no regressions beyond a small delta (e.g., < 10% worse than baseline). See thresholds below.
     - DAST/Security: no high/critical findings untriaged.
     - Observability checks: traces, metrics, and logs available.
   - Failure → rollback or block promotion to production.

5. **Production Promotion / Canary**
   - Strategy: Gradual traffic ramp (canary or blue/green) with automated monitoring for key metrics.  
   - Canary gate criteria (during canary window, e.g., 15–30 minutes):
     - Error rate (5xx) on canary vs baseline: no > 2x increase and absolute error rate < 0.5%.  
     - Latency: p95 no more than 20% above baseline; absolute below hard SLO.  
     - Business KPI (transaction success rate): within error budget.  
   - If criteria fail → automatic rollback.

6. **Post-Deploy / Continuous Monitoring**
   - Run synthetic probes + health checks + alerting on SLO breaches.
   - Weekly/monthly security scans and dependency checks.

### Specific pass/fail criteria
- **Unit tests:** 100% pass; fail on flaky tests >1% occurrence until fixed.  
- **Coverage:** Global coverage ≥ 80%; critical modules ≥ 90%.  
- **Contract tests:** zero contract violations.  
- **Integration tests:** pass all functional scenarios.  
- **E2E smoke:** 100% critical path success.  
- **Performance gate (staging):** no 10%+ regression in p95 latency vs baseline for core APIs.  
- **Security gate:** no new high/critical vulnerabilities; unresolved medium ones must have remediation plan.  
- **Canary gate:** error rate not exceeding baseline + threshold; business transaction failure rate < 0.1% during canary.


---

## 4. Non-Functional Testing — benchmarks and requirements

> NOTE: exact numerical thresholds below are recommended starting points as seen as benchmarks for fintech companies; we need to tune them to production baselines after measuring real traffic.
> Networking will play a huge role with firewalls and ingress and egress of production (PCI and Restricted Non PCI zones)

### Performance & Scalability benchmarks
**Key measured endpoints / flows**
- `POST /transactions` (create & process transaction) — critical write path.
- `GET /transactions/{id}` — read path.
- `POST /notifications` (send notification).
- API Gateway end-to-end (auth + routing).

**Suggested SLOs / thresholds**
- **API Gateway (end-to-end)**  
  - p50 ≤ **100 ms**  
  - p95 ≤ **400 ms**  
  - p99 ≤ **800 ms**
- **Transaction Service (write path)**  
  - p50 ≤ **150 ms**  
  - p95 ≤ **800 ms** (includes DB write & downstream processing)  
  - Transaction processing completion (persisted & queued for settlement): **≤ 2s** under nominal load.
- **Notification Service (enqueue/send)**  
  - p95 ≤ **300 ms** for enqueuing; external delivery may be async.
- **Throughput targets (example baseline)**  
  - Sustained throughput: **200 tps** (transactions/sec) for business peak.  
  - Burst handling: tolerate **1000 tps** bursts for up to 60 seconds with graceful degradation.
- **Error budget / availability**  
  - Availability target: **99.95%** (approx 22 minutes downtime/month).  
  - Error budget configured per monthly window; defenses (rate limit, graceful degradation) when budget low.

**Load & stress testing scenarios**
- **Nominal load test:** run steady 1x expected peak for 1+ hour to verify stability.  
- **Peak load test:** run 1.5–2x expected peak for 30 minutes to verify autoscaling and queuing.  
- **Soak test:** run nominal load for 6–24 hours to surface memory leaks, connection leaks, stateful issues.  
- **Spike test:** sudden jump to 3–5x peak for short windows to validate graceful degradation and circuit breakers.  
- **Chaos/resilience test:** simulate instance failures, network partitions, DB failovers; verify bounded error, rollovers, retries.

**Observability during tests**
- Collect metrics: latency histograms, error rates, throughput, CPU/mem, DB latency & queue lengths.  
- Tracing: distributed traces to find tail latency contributors.  
- Logs: structured logs with correlation IDs.  
- Alerts: set temporary thresholds for tests but use production alerting config for realistic responses.

### Reliability & Fault-Tolerance
- **Idempotency:** Transaction create endpoint idempotent with client-provided idempotency key (verify with integration tests).  
- **Exactly-once / at-least-once guarantees:** define for each flow (e.g., transaction must be exactly-once; notification may be at-least-once) and test accordingly.  
- **Retries & backoff:** Clients should observe circuit breaker patterns; simulate downstream slowness and verify correct backoff.  
- **DB failover:** Tests that simulate Mongo primary failover; ensure app reconnects and data integrity remains.
- **Disaster recovery:** Tests that simulate if datacenters go down.

### Security & Compliance testing
- **SAST** (static analysis) in PRs and builds.  
- **DAST** (dynamic scans) in staging for common OWASP Top 10 vulnerabilities.  
- **Pen-testing / Vulnerability assessments** scheduled periodically and after major changes.  
- **Encryption & data protection** tests: confirm TLS enforced, data at rest encryption, proper key access controls.  
- **Access control tests:** RBAC/permission matrix tests for protected endpoints.

### Observability & Monitoring tests
- Validate that traces and metrics include:
  - Request correlation ID through API Gateway -> services.  
  - Key business metrics exported: transaction count, average processing time, failed transactions.   

---
