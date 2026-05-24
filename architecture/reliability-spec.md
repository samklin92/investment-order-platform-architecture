# Reliability Specification

## Availability Goal
99.95%

## Database Strategy
PostgreSQL RDS Multi-AZ with automatic failover.

## Backup Strategy
- Automated snapshots
- Cross-region replication
- Point-in-time recovery

## Observability
- Prometheus
- Grafana
- OpenTelemetry
- Centralized logging

## Disaster Recovery
- RPO ≤ 5 minutes
- RTO ≤ 30 minutes