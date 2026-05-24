\# Investment Order Service — High Level Architecture

> Highly available, secure, scalable, and fault-tolerant



\## Architecture Diagram

!\[Investment Order Service - High Level Architecture](./architecture/AWS%20investment%20order%20service%20architecture.png)



\## Structure

investment-order-platform-architecture/

├── architecture/        # Architecture diagrams and specs

├── docs/                # Documentation

├── kubernetes/          # Kubernetes manifests

├── observability/       # Monitoring and alerting configs

└── terraform/           # Infrastructure as Code

## Non-Functional Requirements

\- \*\*Availability:\*\* 99.95% uptime

\- \*\*Scalability:\*\* Auto-scale based on load

\- \*\*Durability:\*\* No data loss (RPO ≤ 5 min)

\- \*\*Latency:\*\* P99 < 150ms

\- \*\*Security:\*\* End-to-end encryption

\- \*\*Compliance:\*\* Audit logs, data retention



\## Tech Stack

AWS · EKS · Terraform · Aurora PostgreSQL · ElastiCache Redis · 

Amazon SQS · SNS · CloudFront · WAF · API Gateway · 

Prometheus · Grafana · AWS X-Ray · Amazon OpenSearch

