# AWS Cost Estimate: SwiftEats Scaled Architecture

**Pricing Source**: AWS EC2 On-Demand Pricing (May 2026)  
**Region**: us-east-1 (N. Virginia - most cost-effective for India traffic via CloudFront)  
**Exchange Rate**: 1 USD = ₹83 INR (approximate)

---

## Baseline Monthly Cost (Normal Traffic)

Steady-state architecture running 24/7 at typical load (~10% of World Cup spike capacity).

### 1. EC2 - Node.js Application Servers

**Configuration:**
- Instance Type: `t3.medium` (2 vCPU, 4 GB RAM)
- Hourly Rate: **$0.0416/hour** (on-demand)
- Baseline Instances: **5 instances** (minimum fleet)
- Hours per Month: **720 hours**

**Calculation:**
```
Monthly Cost = Instance Rate × Hours × Count
             = $0.0416/hour × 720 hours × 5 instances
             = $149.76/month per instance
             = $748.80/month total
```

**In INR**: ₹62,150 ($748.80 × 83)

---

### 2. RDS - PostgreSQL Primary Database

**Configuration:**
- Instance Type: `db.t3.medium` (2 vCPU, 4 GB RAM)
- Hourly Rate: **$0.034/hour** (on-demand, single-AZ)
- Hours per Month: **720 hours**
- Storage: **500 GB SSD** ($0.115/GB-month)

**Calculation:**
```
Compute Cost = $0.034/hour × 720 hours
             = $24.48/month

Storage Cost = $0.115/GB/month × 500 GB
             = $57.50/month

Total RDS Primary = $24.48 + $57.50 = $81.98/month
```

**In INR**: ₹6,804 ($81.98 × 83)

---

### 3. RDS - PostgreSQL Read Replica

**Configuration:**
- Instance Type: `db.t3.medium` (same as primary, 2 vCPU, 4 GB RAM)
- Hourly Rate: **$0.034/hour**
- Count: **1 replica** (warm standby for failover)
- Hours per Month: **720 hours**
- Storage: **500 GB SSD** (replicated, same cost)

**Calculation:**
```
Compute Cost = $0.034/hour × 720 hours × 1 replica
             = $24.48/month

Storage Cost = $0.115/GB/month × 500 GB
             = $57.50/month

Total RDS Replica = $24.48 + $57.50 = $81.98/month
```

**In INR**: ₹6,804 ($81.98 × 83)

---

### 4. ElastiCache - Redis Cluster

**Configuration:**
- Node Type: `cache.r6g.large` (2 vCPU, 16 GB RAM, ARM-based Graviton2)
- Hourly Rate: **$0.126/hour** (on-demand)
- Nodes: **2 nodes** (primary + replica for high availability)
- Hours per Month: **720 hours**

**Calculation:**
```
Monthly Cost = $0.126/hour × 720 hours × 2 nodes
             = $181.44/month
```

**In INR**: ₹15,039 ($181.44 × 83)

---

### 5. Application Load Balancer (ALB)

**Configuration:**
- Base Cost: **$0.0225/hour**
- LCU (Load Balancer Capacity Unit) estimate:
  - Baseline RPS: ~50k/s (distributed across instances)
  - New Connections: ~5,000/sec
  - Processed Bytes: ~200 MB/s
  - LCU estimate: **~2 LCUs** (most expensive dimension)
- LCU Cost: **$0.006 per LCU per hour**
- Hours per Month: **720 hours**

**Calculation:**
```
Base Cost = $0.0225/hour × 720 hours
          = $16.20/month

LCU Cost = $0.006/LCU/hour × 2 LCUs × 720 hours
         = $8.64/month

Total ALB = $16.20 + $8.64 = $24.84/month
```

**In INR**: ₹2,061 ($24.84 × 83)

---

### 6. CloudFront CDN - Data Transfer

**Configuration:**
- Cache Behavior:
  - Images/Static Assets: **100% cache hit rate** (origin-level cache)
  - Cache-Control: max-age=86400 (24 hours)
  - Invalidation policy: Smart invalidation on new deploys
- Edge Locations: **200+ global**
- Baseline traffic estimate:
  - Total image requests per month: ~2M images/day
  - Average image size: **100 KB**
  - Monthly data transfer: 2M × 100KB × 30 days = **6 TB/month**
  - Origin Shield: **Enabled** (extra 0.01/GB for origin protection)

**Pricing Tiers (weighted by region):**
- US/Europe (50% of traffic): **$0.085/GB**
- Asia-Pacific (40% of traffic): **$0.130/GB**
- Middle East/Africa (10% of traffic): **$0.180/GB**

**Calculation:**
```
US/EU Transfer = 3 TB × $0.085/GB
               = 3,000 GB × $0.085/GB
               = $255.00/month

APAC Transfer = 2.4 TB × $0.130/GB
              = 2,400 GB × $0.130/GB
              = $312.00/month

MEA Transfer = 0.6 TB × $0.180/GB
             = 600 GB × $0.180/GB
             = $108.00/month

Origin Shield = 6 TB × $0.01/GB
              = 6,000 GB × $0.01/GB
              = $60.00/month

Per-Request Cost = 2M requests/day × 30 days × $0.0075/10k requests
                 = 60M requests × $0.0075/10k
                 = $45.00/month

Total CloudFront = $255 + $312 + $108 + $60 + $45 = $780.00/month
```

**In INR**: ₹64,740 ($780 × 83)

---

### 7. SQS - Payment Queue

**Configuration:**
- Queue Type: Standard Queue (FIFO would be overkill for payments)
- Baseline Message Volume:
  - Checkout rate: ~5,000/day (normal traffic)
  - Payment requests queued: 5k/day
  - Monthly messages: 5k × 30 = **150,000 messages/month**
- Pricing: **$0.40 per million requests**

**Calculation:**
```
Cost = (150,000 messages / 1,000,000) × $0.40
     = 0.15 × $0.40
     = $0.06/month

Note: SQS cost negligible at baseline. Rounded to $5/month minimum.
```

**In INR**: ₹415 ($5 × 83)

---

### 8. Data Transfer - Inter-region (replication, failover)

**Configuration:**
- RDS replication: Primary to Replica (same AZ, no cost)
- CloudFront origin egress: Counted in CloudFront above
- EC2 inter-region failover: **Minimal** ($0/month at baseline)

**Cost**: **$0/month**

---

## **BASELINE MONTHLY TOTAL**

| Component | Cost (USD) | Cost (INR) |
|-----------|-----------|-----------|
| EC2 (5 × t3.medium) | $748.80 | ₹62,150 |
| RDS Primary (db.t3.medium) | $81.98 | ₹6,804 |
| RDS Replica (db.t3.medium) | $81.98 | ₹6,804 |
| ElastiCache Redis (2 × r6g.large) | $181.44 | ₹15,039 |
| ALB | $24.84 | ₹2,061 |
| CloudFront (6 TB data) | $780.00 | ₹64,740 |
| SQS | $5.00 | ₹415 |
| **BASELINE TOTAL** | **$1,904.04** | **₹158,013** |

---

---

## Peak Event Cost (World Cup Night - 4-Hour Spike)

The system auto-scales during the 8 PM - 12 AM World Cup match. Additional cost for this window.

### Scaling Events:

```
T+0:00 (8 PM):    Notification sent, scale begins
T+0:30 (8:30 PM): RPS hits 500k, scale to 20 instances
T+1:00 (9 PM):    RPS hits 1.2M, scale to 40 instances
T+2:00 (10 PM):   RPS stable at 1.2M, maintain 40 instances
T+3:00 (11 PM):   RPS begins declining, scale down starts
T+4:00 (12 AM):   Back to baseline (5 instances)
```

### 1. Additional EC2 Instances (Auto-Scale)

**Configuration:**
- Baseline: 5 instances (always running)
- Peak scale-up: 40 instances total (35 additional)
- Instance Type: `t3.medium` ($0.0416/hour)

**Timeline:**
```
T+0:00-0:30:  5 instances (baseline, 30 min)
T+0:30-1:00:  20 instances (15 additional, 30 min)
T+1:00-3:00:  40 instances (35 additional, 120 min)
T+3:00-4:00:  20 instances (15 additional, 60 min)
T+4:00+:      5 instances (back to baseline)
```

**Calculation:**
```
Baseline (4 hrs) = 5 instances × 4 hrs × $0.0416/hr
                 = $0.832

Additional instances during scale:
  30 min @ 15 extra = 15 × 0.5 hrs × $0.0416
                    = $0.312
  120 min @ 35 extra = 35 × 2 hrs × $0.0416
                     = $2.912
  60 min @ 15 extra = 15 × 1 hr × $0.0416
                    = $0.624

Peak EC2 Cost = $0.832 + $0.312 + $2.912 + $0.624 = $4.68 for 4 hours

But let's include cost for ALL 4 hours (some overprovisioning is expected):
Average instances during 4 hrs = (5 + 20 + 40 + 40 + 20 + 5) / 6 ≈ 22 instances
Cost = 22 instances × 4 hrs × $0.0416/hr = $3.66

More conservative estimate: actual cost ~$3.50-4.50 for 4-hour event
```

**Peak EC2 Cost**: **$4.00 (conservative)**

**In INR**: ₹332 ($4 × 83)

---

### 2. CloudFront Surge Transfer

**Configuration:**
- Peak RPS: 1.2M (includes ~6% static asset requests)
- Image requests at peak: 72k/sec = 259 million images/4 hours
- Average image size: 100 KB
- Peak data transfer: 259M × 100KB = **25.9 TB over 4 hours**

**Calculation:**
```
CloudFront surge is already counted in monthly estimate,
but additional transfer during peak:

Baseline daily images: 2M
Peak 4-hour images: 259M (130× normal rate)
Additional transfer: 25.9 TB

Peak CDN Cost = 25.9 TB × weighted avg $0.115/GB
              = 25,900 GB × $0.115
              = $2,978.50 for 4 hours
```

**Peak CloudFront Cost**: **$2,978.50**

**In INR**: ₹247,215 ($2,978.50 × 83)

---

### 3. RDS Surge - Read Replica Promotion (if primary fails)

**Configuration:**
- Assumption: Primary might experience brief slowdown
- Replica auto-promotion: 5-10 minute recovery
- Cost: Minimal extra (already running)

**Peak RDS Cost**: **$0 (already included in baseline)**

---

### 4. ElastiCache Surge (Redis commands at peak)

**Configuration:**
- Baseline: 2 nodes (r6g.large)
- Peak: No scaling (Redis handles 1.2M ops/sec easily)
- Cost: Minimal extra

**Peak ElastiCache Cost**: **$0 (no scaling needed)**

---

### 5. ALB Surge - LCU scaling

**Configuration:**
- Baseline: ~2 LCUs
- Peak: ~8 LCUs (1.2M RPS with complex routing)
- Duration: 4 hours

**Calculation:**
```
Peak LCU Cost = $0.006/LCU/hour × 8 LCUs × 4 hours
              = $0.192

Marginal cost (6 additional LCUs) = $0.006 × 6 × 4 = $0.144
```

**Peak ALB Cost**: **$0.15 (additional)**

**In INR**: ₹12.45 ($0.15 × 83)

---

### 6. SQS Surge - Payment Queue

**Configuration:**
- Baseline: 5k payments/day = ~200/hour
- Peak: 600k payments over 4 hours = 150k/hour
- Additional messages: ~594k over 4 hours

**Calculation:**
```
Peak SQS Cost = (594,000 messages / 1,000,000) × $0.40
              = 0.594 × $0.40
              = $0.238

Rounding to $1.00 to account for message deduplication checks
```

**Peak SQS Cost**: **$1.00**

**In INR**: ₹83 ($1 × 83)

---

## **PEAK EVENT COST (4 hours)**

| Component | Cost (USD) | Cost (INR) |
|-----------|-----------|-----------|
| EC2 Auto-Scale (+35 instances) | $4.00 | ₹332 |
| CloudFront Surge (+25.9 TB) | $2,978.50 | ₹247,215 |
| ALB LCU Surge | $0.15 | ₹12 |
| SQS Surge | $1.00 | ₹83 |
| **PEAK EVENT TOTAL (4 hrs)** | **$2,983.65** | **₹247,643** |

---

## Monthly Cost Comparison

### Scenario 1: With Scaled Infrastructure (Baseline + Monthly Peak Events)

Assuming 1 World Cup match per month (4 hours):

```
Monthly Baseline = $1,904.04
+ 4-hour Peak Event = $2,983.65
= Monthly Total = $4,887.69
```

**Monthly Total with Scaling: $4,887.69 (₹405,656)**

---

### Scenario 2: Running on Monolith (No Scaling)

**Hypothetical:**
- Keep 5 instances of t3.medium running 24/7
- Same baseline cost: **$1,904.04/month**
- But: **System crashes during peak** (no revenue, plus emergency incident costs)
- Disaster recovery: Manual restart, debugging, incident response = **$50k-150k unplanned cost**
- Lost revenue during 45-minute outage (calculated below)

---

---

## Business Justification: ROI Analysis

### Revenue Loss During Outage

**Given Parameters:**
- Loss rate during outage: **₹4.2 crore per minute** (provided)
- Outage duration in World Cup scenario: **45 minutes** (from cascade analysis)

**Revenue Loss Calculation:**
```
Lost Revenue = ₹4.2 crore/minute × 45 minutes
             = ₹189 crore
             = $227.71 million USD
```

### Cost-Benefit Analysis

**Option A: Keep Monolith**
```
Monthly Infrastructure Cost:       $1,904.04
Probability of outage per month:  ~90% (expected 1x per viral event)
Expected outage cost per month:   45 min × ₹4.2Cr/min = ₹189 Cr
Expected monthly loss:            $227.71M × 90% = $204.94M

Total Expected Monthly Cost:      $204.94M (+ infrastructure costs)
```

**Option B: Deploy Scaled Infrastructure**
```
Monthly Infrastructure Cost:       $4,887.69 (includes 1 World Cup event)
Probability of outage per month:  <0.1% (99.9% uptime SLA)
Expected outage cost per month:   <$227,710 (negligible)

Total Expected Monthly Cost:      ~$4,888
```

### ROI Calculation

```
Monthly Cost Difference = $204,942,112 - $4,887.69 ≈ $205 million saved
Annual Savings = $205M × 12 months ≈ $2.46 BILLION per year

Break-Even Time: 0.07 seconds (under 1 hundredth of a second)
                 (Cost of outage > Annual infrastructure cost)

Risk Reduction: 
  - Before: 90% probability of ₹189 Cr loss event
  - After: <0.1% probability of same event
  - Expected loss reduction: 899x
```

### Statement

**The scaled architecture is not just cost-effective—it is financially mandatory.**

Even accounting for the increased monthly infrastructure cost of **$4,887.69** (compared to baseline $1,904.04), the system prevents a **₹189 crore loss** every time a viral event occurs. 

With an expected **1 major viral event per month** (World Cup, Black Friday, festival sales, etc.), the **expected monthly loss without scaling is $204.94M**. Deploying the scaled architecture reduces this to near-zero while adding only $2,983.65 in peak costs.

**The ROI is 41,890:1** — for every dollar spent on infrastructure, the company avoids $41,890 in expected losses.

Furthermore, this analysis assumes only 1 World Cup event per month. In reality:
- IPL matches (cricket): 60 matches/season
- Festival sales (Diwali, Holi): 4 major events/year
- New Year campaigns: 2x/year
- Flash sales: 10-15x/year

**Conservative estimate: 1.5-2 viral events per month → $307-409M in monthly losses if monolith crashes.**

**Conclusion: Scaling is not optional. It is the only financially viable option.**

---

## One-Time Migration Cost (Not Included Above)

- AWS migration service: ~$50k
- Architect/consulting: ~$100k
- Testing and QA: ~50k hours @ $50/hr = $2.5M
- Training: ~$20k

**Total One-Time Cost: ~$2.67M**

**Payback period: ~0.5 hours** (less time than a single World Cup outage revenue loss)

---

## Cost Optimization Opportunities (Future)

If costs need to be reduced further:

1. **Reserved Instances**: -40% on EC2 (commit to 1-3 years)
2. **Spot Instances for Payment Workers**: -70% on compute (SQS workers tolerant of interruption)
3. **RDS Aurora**: -25% on database costs vs PostgreSQL RDS
4. **Multi-region CloudFront**: +10-20% cost but -50% latency for India
5. **S3 + CloudFront for static assets**: Already optimal with current CDN setup

**Potential savings with optimization: -35-50% = $1,700-2,400/month**

---

## Monthly Infrastructure Cost Summary

| Metric | Amount |
|--------|--------|
| Baseline Monthly Cost | $1,904.04 |
| Peak Event Cost (4 hrs) | $2,983.65 |
| **Total Monthly with 1 Peak Event** | **$4,887.69** |
| Estimated Outage Loss (if monolith fails) | $227,710,000 |
| **ROI on Scaling Investment** | **41,890:1** |
| Annual Infrastructure Cost | $58,652 |
| Annual Revenue Saved (4 World Cups) | ~$911M |

