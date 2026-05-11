# SwiftEats Scaling Analysis

One-line: Analysis and redesign to prevent catastrophic failure during a 14M-user promotional spike.

## The Scenario

World Cup Final promo: 180M users notified, ~8% CTR → ~14.4M users opening the app concentrated in 60s (spike). The repo models the failing monolith and a redesigned, scalable architecture to safely handle 10M concurrent users.

## Document Summary

| Document                  | Contents                                                                                      |
| ------------------------- | --------------------------------------------------------------------------------------------- |
| `docs/ARCHITECTURE.md`    | Current monolith analysis + redesigned multi-tier architecture and component justifications   |
| `docs/FAILURE-CASCADE.md` | Traffic math, capacity numbers, detailed failure triggers and a T+0–T+2h timeline             |
| `docs/COST-ESTIMATE.md`   | Real AWS cost estimate (baseline + 4-hour peak) with line-item math and ROI justification     |
| `docs/RUNBOOK.md`         | 5-step incident runbook: detect, triage, respond, rollback, postmortem — for oncall engineers |

## Key Findings (top surprises)

- The DB pool (max_connections = 100) is the hard cap; at 1.2M peak RPS we need ~144,000 concurrent connections (impossible).
- Synchronous payments (500ms hold) can consume tens of thousands of DB connections: 60k payments/s × 0.5s → 30k connections held.
- Node.js OOM occurs quickly: ~15,000 queued requests in memory can reach the 4GB heap limit and crash the process within seconds.
- Static asset traffic can demand ~57.6 Gbps; a single t3.medium NIC (1 Gbps) is ~57x undersized without a CDN.

## Architecture Overview

The redesign splits responsibilities: CloudFront serves all static assets; an ALB routes traffic to an auto-scaled fleet of Node.js instances; Redis caches hot reads and provides atomic counters for promos; PgBouncer multiplexes DB connections to the RDS primary and read replicas; payment requests are enqueued to SQS and processed by a separate pool of payment workers to avoid blocking the API.

## Tech Stack Context

Node.js (API), PostgreSQL (RDS), Redis (ElastiCache), AWS services: EC2, ALB, CloudFront, RDS, ElastiCache, SQS. The repo contains analysis, diagrams, runbook, and cost estimates for this stack.

---

If you want, I can now run `git add`, `git commit`, and `git push` these changes. Would you like me to proceed? (I can also run the commands now.)
