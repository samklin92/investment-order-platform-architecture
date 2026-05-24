# Investment Order Service — Deployment Architecture & Technical Specification

**Author:** Igwe Ogaji Samuel — Cloud & DevOps Engineer  
**GitHub:** github.com/samklin92  
**Repository:** github.com/samklin92/investment-order-platform-architecture  
**Version:** 1.0  
**Date:** May 2026  
**Status:** Production-Ready Design

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Service Overview](#2-service-overview)
3. [Compute Architecture](#3-compute-architecture)
4. [Database Architecture](#4-database-architecture)
5. [Failover & Disaster Recovery](#5-failover--disaster-recovery)
6. [Backup Strategy](#6-backup-strategy)
7. [Security Architecture](#7-security-architecture)
8. [Observability Stack](#8-observability-stack)
9. [CI/CD Pipeline](#9-cicd-pipeline)
10. [Proposed SLOs](#10-proposed-slos)
11. [Cost Considerations](#11-cost-considerations)

---

## 1. Executive Summary

This document defines the deployment architecture for the Investment Order Service — a critical microservice responsible for processing, matching, and managing all investment orders across the platform. The architecture is designed to meet financial-grade reliability standards: 99.95% availability, sub-150ms P99 latency, zero data loss (RPO ≤ 5 minutes), and full regulatory compliance.

The design is built on AWS, using Amazon EKS for compute orchestration, Aurora PostgreSQL Multi-AZ for persistence, ElastiCache Redis for low-latency caching, and a warm standby cross-region DR strategy. Every component is automated, observable, and reproducible from infrastructure-as-code.

---

## 2. Service Overview

### What the Service Does

The Investment Order Service is the core transaction engine of the platform. It handles:

- **Order ingestion** — receiving buy/sell orders from Web, Mobile, and Partner/Broker APIs
- **Order matching** — routing orders through the Matching Engine microservice
- **Order management** — tracking order lifecycle from submission to settlement
- **Trade processing** — publishing confirmed trades to downstream services via SQS/SNS
- **Compliance** — maintaining full audit trails for regulatory reporting

### Why It Is Critical

- Directly handles user funds and investment transactions
- Any downtime causes immediate financial and reputational loss
- Subject to financial regulatory requirements (audit logs, data retention, encryption at rest and in transit)
- High-frequency, low-latency workload — orders must be processed in milliseconds

### Traffic Profile

- Peak load during market open/close windows (09:30 and 16:00 EST)
- Burst capacity requirement: 10x baseline during major market events
- Target throughput: 50,000+ orders per minute at peak

---

## 3. Compute Architecture

### Platform: Amazon EKS — Multi-AZ

All microservices run on Amazon EKS across three Availability Zones (us-east-1a, 1b, 1c) in the primary region (us-east-1), with a warm standby cluster in the secondary region (us-east-2).

### Node Groups

| Node Group | Instance Type | Purpose | Min Nodes | Max Nodes |
|---|---|---|---|---|
| System | m6i.large | Cluster infrastructure (CoreDNS, metrics-server) | 3 | 3 |
| Application | c6i.2xlarge | Order processing microservices | 3 | 30 |
| Matching Engine | c6i.4xlarge | High-CPU order matching workloads | 3 | 15 |
| Memory-Optimised | r6i.2xlarge | Cache-heavy shared services | 2 | 10 |

All node groups span all three AZs. Cluster Autoscaler manages scale-out/in based on pending pod pressure.

### Microservices

**Application Layer — Stateless**

- **Order API Service** — REST/gRPC endpoint for all order ingestion. Stateless, horizontally scalable. HPA targets 60% CPU utilisation with a minimum of 3 replicas (one per AZ).
- **Matching Engine** — Core order matching logic. CPU-intensive. Runs on dedicated compute-optimised nodes with resource requests equal to limits to guarantee QoS class Guaranteed.
- **Order Management Service** — Tracks order lifecycle, emits status events to SNS.

**Shared Services**

- **Auth Service** — JWT validation and session management, backed by Redis cache
- **Risk Service** — Synchronous pre-trade risk check on every order (position limits, exposure limits)
- **Notification Service** — Publishes order updates to downstream consumers via SNS
- **Reference Data Service** — Serves static market data (symbols, prices, trading rules)

**Async Workers — Event-Driven**

- **Trade Processing Service** — Consumes from SQS, persists confirmed trades to Aurora
- **Position Service** — Updates portfolio positions post-trade
- **Portfolio Service** — Aggregates position data for reporting
- **Compliance Service** — Writes immutable audit records to S3 and OpenSearch

### Traffic Ingress Path

```
Users / Partners / Broker APIs
         ↓
  Route 53 (latency-based routing + health checks)
         ↓
  AWS WAF (rate limiting, DDoS protection, SQL injection filtering)
         ↓
  CloudFront (TLS termination, edge caching)
         ↓
  API Gateway (request validation, throttling, auth)
         ↓
  EKS Ingress Controller (ALB + NLB)
         ↓
  Kubernetes Services → Pods
```

### Autoscaling Policy

```yaml
# Applied to all stateless services
minReplicas: 3          # One per AZ — never below this
maxReplicas: 50         # Full burst capacity
targetCPUUtilizationPercentage: 60
# Custom metric — scale when order throughput exceeds threshold
targetOrdersPerSecond: 1000
```

### Pod Disruption Budgets

All services have PDBs to maintain availability during node drain or rolling updates:

```yaml
minAvailable: 2   # At least 2 pods always running per service
```

---

## 4. Database Architecture

### Primary Store: Amazon Aurora PostgreSQL — Multi-AZ

Aurora PostgreSQL is selected over standard RDS for automatic failover in under 30 seconds, up to 15 read replicas with sub-10ms replication lag, storage auto-scaling to 128TB, and native PITR support.

**Cluster Layout:**

| Role | Instance Type | AZ | Purpose |
|---|---|---|---|
| Writer | db.r6g.2xlarge | us-east-1a | All write operations |
| Reader 1 | db.r6g.2xlarge | us-east-1b | Application read scaling |
| Reader 2 | db.r6g.2xlarge | us-east-1c | Application read scaling |
| Reader 3 | db.r6g.xlarge | us-east-1a | Analytics and reporting queries |

- Encryption at rest: AES-256 via AWS KMS
- Encryption in transit: TLS 1.2+ enforced
- Automated backups: 35-day retention with PITR enabled
- Performance Insights and Enhanced Monitoring (1-second granularity) enabled

### Caching Layer: Amazon ElastiCache for Redis — Multi-AZ

Redis absorbs read traffic for session tokens, reference data, order status lookups, and short-TTL risk check results.

- Cluster mode: 3 shards × 2 nodes (1 primary + 1 replica per shard)
- Automatic failover: replica promoted in under 60 seconds
- Encryption in-transit and at-rest enabled
- Eviction policy: `allkeys-lru`

### Async Messaging: Amazon SQS + SNS

Order events fan out from SNS to dedicated SQS queues per consumer:

| Queue | Type | Purpose |
|---|---|---|
| trade-processing-queue | Standard | Confirmed trade persistence |
| position-update-queue | Standard | Portfolio position updates |
| compliance-audit-queue | FIFO | Ordered, exactly-once audit logging |
| notification-queue | Standard | User/partner notifications |

FIFO queue is used for compliance to guarantee ordering and exactly-once processing — critical for audit integrity.

---

## 5. Failover & Disaster Recovery

### Regional Failover Strategy: Warm Standby

A warm standby environment runs continuously in us-east-2. It is not idle — it runs at reduced capacity (20–30% of production) and can scale to full production within 30 minutes.

**Primary Region:** us-east-1 (active, full traffic)  
**Standby Region:** us-east-2 (warm, scaled-down, receiving replicated data)

### Failover Components

| Component | Failover Mechanism | RTO |
|---|---|---|
| EKS Compute | Standby cluster scales up via Cluster Autoscaler | < 30 min |
| Aurora PostgreSQL | Global Database cross-region replica promoted | < 1 min |
| ElastiCache Redis | Warm standby cluster pre-provisioned | < 5 min |
| DNS | Route 53 health checks trigger weighted failover | < 60 sec |
| SQS/SNS | Regional services — standby queues pre-configured | Immediate |

### Aurora Global Database

Aurora Global Database replicates from us-east-1 to us-east-2 with typical lag under 1 second. In a regional failure:

1. Route 53 health checks detect us-east-1 degradation
2. Automated runbook triggers Aurora Global Database failover (promotes us-east-2 replica to writer)
3. DNS weighted routing shifts 100% traffic to us-east-2
4. Standby EKS cluster scales to full production capacity
5. Operations team notified via PagerDuty

### RPO / RTO Targets

| Metric | Target | How Achieved |
|---|---|---|
| RPO (Recovery Point Objective) | ≤ 5 minutes | Aurora Global Database replication lag < 1s; PITR as fallback |
| RTO (Recovery Time Objective) | ≤ 30 minutes | Warm standby + automated DNS failover + pre-scaled cluster |

---

## 6. Backup Strategy

### Database Backups

| Backup Type | Frequency | Retention | Storage |
|---|---|---|---|
| Aurora Automated Backup | Continuous (PITR) | 35 days | S3 (managed by Aurora) |
| Aurora Manual Snapshot | Daily | 90 days | S3 |
| Cross-Region Snapshot | Daily | 90 days | S3 in us-east-2 |
| Aurora Global Database | Continuous replication | Live | us-east-2 Aurora cluster |

### Application State Backups

| Asset | Backup Method | Frequency | Retention |
|---|---|---|---|
| Kubernetes manifests | Git (GitHub) | On every change | Indefinite |
| Terraform state | S3 + DynamoDB lock | On every apply | Versioned indefinitely |
| Secrets | AWS Secrets Manager | Versioned automatically | 90 days |
| Audit logs | S3 (compliance bucket) | Real-time streaming | 7 years (regulatory) |
| Application logs | OpenSearch + S3 | Real-time streaming | 90 days hot, 7 years cold |

### Backup Validation

Backups are tested monthly via automated restore drills:
- Aurora snapshot restored to isolated test environment
- Data integrity checks run against known checksums
- Restore time measured and compared against RTO targets
- Results logged and reviewed by the platform team

---

## 7. Security Architecture

### Network Security

- All EKS worker nodes run in **private subnets** — no direct internet access
- **Calico NetworkPolicies** enforce pod-level microsegmentation — services only communicate with explicitly permitted peers
- **Istio Service Mesh** with STRICT mTLS mode — all service-to-service traffic encrypted, unauthenticated calls rejected
- **AWS WAF** at the edge — protects against OWASP Top 10, rate limiting per IP and per API key
- **VPC Flow Logs** enabled — all network traffic logged to S3 for audit

### Identity & Access

- **IAM Roles for Service Accounts (IRSA)** — each microservice has its own scoped IAM role, no shared credentials
- **Least-privilege IAM policies** — each role grants only the specific AWS API actions required
- **No long-lived credentials** — all access is via IRSA or AWS SSO
- **AWS Secrets Manager** — all database credentials, API keys, and TLS certificates stored and rotated automatically

### Data Security

- Aurora: AES-256 encryption at rest via customer-managed KMS key
- S3 audit buckets: SSE-KMS encryption, versioning enabled, MFA delete enabled
- ElastiCache: in-transit and at-rest encryption enabled
- All inter-service communication: TLS 1.2+ enforced via Istio

### Compliance Controls

- **Immutable audit trail** — all order events written to FIFO SQS → Compliance Service → S3 (with Object Lock) and OpenSearch
- **CloudTrail** enabled across all regions — logs all AWS API calls
- **AWS Config** — continuous compliance monitoring against defined rules
- **Automated security scanning** in CI/CD pipeline — Trivy for container image scanning, Checkov for Terraform IaC scanning

---

## 8. Observability Stack

### Metrics

- **Prometheus** — scrapes all Kubernetes workloads, node exporters, and custom application metrics
- **Grafana** — dashboards for infrastructure health, order throughput, error rates, and latency distribution
- **AWS CloudWatch** — Aurora performance metrics, EKS control plane metrics, ALB request metrics

### Distributed Tracing

- **AWS X-Ray** — end-to-end request tracing across all microservices
- Trace sampling rate: 5% normal load, 100% on errors

### Log Management

- **Amazon OpenSearch** — centralised log aggregation and full-text search
- **Fluent Bit** — lightweight log shipping from all pods to OpenSearch
- **S3** — long-term log archival (90 days hot in OpenSearch, 7 years cold in S3 Glacier)

### Alerting

All alerts route through Alertmanager → PagerDuty for on-call escalation.

| Alert | Threshold | Severity |
|---|---|---|
| Order API error rate | > 1% over 5 minutes | Critical |
| P99 latency | > 150ms over 5 minutes | Critical |
| Pod availability | Any service below minAvailable | Critical |
| Aurora writer CPU | > 80% over 10 minutes | Warning |
| Aurora replication lag | > 30 seconds | Critical |
| Redis memory utilisation | > 85% | Warning |
| SQS queue depth | > 10,000 messages | Warning |
| Certificate expiry | < 30 days | Warning |

### Key SLIs Tracked

- **Availability** — percentage of successful HTTP responses (non-5xx) over rolling 30-day window
- **Latency** — P50, P95, P99 of order API response time
- **Error rate** — percentage of failed order submissions
- **Throughput** — orders processed per minute
- **Data freshness** — Aurora replication lag to read replicas and standby region

---

## 9. CI/CD Pipeline

### Pipeline: GitHub Actions → ArgoCD

```
Developer pushes to feature branch
         ↓
GitHub Actions triggers:
  1. Unit & integration tests
  2. Trivy container image security scan
  3. Checkov Terraform IaC scan
  4. Build Docker image
  5. Push to Amazon ECR
  6. Update Kustomize image tag in GitOps repo
         ↓
ArgoCD detects Git change
         ↓
ArgoCD syncs to Kubernetes cluster
         ↓
Argo Rollouts executes canary deployment:
  - 20% traffic → new version
  - Monitor error rate and latency for 10 minutes
  - 50% traffic if healthy
  - 100% traffic if healthy
  - Automatic rollback if error rate > 1% or P99 > 150ms
```

### Deployment Strategy

- **Production:** Canary via Argo Rollouts (20% → 50% → 100%) with automated analysis
- **Staging:** Rolling update with full test suite gate
- **Dev:** Direct sync, no gates

### Rollback

Rollback is automatic if analysis steps fail during canary. Manual rollback is a single Git revert — ArgoCD reconciles within 3 minutes.

---

## 10. Proposed SLOs

### Why These SLOs

SLOs for a financial order service must be set based on three factors: user impact, regulatory obligations, and operational feasibility. Each SLO below is tied directly to a business or compliance requirement — not arbitrary targets.

---

### SLO 1 — Availability: 99.95%

**Target:** 99.95% of order API requests return a successful response (HTTP 2xx or 4xx client errors are not counted as failures) over a rolling 30-day window.

**Allowable downtime:** ~21.9 minutes per month

**Why 99.95% and not 99.99%:**
- 99.99% requires near-zero planned maintenance windows and extremely expensive redundancy at every layer
- 99.95% is achievable with the multi-AZ EKS setup, Aurora Multi-AZ, and warm standby — without over-engineering
- The gap between 99.95% and 99.99% is 17 minutes per month — meaningful for a financial service but manageable with proper incident response
- If the platform matures and evidence shows we consistently exceed 99.95%, the SLO can be raised to 99.99%

**Error Budget:** 0.05% = ~21.9 minutes/month of allowable downtime before SLO is breached

---

### SLO 2 — Latency: P99 < 150ms

**Target:** 99% of order API requests complete in under 150 milliseconds, measured end-to-end at the API Gateway.

**Why 150ms:**
- Investment order placement is latency-sensitive — users and partner systems expect near-instant confirmation
- 150ms P99 is achievable with ElastiCache Redis absorbing read traffic, Aurora read replicas for non-write queries, and EKS pods co-located in the same AZ as the database writer
- P50 target is < 30ms, P95 target is < 80ms — 150ms P99 accounts for tail latency from occasional cold starts, GC pauses, or transient network jitter
- Setting P99 below 100ms would require significantly more expensive infrastructure and tighter resource guarantees across all services

---

### SLO 3 — Error Rate: < 0.1%

**Target:** Fewer than 0.1% of order submissions result in a server-side error (HTTP 5xx) over a rolling 24-hour window.

**Why 0.1%:**
- Any server error on an order submission is a direct user-facing failure — the user's order was not placed
- 0.1% at 50,000 orders/minute = 50 failed orders per minute at peak — a threshold that triggers immediate investigation
- Client errors (4xx — invalid order parameters, insufficient funds) are excluded as they are expected and handled gracefully

---

### SLO 4 — Durability: RPO ≤ 5 minutes

**Target:** In the event of a regional failure, no more than 5 minutes of committed order data is lost.

**Why 5 minutes:**
- Aurora Global Database replication lag is typically under 1 second — RPO of 5 minutes is a conservative, safe target that accounts for extreme failure scenarios
- Financial regulations require strict data durability guarantees — 5 minutes is defensible from a compliance standpoint
- Zero RPO (synchronous replication) would require active-active multi-region write architecture, which significantly increases complexity and cost

---

### SLO 5 — Recovery Time: RTO ≤ 30 minutes

**Target:** Full service restoration within 30 minutes of a confirmed regional failure.

**Why 30 minutes:**
- Warm standby in us-east-2 can scale to full production in 15–20 minutes
- Aurora Global Database failover completes in under 1 minute
- Route 53 DNS failover completes in under 60 seconds
- The 30-minute target includes time for automated failover + human verification + post-failover smoke tests
- A tighter RTO (e.g. 5 minutes) would require active-active architecture with synchronous cross-region writes — a significant architectural uplift for marginal gain

---

### SLO Summary Table

| SLO | Metric | Target | Error Budget |
|---|---|---|---|
| Availability | Successful request rate | 99.95% / 30 days | 21.9 min/month |
| Latency | P99 response time | < 150ms | 1% of requests |
| Error Rate | Server-side errors (5xx) | < 0.1% / 24h | 50 errors/min at peak |
| Durability (RPO) | Data loss on regional failure | ≤ 5 minutes | — |
| Recovery (RTO) | Time to full restoration | ≤ 30 minutes | — |

---

### Error Budget Policy

- **Error budget > 50% remaining:** Normal deployment velocity, canary releases permitted
- **Error budget 25–50% remaining:** Increased canary analysis window (20 minutes instead of 10), change freeze on non-critical services
- **Error budget < 25% remaining:** Full change freeze except critical security patches. Engineering focus shifts to reliability improvements.
- **Error budget exhausted:** Incident review required before any new deployments. SLO review with stakeholders.

---

## 11. Cost Considerations

The following cost optimisations are built into the architecture without compromising reliability:

- **Spot Instances** for non-critical worker nodes (analytics, reporting) — up to 70% cost reduction on those node groups
- **Aurora Auto Pause** disabled in production (always-on for critical path), enabled for dev/staging
- **S3 Intelligent Tiering** for audit logs — automatically moves infrequently accessed data to cheaper storage tiers
- **Reserved Instances** for base EKS node groups and Aurora writer — 1-year commitment for ~40% saving over on-demand
- **CloudFront caching** reduces origin hits and lowers API Gateway costs for cacheable reference data endpoints

---

*This specification is maintained as living documentation. All infrastructure defined herein is codified in Terraform and available at github.com/samklin92/investment-order-platform-architecture.*
