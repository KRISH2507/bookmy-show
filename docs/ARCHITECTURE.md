# System Architecture

## Current Architecture Diagram

```
┌──────────────┐
│   Internet   │
└──────┬───────┘
       │
┌──────▼───────────────────────────────────────────────┐
│  CloudFront CDN (Static Assets + API Cache)          │
└──────┬───────────────────────────────────────────────┘
       │
┌──────▼──────────┐
│  Route 53 DNS   │
└──────┬──────────┘
       │
┌──────▼──────────────────────────────────────────────┐
│  Application Load Balancer (ALB)                    │
│  - SSL Termination                                  │
│  - Health Checks                                    │
└──────┬──────────────────────────────────────────────┘
       │
┌──────▼────────────────────────────────────────────┐
│  Auto Scaling Group (6x t3.medium)                │
│  ┌──────────────────────────────────────────────┐ │
│  │  Node.js API Servers (Express)               │ │
│  │  - Booking API                               │ │
│  │  - Event API                                 │ │
│  │  - User API                                  │ │
│  └──────────────────────────────────────────────┘ │
└───┬────────────────────────┬─────────────────────┘
    │                        │
    │ ┌──────────────────────▼───────────────────┐
    │ │  ElastiCache Redis Cluster               │
    │ │  (cache.r6g.large - 3 nodes)             │
    │ │  - Event cache                           │
    │ │  - Seat count cache                      │
    │ │  - Distributed locks                      │
    │ └──────────────────────────────────────────┘
    │
    │ ┌──────────────────────────────────────────┐
    └─► RDS PostgreSQL (db.r5.xlarge)            │
      │  - Primary (writes)                      │
      │  - Read Replicas (2x db.t4g.large)       │
      └──────────────────────────────────────────┘

┌──────────────────────────────────────────────────┐
│  Async Payment Processing                        │
│  ┌────────────┐     ┌──────────────┐            │
│  │ SQS Queue  │────►│ Lambda Worker │            │
│  │ (Standard) │     │ (Node.js)     │            │
│  └────────────┘     └───────┬──────┘            │
│                             │                    │
│                     ┌───────▼────────────┐       │
│                     │ Payment Gateway    │       │
│                     │ (Stripe/Razorpay)  │       │
│                     └────────────────────┘       │
└──────────────────────────────────────────────────┘
```

---

## Target Architecture Diagram

```
                    ┌─────────────────┐
                    │   Internet      │
                    └────────┬────────┘
                             │
              ┌──────────────▼──────────────┐
              │  CloudFront (Global CDN)    │
              │  - Edge caching             │
              │  - DDoS protection          │
              └──────────────┬──────────────┘
                             │
              ┌──────────────▼──────────────┐
              │  ALB (us-east-1)            │
              │  - Health checks every 10s  │
              └──────────────┬──────────────┘
                             │
         ┌───────────────────┼────────────────────┐
         │                   │                    │
    ┌────▼────┐         ┌────▼────┐        ┌────▼────┐
    │ t3.med  │         │ t3.med  │        │ t3.med  │
    │ Node.js │         │ Node.js │  ...   │ Node.js │
    └────┬────┘         └────┬────┘        └────┬────┘
         │                   │                    │
         └───────────────────┼────────────────────┘
                             │
          ┏━━━━━━━━━━━━━━━━━━┻━━━━━━━━━━━━━━━━━━┓
          ┃                                      ┃
   ┌──────▼────────┐                   ┌─────────▼────────┐
   │ Redis Cluster │                   │ RDS PostgreSQL   │
   │ cache.r6g.lg  │                   │ db.r5.xlarge     │
   │               │                   │                  │
   │ [Locks+Cache] │                   │ [Primary Write]  │
   └───────────────┘                   └─────────┬────────┘
                                                 │
                                      ┌──────────┴──────────┐
                                      │                     │
                              ┌───────▼──────┐      ┌──────▼──────┐
                              │ Read Replica │      │Read Replica │
                              │ db.t4g.large │      │db.t4g.large │
                              └──────────────┘      └─────────────┘

   ┌────────────────────────────────────────────────────────┐
   │  Background Processing Pipeline                        │
   │                                                        │
   │  ┌──────────┐      ┌──────────────┐                  │
   │  │   SQS    │─────►│ Lambda Worker│──┐               │
   │  │  Queue   │      │   (Pool: 50) │  │               │
   │  └──────────┘      └──────────────┘  │               │
   │       │                               │               │
   │       │ (DLQ)                         │               │
   │  ┌────▼─────┐                         │               │
   │  │ SQS DLQ  │                         │               │
   │  │ (Manual) │              ┌──────────▼─────────────┐ │
   │  └──────────┘              │ Payment Gateway API    │ │
   │                            │ (External Service)     │ │
   │                            └────────────────────────┘ │
   └────────────────────────────────────────────────────────┘
```

---

## Component Table

| Component | Purpose | Scaling Strategy |
|-----------|---------|------------------|
| **CloudFront** | CDN for static assets + API response caching | Global edge locations (auto-scaled) |
| **ALB** | Load distribution + SSL termination | AWS-managed, auto-scales |
| **EC2 Auto Scaling** | API servers for booking logic | Horizontal: 2→12 instances based on CPU/RPS |
| **Node.js Services** | RESTful API (Express.js) | Stateless, scale out during peaks |
| **Redis Cluster** | Distributed locks + cache layer | Vertical scaling (upgrade instance size) |
| **PostgreSQL Primary** | Authoritative seat inventory | Vertical scaling (32GB RAM) |
| **Read Replicas (2x)** | Analytics, reporting, user dashboards | Separate read traffic from writes |
| **SQS** | Payment message queue | AWS-managed, infinite throughput |
| **Lambda Workers** | Async payment processing | Auto-scale 0→50 concurrent executions |
| **S3** | Backup storage, logs | AWS-managed, infinite storage |

---

## Traffic Flow

### Read-Heavy Path (Event Browsing)
```
User → CloudFront (cache hit) → Response (50ms)
User → CloudFront (miss) → ALB → Node.js → Redis (cache hit) → Response (100ms)
User → All miss → ALB → Node.js → PostgreSQL Read Replica → Response (200ms)
```

### Write Path (Seat Booking)
```
User → ALB → Node.js → Redis SETNX lock → PostgreSQL Write → SQS publish → Response (300ms)
Background: SQS → Lambda → Payment Gateway → PostgreSQL Update
```

### Data Consistency Flow
```
Write to PostgreSQL → Invalidate Redis cache → Next read refreshes from DB
```

---

## High Availability

- **Multi-AZ Deployment:** RDS + Redis across 3 availability zones
- **Auto-failover:** RDS promotes read replica to primary (60s RTO)
- **Health Checks:** ALB removes unhealthy instances in 30s
- **Circuit Breaker:** Skip Redis on failure, direct DB queries

---

## Monitoring Stack

- **CloudWatch:** CPU, memory, disk I/O, query latency
- **X-Ray:** Distributed tracing for API requests
- **RDS Performance Insights:** Slow query detection
- **Custom Metrics:** Booking success rate, lock contention, queue depth
