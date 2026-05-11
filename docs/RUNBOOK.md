# SwiftEats Incident Runbook

**Version**: 1.0  
**Last Updated**: May 2026  
**Owner**: Platform Team  
**Severity**: Critical

---

## CRITICAL: READ THIS FIRST

You are reading this because the system is down or degraded. **Every second counts.**

- **Do NOT overthink diagnosis** - use the decision tree in STEP 2
- **Do NOT make multiple changes at once** - change one thing, measure result, then decide next action
- **Do NOT touch the database schema** - only roll back application code
- **Do call for help** - Page: `@platform-oncall` in #swiggy-incidents
- **Timeout for diagnosis**: 5 minutes. If root cause not found by then, escalate to engineering manager

---

## STEP 1 - DETECT: CloudWatch Alarms

These alarms must be configured and active. If you don't see them, **something is very wrong**.

### CRITICAL ALARMS

#### Alarm 1: RDS Connection Pool Exhaustion

| Property        | Value                                               |
| --------------- | --------------------------------------------------- |
| **Alarm Name**  | `swiggy-db-connections-critical`                    |
| **Metric**      | `AWS/RDS → DatabaseConnections`                     |
| **Threshold**   | **≥90 connections** (out of 100 max)                |
| **Duration**    | 1 minute (if sustained for 60 seconds)              |
| **Alert Level** | 🔴 **CRITICAL** - immediate escalation              |
| **SNS Topic**   | `arn:aws:sns:us-east-1:ACCT:platform-critical`      |
| **Action**      | PagerDuty trigger + Slack #swiggy-incidents + email |

**What this means**: Database cannot accept new connections. System is failing fast (connection timeouts).

**Check it**:

```bash
# In AWS Console → RDS → Databases → swiggy-prod-primary
# Scroll to "Connections" graph
# If this alarm fires, connections are at 90+ of 100
```

---

#### Alarm 2: Node.js Application CPU Saturation

| Property        | Value                                                |
| --------------- | ---------------------------------------------------- |
| **Alarm Name**  | `swiggy-app-cpu-critical`                            |
| **Metric**      | `AWS/EC2 → CPUUtilization` (across ALB target group) |
| **Threshold**   | **≥90% for ≥2 minutes**                              |
| **Alert Level** | 🔴 **CRITICAL**                                      |
| **SNS Topic**   | `arn:aws:sns:us-east-1:ACCT:platform-critical`       |
| **Action**      | Auto-scale up to max 50 instances + alert oncall     |

**What this means**: Event loop is maxed out. All requests are slow or timing out.

**Check it**:

```bash
# In AWS Console → CloudWatch → Metrics → EC2 → CPUUtilization
# Filter by: "swiggy-app-asg" (auto-scaling group)
# Should trigger auto-scale action automatically
```

---

#### Alarm 3: Node.js Application Memory Usage

| Property        | Value                                                          |
| --------------- | -------------------------------------------------------------- |
| **Alarm Name**  | `swiggy-app-memory-critical`                                   |
| **Metric**      | `AWS/EC2 → MemoryUtilization` (custom CloudWatch agent metric) |
| **Threshold**   | **≥85% for ≥2 minutes**                                        |
| **Alert Level** | 🟠 **WARNING** → 🔴 **CRITICAL** if hits 95%                   |
| **SNS Topic**   | `arn:aws:sns:us-east-1:ACCT:platform-warning`                  |
| **Action**      | Alert + check for memory leak                                  |

**What this means**: Application heap is growing. If this hits 95%, process will crash with OOM.

**Check it**:

```bash
# In AWS Console → CloudWatch → Metrics
# Search: "MemoryUtilization"
# Filter by "swiggy-app-asg"
```

---

#### Alarm 4: ALB Target Health

| Property        | Value                                                           |
| --------------- | --------------------------------------------------------------- |
| **Alarm Name**  | `swiggy-alb-unhealthy-targets`                                  |
| **Metric**      | `AWS/ApplicationELB → UnHealthyHostCount`                       |
| **Threshold**   | **≥1 unhealthy target**                                         |
| **Duration**    | 1 minute                                                        |
| **Alert Level** | 🟠 **WARNING** if 1-2 targets, 🔴 **CRITICAL** if >20% of fleet |
| **SNS Topic**   | `arn:aws:sns:us-east-1:ACCT:platform-warning`                   |
| **Action**      | Check instance logs, restart if needed                          |

**What this means**: One or more app servers have failed health checks. They're being removed from load balancer.

**Check it**:

```bash
# In AWS Console → EC2 → Load Balancers → swiggy-alb
# Target Groups → swiggy-app-tg
# See "Registered Targets" section - look for any in "Unhealthy" state
```

---

#### Alarm 5: RDS Replication Lag

| Property        | Value                                                    |
| --------------- | -------------------------------------------------------- |
| **Alarm Name**  | `swiggy-rds-replica-lag`                                 |
| **Metric**      | `AWS/RDS → AuroraBinlogReplicaLag` (or custom SQL query) |
| **Threshold**   | **≥5 seconds**                                           |
| **Duration**    | 2 minutes                                                |
| **Alert Level** | 🟠 **WARNING** - replica might not be able to failover   |
| **SNS Topic**   | `arn:aws:sns:us-east-1:ACCT:platform-warning`            |
| **Action**      | Investigate primary DB load, check network               |

**What this means**: Read replica is falling behind primary. If primary crashes, failover will lose data.

**Check it**:

```bash
# In AWS Console → RDS → Databases → swiggy-prod-replica
# Scroll to "Replication" section
# See "Replica lag" metric
```

---

#### Alarm 6: SQS Payment Queue Depth

| Property        | Value                                              |
| --------------- | -------------------------------------------------- |
| **Alarm Name**  | `swiggy-sqs-payment-queue-depth`                   |
| **Metric**      | `AWS/SQS → ApproximateNumberOfMessagesVisible`     |
| **Threshold**   | **≥10,000 messages**                               |
| **Duration**    | 2 minutes                                          |
| **Alert Level** | 🟠 **WARNING** - payment processing falling behind |
| **SNS Topic**   | `arn:aws:sns:us-east-1:ACCT:platform-warning`      |
| **Action**      | Scale payment workers, check for processing errors |

**What this means**: Payment requests are being queued faster than they're being processed. Users see "Processing..." spinner.

**Check it**:

```bash
# In AWS Console → SQS → swiggy-payment-queue
# See "Messages Available" metric
# Compare to "Messages in Flight" (being processed now)
```

---

#### Alarm 7: Redis Cache Evictions

| Property        | Value                                            |
| --------------- | ------------------------------------------------ |
| **Alarm Name**  | `swiggy-redis-evictions-high`                    |
| **Metric**      | `AWS/ElastiCache → Evictions`                    |
| **Threshold**   | **>100 evictions per minute**                    |
| **Duration**    | 2 minutes                                        |
| **Alert Level** | 🟠 **WARNING** - cache running out of memory     |
| **SNS Topic**   | `arn:aws:sns:us-east-1:ACCT:platform-warning`    |
| **Action**      | Check cache hit ratio, consider scaling Redis up |

**What this means**: Redis is full and evicting old data. Cache is less effective.

**Check it**:

```bash
# In AWS Console → ElastiCache → Clusters → swiggy-cache
# See "Evictions" metric in monitoring
```

---

#### Alarm 8: Application Error Rate

| Property        | Value                                            |
| --------------- | ------------------------------------------------ |
| **Alarm Name**  | `swiggy-app-error-rate-critical`                 |
| **Metric**      | `AWS/ApplicationELB → HTTPCode_Target_5XX_Count` |
| **Threshold**   | **>5% of requests returning 5XX**                |
| **Duration**    | 1 minute                                         |
| **Alert Level** | 🔴 **CRITICAL**                                  |
| **SNS Topic**   | `arn:aws:sns:us-east-1:ACCT:platform-critical`   |
| **Action**      | Check app logs immediately                       |

**What this means**: Application is throwing errors. Could be database, external API, or application bug.

**Check it**:

```bash
# In AWS Console → CloudWatch → Alarms
# Search: "swiggy-app-error-rate-critical"
# See current error rate percentage
```

---

## STEP 2 - TRIAGE: Decision Tree (30 Seconds)

Follow this checklist **in order**. Stop at the first RED condition you find.

```
START HERE
    │
    ├─ [ALARM 1] DB Connections ≥90/100?
    │  YES (🔴) → GO TO: STEP 3A - DB CONNECTION EXHAUSTION
    │  NO (✅) → Continue
    │
    ├─ [ALARM 2] App CPU ≥90% for 2+ min?
    │  YES (🔴) → GO TO: STEP 3B - CPU SATURATION (Check auto-scale)
    │  NO (✅) → Continue
    │
    ├─ [ALARM 3] App Memory ≥85% for 2+ min?
    │  YES (🟠) → GO TO: STEP 3C - MEMORY LEAK
    │  NO (✅) → Continue
    │
    ├─ [ALARM 5] Unhealthy Targets ≥1?
    │  YES (🟠) → GO TO: STEP 3D - UNHEALTHY INSTANCES
    │  NO (✅) → Continue
    │
    ├─ [ALARM 6] Payment Queue Depth ≥10k messages?
    │  YES (🟠) → GO TO: STEP 3E - PAYMENT QUEUE BACKUP
    │  NO (✅) → Continue
    │
    ├─ [ALARM 8] Error Rate >5% (5XX responses)?
    │  YES (🔴) → GO TO: STEP 3F - APPLICATION ERRORS
    │  NO (✅) → Continue
    │
    └─ If NO ALARMS are red, but users are reporting issues:
       → GO TO: STEP 3G - CHECK APPLICATION LOGS (Memory leak, GC pause)

```

**Reference Card** (print and tape to your monitor):

| Symptom                        | Check First                 | Second Check          | Root Cause                         |
| ------------------------------ | --------------------------- | --------------------- | ---------------------------------- |
| "Cannot connect" errors        | DB Connections (Alarm 1)    | App CPU (Alarm 2)     | Pool exhausted OR Event loop stuck |
| Everything slow                | App CPU (Alarm 2)           | Query performance     | CPU maxed out                      |
| Intermittent crashes           | App Memory (Alarm 3)        | App logs (OOM errors) | Memory leak                        |
| 50% of users seeing errors     | Unhealthy targets (Alarm 5) | Auto-scale status     | Instance crashed                   |
| Payments stuck on "Processing" | SQS depth (Alarm 6)         | Payment worker logs   | Payment workers down               |
| Random 500 errors              | Error rate (Alarm 8)        | App logs              | Application exception              |

---

## STEP 3 - RESPOND: Component-Specific Actions

### STEP 3A: DB Connection Pool Exhaustion

**Symptom**: Alarm 1 fires. Users see "Connection refused" or requests timeout after 30s.

**This is CRITICAL. You have ~2 minutes before cascading failures.**

#### Check 1: Verify the problem

```bash
# SSH to bastion host
ssh -i ~/.ssh/prod-key ec2-user@bastion.swiggy.internal

# Connect to PostgreSQL and check connections
psql -h swiggy-prod-primary.XXXXX.us-east-1.rds.amazonaws.com \
     -U swiggy_admin \
     -d swiggy_prod \
     -c "SELECT count(*) as connection_count FROM pg_stat_activity WHERE state = 'active';"

# Expected: Should be <50 normally. If >90, problem confirmed.

# See which queries are holding connections
psql -h swiggy-prod-primary.XXXXX.us-east-1.rds.amazonaws.com \
     -U swiggy_admin \
     -d swiggy_prod \
     -c "SELECT pid, usename, query, state, wait_event FROM pg_stat_activity WHERE state != 'idle' ORDER BY query_start DESC LIMIT 10;"

# Look for:
# 1. Long-running queries (query_start is old)
# 2. Razorpay payment queries (query contains 'razorpay' or takes >500ms)
# 3. Unresponsive connections (wait_event = 'something external')
```

#### Check 2: Is this a Razorpay payment call holding the connection?

```bash
# Check SQS queue depth - if payments are being queued properly,
# payment queries should NOT be holding DB connections
aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/ACCT/swiggy-payment-queue \
  --attribute-names ApproximateNumberOfMessages

# Expected: Should be empty or <1000 at baseline
# If >10000, payment workers are down (go to STEP 3E)
```

#### Action 1: Kill long-running queries (⚠️ Use with caution)

```bash
# Identify blocking query PID from above. Let's say PID = 12345
# ONLY kill if query is >10 minutes old AND not a production migration

pg_terminate_backend(12345);  -- Run this in psql
```

**What success looks like:**

- Connection count drops from 90+ to <50 within 10 seconds
- Users stop seeing "Connection refused" errors
- Alarm 1 goes green within 1 minute

**If this doesn't work:** Go to STEP 3B (CPU saturation) and then scale.

**Responsible Team**: @database-oncall (Slack channel: #swiggy-incidents)

---

### STEP 3B: CPU Saturation

**Symptom**: Alarm 2 fires. CPU > 90% sustained. Alarm should trigger auto-scale, but verify it's working.

#### Check 1: Verify auto-scaling is enabled and working

```bash
# Check auto-scaling group status
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names swiggy-app-asg \
  --query 'AutoScalingGroups[0].[MinSize,DesiredCapacity,MaxSize,Instances[*].InstanceId]'

# Expected output:
# MinSize: 5
# DesiredCapacity: should be increasing (5 → 10 → 20 → 50)
# MaxSize: 50
# Instances: count should be growing

# If DesiredCapacity is NOT increasing, auto-scale is broken
```

#### Check 2: Manual scale-up (if auto-scale is stuck)

```bash
# Override auto-scaling and manually scale to 20 instances
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name swiggy-app-asg \
  --desired-capacity 20 \
  --honor-cooldown

# Monitor the scaling:
watch -n 5 'aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names swiggy-app-asg \
  --query "AutoScalingGroups[0].Instances | length"'

# Press Ctrl+C when DesiredCapacity = 20 and InService count = 20
```

#### Check 3: Is there a deployment in progress?

```bash
# If scaling is happening, but new instances aren't ready,
# there might be a deployment issue

# Check CodeDeploy status
aws deploy list-deployments \
  --application-name swiggy-app \
  --deployment-status-filter InProgress

# If there's a deployment stuck, you may need to abort it:
# aws deploy stop-deployment --deployment-id d-XXXXXX

# Then retry scaling
```

**What success looks like:**

- CPU drops from 90%+ to 60-70% within 2 minutes
- Alarm 2 goes green
- Latency (p95) drops from 5000ms+ to 500-1000ms

**If this doesn't work:** Likely DB connection pool is also exhausted (Alarm 1). Do STEP 3A first.

**Responsible Team**: @platform-oncall (Slack: #swiggy-incidents)

---

### STEP 3C: Memory Leak

**Symptom**: Alarm 3 fires. Memory > 85%. This usually means a memory leak in application code.

#### Check 1: Identify the leak

```bash
# SSH to one of the app instances
aws ec2 describe-instances \
  --filters "Name=tag:aws:autoscaling:groupName,Values=swiggy-app-asg" \
             "Name=instance-state-name,Values=running" \
  --query 'Reservations[0].Instances[0].PrivateIpAddress' \
  --output text

# SSH to that instance
ssh -i ~/.ssh/prod-key ec2-user@<PRIVATE_IP>

# Check Node.js process memory
ps aux | grep node

# Look for memory growth over time. If memory is consistently >80% of 4GB,
# there's a leak or code path that allocates too much

# Check heap snapshot
curl -s http://localhost:9229/json/list | jq .  # V8 debugger protocol

# Or check application metrics directly
curl -s http://localhost:3000/metrics | grep nodejs_heap_size_bytes
```

#### Check 2: Is this a known leak?

```bash
# Check recent deployments
aws deploy list-deployments \
  --application-name swiggy-app \
  --query 'deployments[0:3]'

# Compare to last stable version
# Check release notes: docs/RELEASES.md

# If deployed <30 min ago, this might be a regression
```

#### Action 1: Restart affected instances (graceful termination)

```bash
# Don't kill instances abruptly. Use connection draining.
# ALB will stop sending NEW requests, but existing requests complete.

aws elb deregister-instances-from-load-balancer \
  --load-balancer-name swiggy-alb \
  --instances i-0123456789abcdef0

# Wait 30 seconds for connections to drain
sleep 30

# Now terminate the instance
aws ec2 terminate-instances \
  --instance-ids i-0123456789abcdef0

# Auto-scaling will launch a replacement instance automatically
```

#### Action 2: If memory issue is widespread (>50% of fleet affected), consider rollback

```bash
# See STEP 4 - ROLLBACK for detailed rollback procedure
# But first, confirm it's a code leak, not just load
```

**What success looks like:**

- Memory on restarted instances drops from 85% to 30-40%
- Remaining instances gradually drain traffic
- Alarm 3 goes green

**Responsible Team**: @app-dev-oncall (Slack: #swiggy-incidents)

---

### STEP 3D: Unhealthy Targets (Crashed Instances)

**Symptom**: Alarm 5 fires. One or more instances showing "Unhealthy" in ALB target group.

#### Check 1: Identify unhealthy instances

```bash
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:ACCT:targetgroup/swiggy-app-tg/50dc6d86c83f7207 \
  --query 'TargetHealthDescriptions[?TargetHealth.State==`unhealthy`] | [*].[Target.Id,TargetHealth.Reason]'

# Output example:
# i-0123456789abcdef0  "Target response code was 500"
# i-1111111111111111   "Health checks failed with these codes: [500]"
```

#### Check 2: Get logs from unhealthy instance

```bash
# Get one unhealthy instance ID
UNHEALTHY_ID="i-0123456789abcdef0"

# SSH to it
aws ec2 describe-instances \
  --instance-ids $UNHEALTHY_ID \
  --query 'Reservations[0].Instances[0].PrivateIpAddress'

ssh -i ~/.ssh/prod-key ec2-user@<PRIVATE_IP>

# Check Node.js process
ps aux | grep node

# If process is NOT running:
systemctl status swiggy-app
sudo journalctl -u swiggy-app -n 50

# Look for:
# - Out of Memory error (OOM)
# - Application exception
# - Connection pool exhaustion

# Check if process crashed
ls -ltr /var/log/swiggy-app/error.log | tail -20
```

#### Action 1: Restart the application

```bash
# On the unhealthy instance:
sudo systemctl restart swiggy-app

# Check if it's healthy now
sudo systemctl status swiggy-app

# It should show "active (running)"
```

#### Action 2: If restart doesn't fix it, terminate the instance

```bash
# The instance is likely corrupted or has unrecoverable issue
# Terminate it and auto-scaling will replace it

aws ec2 terminate-instances --instance-ids i-0123456789abcdef0

# Wait 1-2 minutes for replacement to launch and become healthy
```

**What success looks like:**

- Unhealthy instance either recovers (status = "healthy") or is replaced
- ALB removes it from rotation and routes traffic to healthy instances
- Alarm 5 goes green
- Error rate drops

**Responsible Team**: @platform-oncall (Slack: #swiggy-incidents)

---

### STEP 3E: Payment Queue Backup

**Symptom**: Alarm 6 fires. SQS queue depth > 10,000 messages. Users see "Processing..." stuck for >5 seconds.

#### Check 1: Verify payment workers are running

```bash
# List payment worker instances
aws ec2 describe-instances \
  --filters "Name=tag:service,Values=payment-worker" \
           "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[*].[InstanceId,PrivateIpAddress,State.Name]'

# If no instances running, workers are down (critical!)
```

#### Check 2: Check worker health

```bash
# SSH to one payment worker
ssh -i ~/.ssh/prod-key ec2-user@<PAYMENT_WORKER_IP>

# Check process status
ps aux | grep payment-worker

# If process is down:
sudo systemctl restart payment-worker
sudo systemctl status payment-worker

# Check logs
sudo journalctl -u payment-worker -n 50

# Look for:
# - API failures to Razorpay
# - Database connection errors
# - Out of memory
```

#### Check 3: Scale up payment workers if overloaded

```bash
# Current: how many payment workers are running?
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names payment-worker-asg \
  --query 'AutoScalingGroups[0].[DesiredCapacity,Instances | length]'

# Scale to 50 workers if queue is very deep
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name payment-worker-asg \
  --desired-capacity 50 \
  --honor-cooldown

# Monitor queue depth as workers process messages:
watch -n 10 'aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/ACCT/swiggy-payment-queue \
  --attribute-names ApproximateNumberOfMessages \
  --query "Attributes.ApproximateNumberOfMessages"'
```

#### Action: Send DLQ messages back to main queue (if they keep failing)

```bash
# Payments that fail 3 times go to Dead Letter Queue (DLQ)
# Don't retry endlessly. After 3 retries, manual review is needed.

# For now, just scale and wait for queue to drain
# Don't manually redrive DLQ messages without analyzing them first
```

**What success looks like:**

- Queue depth drops from 10k+ to <1000 within 5 minutes
- Latency for payment requests drops
- Users stop seeing stuck "Processing..." state
- Alarm 6 goes green

**Responsible Team**: @payment-oncall (Slack: #swiggy-incidents)

---

### STEP 3F: Application Errors (5XX responses)

**Symptom**: Alarm 8 fires. Error rate > 5%. Users seeing 500 errors.

#### Check 1: Identify the error

```bash
# Get error logs from CloudWatch Logs
aws logs tail /aws/ecs/swiggy-app --follow

# Or query last 1000 errors:
aws logs filter-log-events \
  --log-group-name /aws/ecs/swiggy-app \
  --filter-pattern "ERROR" \
  --query 'events[*].[timestamp,message]' \
  --limit 10

# Look for patterns:
# - "Connection refused" → DB connection issue (STEP 3A)
# - "ENOENT: file not found" → Asset serving issue
# - "ReferenceError" → Code bug
# - "Razorpay API error" → External service issue
```

#### Check 2: Identify affected API endpoint

```bash
# Which endpoint is throwing errors?
aws logs filter-log-events \
  --log-group-name /aws/ecs/swiggy-app \
  --filter-pattern "500" \
  --query 'events[*].message' \
  --limit 20

# Parse logs to find path:
# Example: "POST /api/checkout returned 500"
# Example: "GET /api/restaurants returned 500"
```

#### Action 1: Correlate with deployment

```bash
# Did we deploy recently?
aws deploy list-deployments \
  --application-name swiggy-app \
  --query 'deployments[0:1]' \
  --deployment-status-filter Succeeded

# Check timestamp of last deployment
# If <30 minutes ago, this is likely a regression

# See STEP 4 - ROLLBACK for rollback procedure
```

#### Action 2: Restart the application if not a code issue

```bash
# If errors are intermittent (not sustained), try restart
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name swiggy-app-asg \
  --desired-capacity 5 \
  --honor-cooldown

# Then scale back up:
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name swiggy-app-asg \
  --desired-capacity 20 \
  --honor-cooldown
```

**What success looks like:**

- Error rate drops below 1%
- Alarm 8 goes green
- Users stop reporting 500 errors

**Responsible Team**: @app-dev-oncall (Slack: #swiggy-incidents)

---

### STEP 3G: Check Application Logs (Suspected GC/Memory Issue)

**Symptom**: No specific alarm is firing, but users report slowness or occasional crashes.

#### Check 1: Look for GC pauses

```bash
# Check Node.js GC logs (if enabled)
aws logs tail /aws/ecs/swiggy-app --filter-pattern "GC" --follow

# Expected: "GC completed in 500ms" (not abnormal)
# Problem: "GC completed in 5000ms" (too long) or frequent GC

# Alternative: Check V8 heap metrics
curl -s http://localhost:3000/metrics | grep -i gc
```

#### Check 2: Look for memory growth

```bash
# If memory keeps growing but never drops, there's a leak
# Check recent code changes in memory-intensive paths:

# Services that are typically culprits:
# 1. Image processing (if still serving images directly)
# 2. Bulk data operations (user exports, analytics)
# 3. Cache warming (loading all restaurants into memory)
# 4. Event listeners not being cleaned up

# Search logs for warnings:
aws logs filter-log-events \
  --log-group-name /aws/ecs/swiggy-app \
  --filter-pattern "memory" \
  --query 'events[*].message'
```

#### Check 3: Look for connection leaks

```bash
# Are database connections being properly released?
# Check for "connection never released" or similar warnings

aws logs filter-log-events \
  --log-group-name /aws/ecs/swiggy-app \
  --filter-pattern "connection" \
  --query 'events[*].message'

# If you see warnings like "connection acquired but never released",
# there's likely a code bug where .close() is not being called
```

**What success looks like:**

- No more crashes
- Memory stabilizes
- If a leak was found, it's tracked in an issue for dev team to fix

**Responsible Team**: @app-dev-oncall (Slack: #swiggy-incidents)

---

## STEP 4 - ROLLBACK: When and How

### Rollback Decision Tree

```
Is the incident caused by a code deployment?
    YES → Go to SECTION A: Code Rollback
    NO  → Go to SECTION B: No Code Rollback Needed

SECTION A: Code Rollback
    │
    ├─ Was the deployment <30 min ago?
    │  YES → Proceed with rollback (high confidence this is the cause)
    │  NO  → Don't rollback (too much time has passed, likely unrelated)
    │
    ├─ Is error rate still rising despite other mitigations?
    │  YES → Rollback NOW (don't wait for perfect diagnosis)
    │  NO  → Keep the deployment (it's helping)
    │
    └─ Did your mitigations (scaling, restart) fix the issue?
       YES → Keep current deployment + add hotfix (don't rollback)
       NO  → Rollback immediately

SECTION B: No Code Rollback Needed
    └─ Continue troubleshooting with STEP 3 procedures
       (Database, Memory, Scaling issues don't need rollback)
```

### NEVER Rollback: Database Schema

```
⚠️ WARNING: DO NOT ROLLBACK DATABASE MIGRATIONS

If you rolled back application code but the database schema changed:
- Old code will NOT understand new schema
- Result: Application errors, data corruption, data loss

INSTEAD:
1. If schema migration is incomplete/wrong:
   - Stop the rollback
   - Contact @database-oncall immediately
   - Create a forward migration to fix the issue
   - Deploy the fix as a NEW version

2. If schema migration broke something:
   - Keep the schema (you can't unbreak it easily)
   - Deploy application code that handles both old and new schema
   - This is called a "backward compatible migration"

3. Example - if you added a new required column:
   - Don't delete the column (that's a data loss risk)
   - Instead: deploy code that provides a default value
   - Then: migrate data in background
   - Then: make column required once data is migrated
```

### Code Rollback: Step-by-Step

#### Step 1: Confirm which version to rollback to

```bash
# Get deployment history
aws deploy list-deployments \
  --application-name swiggy-app \
  --query 'deployments[0:5]'

# Output example:
# d-XXXXXX  2024-05-11 10:00:00 UTC  Succeeded  (CURRENT - problematic)
# d-YYYYY   2024-05-11 09:30:00 UTC  Succeeded  (PREVIOUS - stable)
# d-ZZZZZ   2024-05-10 22:00:00 UTC  Succeeded

# Last successful deployment was PREVIOUS (d-YYYYY)
# Rollback to that version
```

#### Step 2: Get the previous version details

```bash
# Get the revision (git commit hash) of the previous version
aws deploy get-deployment \
  --deployment-id d-YYYYY \
  --query 'deploymentInfo.[Revision.S3Bucket,Revision.S3Key,Revision.RevisionType]'

# Or simpler: check git for the previous stable version
git log --oneline | head -5
# Find the commit before the problematic one
```

#### Step 3: Create a new rollback deployment

```bash
# Option 1: Using AWS CodeDeploy console
# Go to Deployments → Create deployment
# Select: Application = swiggy-app
# Revision: manually paste the S3 location of previous version
# Or select git revision from history

# Option 2: Using CLI
aws deploy create-deployment \
  --application-name swiggy-app \
  --deployment-group-name swiggy-app-prod \
  --revision revisionType=S3,s3Location=s3://swiggy-deploys/app-v1.2.3.zip \
  --deployment-config-name CodeDeployDefault.AllAtOnceBlueGreen

# Wait for deployment to complete:
watch -n 5 'aws deploy get-deployment \
  --deployment-id <DEPLOYMENT_ID_FROM_ABOVE> \
  --query "deploymentInfo.Status"'

# When it shows "Succeeded", rollback is done
```

#### Step 4: Monitor the rollback

```bash
# Error rate should drop immediately
aws logs tail /aws/ecs/swiggy-app --follow

# Check if alarms go green
aws cloudwatch describe-alarms \
  --alarm-names swiggy-app-error-rate-critical \
  --query 'MetricAlarms[0].StateValue'

# Should show "OK" (green)
```

### Post-Rollback

```bash
# Once rolled back and stable:
1. Create an issue: "Rollback from version X: <reason>"
2. Assign to @app-dev-oncall for root cause analysis
3. Schedule postmortem (STEP 5)
4. Do NOT re-deploy same code until issue is fixed
```

**Success Criteria for Rollback:**

- Error rate drops from >5% to <1% within 2 minutes
- All alarms go green
- Users stop reporting issues
- No new errors in logs

---

## STEP 5 - POSTMORTEM: Documentation Template

**Complete this document within 24 hours of incident resolution.**

### POSTMORTEM TEMPLATE

---

**Incident Title**: [INSERT TITLE]  
**Date**: [YYYY-MM-DD]  
**Severity**: [SEV-1 / SEV-2 / SEV-3]  
**Owner**: [Your name]  
**Duration**: [Start time] to [End time] UTC = [X] minutes

---

#### SECTION 1: Executive Summary

**Write this section LAST, after you've filled in everything else.**

This section is a 2-3 sentence summary for non-technical people (CEO, CFO).

**Template:**

- What happened: "[System] experienced [problem] at [time]"
- Impact: "[X] customers affected, [Y] orders lost, ₹[Z] revenue impact"
- Root cause: "[System component] failed because [reason]"

**Example:**
"Our payment system experienced a database connection pool exhaustion at 8:15 PM IST during the World Cup promotional event. This affected 2.3 million users for 45 minutes, causing approximately ₹189 crore in lost revenue. The root cause was a Razorpay API degradation that caused payment requests to hold database connections for longer than expected, starving the connection pool."

---

#### SECTION 2: Timeline

**Write this section DURING the incident, then update after.**

Format: [Time] [Actor] [Action/Event]

**Example:**

```
08:00 UTC  System  Promotion notification sent to 180M users
08:01 UTC  Metrics Alarm 1 (DB connections) fires - connections at 95/100
08:02 UTC  Oncall  Diagnosis: Razorpay API is slow (200ms → 2000ms)
08:03 UTC  Oncall  Payment requests now holding connections for 2000ms instead of 200ms
08:04 UTC  Oncall  Decision: Scale payment workers from 20 to 50 instances
08:06 UTC  System  Payment workers scaled. SQS queue depth starts decreasing.
08:15 UTC  System  Alarm 1 goes green. DB connections drop to 60/100.
08:16 UTC  Oncall  Error rate drops. Users stop reporting issues.
08:20 UTC  Oncall  Declare incident resolved.
```

**Instructions for filling this in:**

1. Get Slack transcript from #swiggy-incidents channel
2. Cross-reference with CloudWatch logs timestamps
3. Include EVERY action taken, not just successful ones
4. Include every alarm that fired
5. Include external events (Razorpay alert, AWS status page, etc.)

---

#### SECTION 3: Impact Assessment

**Instructions**: Fill in actual numbers, not estimates.

```
Metric                          Value
─────────────────────────────────────────────────
Duration (minutes)              [  ]
Peak RPS affected               [  ] out of 1.2M
Customers affected              [  ] million
Orders failed/cancelled          [  ]
Revenue lost                     ₹[  ] crore
Reputation damage                [Describe]
Data loss                        [Yes/No - list if yes]
SLA breach                       [Yes/No]
```

**Example:**

```
Metric                          Value
─────────────────────────────────────────────────
Duration (minutes)              45
Peak RPS affected               ~600k out of 1.2M
Customers affected              2.3 million
Orders failed/cancelled          150,000
Revenue lost                     ₹189 crore
Reputation damage               Viral Twitter complaints (#SwiggyCrash)
Data loss                        No
SLA breach                       Yes (99.9% SLA breached)
```

---

#### SECTION 4: Root Cause Analysis

**Instructions**: Answer these questions:

**Primary Root Cause:**

- What is the direct cause of the incident?
- Not "traffic spike" (that's expected) - what actually broke?

**Example:**
"Razorpay API degraded: their p95 response time increased from 200ms to 2000ms. This caused payment request handlers to hold database connections for 10x longer than expected. Our system did not queue payment requests asynchronously; they were processed synchronously, blocking the connection pool. When 60,000 payments/second hit the system, all connections were held by payment requests, and other API queries couldn't execute."

**Contributing Factors:** (things that made it worse)

1. [Factor 1]
2. [Factor 2]
3. [Factor 3]

**Example:**

1. "We did not have a circuit breaker for Razorpay API. If API is slow, we keep trying instead of failing fast."
2. "Payment processing was synchronous (as noted in architecture). We added SQS queuing, but this was the first time the code path was actually tested at scale."
3. "No alerting on Razorpay API latency - we should have detected the degradation and automatically scaled payment workers."

**Trigger:** (what caused the problem to happen right now)

- What event made this problem surface?

**Example:**
"India vs Pakistan World Cup Final match generated the largest promotion spike in our history: 180M notifications, 14.4M concurrent users. This spike exceeded Razorpay's capacity threshold. Their p95 latency went from 200ms to 2000ms, which we didn't handle gracefully."

---

#### SECTION 5: What Went Well

**Instructions**: List 3-5 things that helped mitigate the incident.

**Example:**

1. "CloudFront CDN prevented static asset requests from overwhelming the API servers. Otherwise, the NIC would have been saturated within 30 seconds."
2. "PgBouncer connection pooling multiplexed our 100 available connections to serve 500+ client connections. Without it, the incident would have been 5x worse."
3. "Auto-scaling triggered automatically and scaled to 40 instances within 2 minutes. This prevented cascading failures."
4. "Redis cache served 70% of read requests, preventing 70% of database queries. Otherwise, connections would have exhausted even faster."
5. "Payment worker service was isolated. Payment slowness didn't crash the main API, so users could still browse even if payments were slow."

---

#### SECTION 6: What Went Wrong (Besides Root Cause)

**Instructions**: List 3-5 things that made the incident worse.

**Example:**

1. "No circuit breaker on Razorpay API. We kept retrying even when their service was degraded, making the problem worse."
2. "No alerting on Razorpay latency. We didn't notice their degradation until our system was already cascading."
3. "Incident response was slow. It took 5 minutes to diagnose (should have been 30 seconds). This lost time meant the incident compounded."
4. "Documentation was vague. The oncall engineer didn't immediately know to check Razorpay's status page."
5. "No observability on payment queue depth in the main dashboards. Payment workers were falling behind, but no one knew."

---

#### SECTION 7: Action Items

**Instructions**: For each action item:

- What needs to be fixed?
- Who is responsible? (assign a person, not a team)
- When is it due?
- Link to the ticket

| Action Item   | Owner   | Due Date | Ticket |
| ------------- | ------- | -------- | ------ |
| [Description] | @person | [Date]   | [Link] |

**Example:**
| Add Razorpay API latency alerting | @bhavin-gupta | 2024-05-20 | SWG-4521 |
| Implement circuit breaker pattern for external APIs | @priya-sharma | 2024-06-01 | SWG-4522 |
| Increase auto-scaling max capacity from 50 to 100 instances | @rajesh-patel | 2024-05-18 | SWG-4523 |
| Document Razorpay degradation playbook | @deepak-singh | 2024-05-15 | SWG-4524 |
| Increase RDS connection pool from 100 to 200 | @neha-gupta | 2024-05-25 | SWG-4525 |

**Instructions for owners:**

- Each owner must sign off on their action item's due date
- If date is not feasible, negotiate NOW, don't skip it
- Link the ticket in the Jira comment section

---

#### SECTION 8: Lessons Learned

**Instructions**: What will we do differently next time?

**Template:**

- "We learned that [X]. Next time, we will [Y]."

**Example:**

1. "We learned that third-party API degradation can cascade through our entire system. Next time, we will implement circuit breakers and graceful degradation for ALL external API calls."
2. "We learned that single-region deployments cannot handle regional traffic spikes. We should add multi-region failover."
3. "We learned that observability on payment processing is critical. We added real-time dashboards for payment queue depth, latency, and error rate."
4. "We learned that 45-minute outages cost ₹189 crore. The infrastructure scaling investment ($4,887/month) is the cheapest possible insurance."

---

#### SECTION 9: Prevention Measures (Already Implemented)

**Instructions**: What did we deploy to prevent this exact incident?

**Example:**

1. "Deployed PgBouncer connection pooler (increases effective connections from 100 to 500)"
2. "Moved payment processing to SQS queue + async workers (no longer blocks main API)"
3. "Added CloudFront CDN for all static assets (saves 57.6 Gbps of bandwidth)"
4. "Implemented Redis caching (70% fewer database queries)"
5. "Deployed multi-instance auto-scaling (1 instance → 50 instances as needed)"

---

#### SECTION 10: Approval

```
Incident Lead: ________________  Date: ________
Engineering Manager: ________________  Date: ________
Exec Sponsor: ________________  Date: ________
```

---

## FINAL CHECKLIST: Before You Finish Your Shift

- [ ] Incident status: **RESOLVED** (or escalated with reason)
- [ ] Postmortem created (if severity SEV-1 or SEV-2)
- [ ] All action items assigned with due dates
- [ ] Slack notification sent to #swiggy-team: "Incident resolved at [TIME]. Details: [LINK TO POSTMORTEM]"
- [ ] Hand-off to next oncall engineer documented in Slack thread
- [ ] Log rotated so next incident starts fresh (no carryover from this one)

---

## Oncall Contact Information

| Team                | Slack Channel   | Phone         |
| ------------------- | --------------- | ------------- |
| Platform (main API) | #swiggy-oncall  | +91-XXXXXXXXX |
| Database            | #db-oncall      | +91-XXXXXXXXX |
| Payment Systems     | #payment-oncall | +91-XXXXXXXXX |
| Infrastructure      | #infra-oncall   | +91-XXXXXXXXX |
| DevOps              | #devops-oncall  | +91-XXXXXXXXX |

**If you're unsure, just page @platform-oncall. They can route you to the right team.**

---

## Emergency Escalation Path

```
Incident is getting worse (not improving after 5 min)?
    ↓
Page: @platform-oncall
    ↓
Still failing after 10 min?
    ↓
Page: @engineering-manager
    ↓
Still failing after 15 min?
    ↓
Page: @vp-engineering (executive escalation)
```

**DO NOT attempt to fix alone for more than 10 minutes. Ask for help early.**

---

**Document Version**: 1.0  
**Last Updated**: May 2026  
**Next Review**: November 2026
