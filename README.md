# BookMyShow - High-Scale Ticketing Platform

## Project Overview
Production-grade ticket booking system designed to handle massive concurrent traffic spikes during flash sales while guaranteeing zero double bookings and maintaining cost efficiency.

## System Constraints

### Constraint 1: 500,000 Concurrent Users

**Peak RPS Calculation:**
- Active users: 500,000
- Actions per user in first minute: ~3 (load event → select seats → attempt booking)
- Peak RPS = (500,000 × 3) / 60 = **25,000 RPS**

**Assumptions:**
- 80% users land at exactly 12:00:00
- Booking attempts spread over first 60 seconds
- 20% proceed to payment (5,000 payment transactions)

### Constraint 2: Zero Double Bookings

**What is Double Booking?**
Two or more users successfully booking the same seat for the same event.

**Why Catastrophic?**
- Legal liability and refunds
- Brand reputation damage
- Customer support overload
- Revenue loss from cancellations

**Required Guarantees:**
- ACID transactions with SERIALIZABLE isolation or row-level locking
- Atomic seat state transitions (available → held → booked)
- No race conditions in concurrent bookings
- Idempotent payment processing

### Constraint 3: $2,000/month AWS Budget

**Infrastructure Breakdown:**

| Component | Specification | Cost/month |
|-----------|--------------|------------|
| RDS PostgreSQL | db.r5.xlarge (4 vCPU, 32GB) | $580 |
| Read Replica (2x) | db.t4g.large | $280 |
| ElastiCache Redis | cache.r6g.large cluster | $260 |
| EC2 Auto Scaling | 6x t3.medium (burst) | $250 |
| ALB + CloudFront | Data transfer + requests | $180 |
| SQS + Lambda Workers | Payment processing | $150 |
| Data Transfer | Outbound traffic | $200 |
| S3 + Misc | Logs, backups | $100 |
| **TOTAL** | | **$2,000** |

**Design Constraints:**
- Use vertical scaling for databases (single powerful instance)
- Aggressive caching to reduce DB load
- Async processing for non-critical paths
- Auto-scaling only during peak hours
- Read replicas for reporting/analytics only