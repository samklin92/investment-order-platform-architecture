# Investment Order Platform Architecture

Production-grade cloud-native architecture for a highly available financial investment order platform designed with Kubernetes, Terraform, AWS, observability, disaster recovery, and multi-AZ resilience.

---

# Architecture Diagram

![Architecture](architecture/investment-order-architecture.png)

---

# Key Features

- Multi-AZ Kubernetes deployment
- PostgreSQL RDS with automatic failover
- Redis replication and persistence
- Horizontal Pod Autoscaling
- Disaster recovery strategy
- Observability with Prometheus/Grafana
- Secure secret management
- GitOps-ready deployment workflow

---

# Reliability Targets

| Objective | Target |
|---|---|
| Availability | 99.95% |
| P99 Latency | < 1 second |
| RPO | ≤ 5 minutes |
| RTO | ≤ 30 minutes |

---

# Technology Stack

- AWS EKS
- Terraform
- PostgreSQL RDS
- Redis
- Kubernetes
- Prometheus + Grafana
- GitHub Actions
- OpenTelemetry

---

# Project Goals

This project demonstrates production-grade infrastructure design principles for a mission-critical financial platform including:

- High availability
- Fault tolerance
- Scalability
- Observability
- Disaster recovery
- Security best practices

---

# Future Improvements

- GitOps with ArgoCD
- Service mesh integration
- Multi-region active-active deployment
- Chaos engineering testing