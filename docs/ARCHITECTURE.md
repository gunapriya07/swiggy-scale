# SwiftEats Monolith - Pre-Scaling Analysis

## System Understanding

### Current Architecture
- **Compute**: Single Node.js Express process on 1 CPU core, 4GB RAM
- **Database**: PostgreSQL with `max_connections = 100`
- **Caching**: None
- **CDN**: None - all static assets (images) served from the same Node.js process
- **Load Balancing**: None - single IP, single process
- **Scaling**: No auto-scaling capability
- **External Dependencies**: Razorpay payment gateway (synchronous, 200-2000ms per call)

---

## The Catastrophic Scenario

**Event**: India vs Pakistan World Cup Final, 8 PM IST  
**Promotion**: 50% off all orders sent to 180M users via push notification  
**Expected engagement**: ~8% CTR = **14 million users opening the app in 60 seconds**

### Request Volume Math
- **14 million requests in 60 seconds** = **~233,000 requests per second**
- This is 233 times the typical system load

---

## Critical Analysis: What Breaks and In What Order

### The Hard Limits (Ranked by Impact)

#### 1. **Database Connection Pool - FIRST TO FAIL** 
- **Limit**: 100 concurrent connections
- **Incoming requests**: 233,000 per second
- **Ratio**: 0.043% of peak requests can get a connection
- **What happens**: 
  - t=0-1s: All 100 connections consumed
  - t=1s+: 232,900+ requests per second are queuing in Node.js memory, waiting for a connection
  - Requests start timing out after `connectionAcquire` timeout (typically 30s-60s)
  - Users see "Connection refused" or "ECONNREFUSED" errors

**Why this matters**: This is the absolute bottleneck. Even if CPU and RAM were infinite, you physically cannot process more than ~100 sequential database operations at a time.

---

#### 2. **Synchronous Payment Processing - Creates Connection Starvation** 
- **Scenario**: 10-15% of traffic is checkout requests (1.4-2.1 million payment requests in 60s)
- **Per request**: 200-2000ms holds a DB connection open while waiting for Razorpay
- **Connection hold time**: Assume 500ms average per payment call
- **Impact**: 
  - 1.4M payment requests × 0.5s = 700,000 connection-seconds needed
  - But we only have 100 connections × 60 seconds = 6,000 connection-seconds available
  - Payment requests alone need 116x more connection time than available
  - **Result**: Non-payment requests starve because connections never release; they're stuck waiting on Razorpay

---

#### 3. **Single CPU Core - Process Bottleneck** 
- **Limit**: 1 CPU core
- **Node.js baseline**: ~1,000-10,000 req/s per core *under optimal conditions* (minimal I/O, simple operations)
- **Our conditions**: Complex DB queries, payment calls, image serving, all on 1 core
- **Expected throughput**: ~500-2,000 req/s realistically
- **Ratio**: 233,000 ÷ 2,000 = **116x overload**
- **What happens**:
  - CPU goes to 100% immediately
  - Event loop blocked by synchronous operations (payment calls)
  - All requests experience multi-second latencies
  - Garbage collection becomes aggressive and frequent, causing "stop-the-world" pauses

---

#### 4. **RAM Exhaustion - OOM Crash Incoming** 
- **Available**: 4GB
- **Node.js overhead**: ~300-500MB
- **Request queue**: Each queued request takes ~50-100KB in memory (headers, body, metadata)
- **Worst case**: 232,900 queued requests × 100KB = **23GB needed** for queue alone
- **What happens**:
  - RAM fills up in seconds (before any connections even release)
  - Garbage collection can't keep up
  - Node.js crashes with `FATAL ERROR: CALL_AND_RETRY_LAST Allocation failed - JavaScript heap out of memory`
  - **t~3-5s**: Process dies

---

#### 5. **No CDN - Static Assets Starve the API** 
- **Static file requests**: Assume 30-40% of requests are for images/static assets
- **Volume**: ~70 million image requests in 60 seconds
- **Node.js cost per image**: Disk I/O + HTTP serialization + network buffer
- **Impact**: Each image request takes up a precious event loop tick and a file descriptor
- **Cascading effect**: Image requests fail → users see broken images + retry → more requests

---

#### 6. **No Load Balancer - All-or-Nothing Failure** 
- **Single IP**: One process serves everything
- **Process crash**: All 14 million users get disconnected simultaneously
- **No graceful degradation**: No failover, no circuit breaker
- **Recovery**: Process restart takes 5-10s. During that time, 100% traffic loss.

---

## Failure Cascade Timeline

```
t=0s:    Notification sent to 180M users
         │
t=0.5s:  First wave hits the server (~100k req/s)
         ├─ DB connection pool instantly saturated (all 100 connections taken)
         ├─ Remaining ~99,900 req/s queuing in Node.js event loop
         └─ CPU jumps to 100%
         │
t=1-2s:  Second wave hits (~150k req/s new requests + retries)
         ├─ Memory usage climbing fast (queued requests consuming RAM)
         ├─ Payment requests hitting Razorpay, holding connections open
         ├─ No connection released = no request dequeued
         └─ Event loop completely blocked
         │
t=2-3s:  Memory pressure critical
         ├─ GC running constantly but can't reclaim fast enough
         ├─ Heap bloated with queued request objects
         └─ Latencies now in 10-30 second range
         │
t=3-5s:  Out of Memory
         ├─ Node.js process crashes
         ├─ All 14M connected users disconnected
         └─ Load balancer detects dead process (no load balancer to detect it!)
         │
t=5-10s: Manual intervention needed
         ├─ Process restart
         ├─ Database still maxed at 100 connections (slow query stack)
         └─ System back online but recovering
         │
t=10-30s: Thundering herd problem
         ├─ All retrying users hit the server again
         ├─ Same cascade repeats
         └─ Repeated crashes until spike subsides
```

---

## The Weakest Component: **Database Connection Pool**

### Why?
The DB connection limit (100) is the **absolute hard ceiling** that no amount of CPU, RAM, or network optimization can overcome. It's a finite resource that depletes at a rate of 233,000 requests per second to fulfill.

**Analogy**: Imagine a single checkout counter at a supermarket. 233,000 customers arrive in 1 minute but only 1 person can checkout at a time (if payments weren't slow). With synchronous payment calls, it's like that 1 person stops and calls the bank for every transaction while staying at the counter.

### Secondary Failure Points (in order):
1. **Synchronous payment calls** - Hold connections for 200-2000ms = massive bottleneck
2. **Single CPU core** - Can't process the load even if connections were infinite
3. **4GB RAM** - Not enough to queue 14M requests
4. **No load balancer** - Single point of catastrophic failure

---

## What Fails First: **100% Connection Pool Exhaustion**

Within **1 second** of the spike, all 100 database connections are consumed. After that:
- No new DB queries can execute
- All requests queue in Node.js memory (bleeding into #2 and #3 failures)
- System enters a death spiral: requests pile up → memory fills → GC thrashes → CPU stuck → process crashes

The system has **no graceful degradation path**. It's binary: working or crashed.

---

## Key Takeaway

This monolith doesn't just need optimization—it needs **fundamental architectural transformation**:
- ✗ Cannot handle 233k req/s with 1 CPU, 1 process, 100 DB connections
- ✗ Synchronous payment processing turns every checkout into a connection leak
- ✗ No caching, no CDN, no load balancing means zero resilience
- ✓ Requires: Database scaling, asynchronous processing, caching layer, load balancing, CDN

The system **fails within 1-5 seconds** of the spike, not gradually, but catastrophically.
