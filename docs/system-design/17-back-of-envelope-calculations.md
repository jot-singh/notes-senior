# Back-of-Envelope Calculations

## What You'll Learn

- Capacity estimation techniques
- QPS and throughput calculations
- Storage and bandwidth requirements  
- Memory and CPU estimation
- Common approximations and rules of thumb

## Why This Matters

System design interviews and real architecture decisions require quick estimations. How many servers do you need? What's the database size in 5 years? Can your cache fit in memory? Back-of-envelope calculations help make informed decisions without perfect information. Companies like Google, Facebook, and Amazon rely on these estimates daily for capacity planning.

## Powers of Two Table

Essential for quick calculations:

| Power | Exact Value | Approximate | Short |
|-------|-------------|-------------|-------|
| 10 | 1,024 | 1 thousand | 1 KB |
| 20 | 1,048,576 | 1 million | 1 MB |
| 30 | 1,073,741,824 | 1 billion | 1 GB |
| 40 | 1,099,511,627,776 | 1 trillion | 1 TB |
| 50 | 1,125,899,906,842,624 | 1 quadrillion | 1 PB |

## Latency Numbers

Every programmer should know (2024 updated):

| Operation | Latency | Relative |
|-----------|---------|----------|
| L1 cache reference | 0.5 ns | 1x |
| Branch mispredict | 5 ns | 10x |
| L2 cache reference | 7 ns | 14x |
| Mutex lock/unlock | 100 ns | 200x |
| Main memory reference | 100 ns | 200x |
| Read 1MB sequentially from SSD | 1 ms | 2,000,000x || SSD random read | 150 
| Disk seek | 10 ms | 20,000,000x |
| Read 1MB sequentially from disk | 30 ms | 60,000,000x |
| Send packet CA to Netherlands | 150 ms | 300,000,000x |

## Availability Numbers

| Availability | Downtime/Year | Downtime/Month | Downtime/Week |
|--------------|---------------|----------------|---------------|
| 99% | 3.65 days | 7.2 hours | 1.68 hours |
| 99.9% | 8.76 hours | 43.8 minutes | 10.1 minutes |
| 99.99% | 52.56 minutes | 4.32 minutes | 1.01 minutes |
| 99.999% | 5.26 minutes | 25.9 seconds | 6.05 seconds |

## Step-by-Step Estimation Process

### 1. Clarify Requirements

Ask about scale, features, and constraints.

### 2. Make Assumptions

Document all assumptions explicitly.

### 3. Calculate

Use powers of 2 and round numbers.

### 4. Validate

Sanity check the results.

## Example: Twitter-like System

### Traffic Estimation

**Assumptions**:
- 300M monthly active users (MAU)
- 50% daily active (150M DAU)
- Each user reads 50 tweets/day
- Each user posts 2 tweets/day
- 10% of tweets have media

**Read QPS**:
```
150M DAU  50 reads/day = 7.5B reads/day
7.5B / 86,400 seconds = ~87,000 RPS (reads per second)
Peak (3x average) = 260,000 RPS
```

**Write QPS**:
```
150M DAU  2 writes/day = 300M writes/day
300M / 86,400 = ~3,500 WPS
Peak = 10,500 WPS
```

### Storage Estimation

**Tweet storage**:
```
Average tweet: 300 bytes (text + metadata)
Media tweets: 10% have 2MB photo average

Daily new tweets = 300M
Text storage/day = 300M  300 bytes = 90GB
Media storage/day = 300M  10%  2MB = 60TB

Total/day = ~60TB (dominated by media)
5 years = 60TB  365  5 = ~110PB
```

**Metadata storage**:
```
User: 1KB per user
300M users  1KB = 300GB

Tweet metadata: 200 bytes
300M tweets/day  200 bytes = 60GB/day
5 years = 60GB  365  5 = ~110TB
```

### Bandwidth Estimation

**Incoming**:
```
60TB/day media uploads
60TB / 86,400 seconds = ~700MB/second
Peak = 2.1GB/second
```

**Outgoing (reads > writes)**:
```
87K RPS reads
10% with media = 8.7K RPS with 2MB photos
8.7K  2MB = ~17GB/second
Peak = 51GB/second
```

### Memory (Cache) Estimation

```
80/20 rule: 20% of tweets generate 80% of traffic

Cache hot tweets from past 24 hours:
300M tweets/day  20% = 60M tweets
60M  300 bytes = 18GB (text only)

Cache hot media:
60M  10%  2MB = 12TB (impractical)
Use CDN for media instead
```

### Server Estimation

**Application servers**:
```
Assume 1 server handles 1,000 RPS

Read servers:
260K peak RPS / 1K per server = 260 servers
Add 20% buffer = 312 servers

Write servers:
10.5K peak WPS / 1K per server = 11 servers
Add 20% buffer = 14 servers
```

**Database servers**:
```
Write-heavy: Shard by user_id

300M tweets/day  365  5 years = 550B tweets
Each tweet 500 bytes = 275TB

Assume 10TB per database server
275TB / 10TB = 28 shards minimum
Add replication (3x) = 84 database servers
```

## Example: URL Shortener

### Traffic Estimation

**Assumptions**:
- 100M new URLs per month
- Read:Write ratio = 100:1

**Write QPS**:
```
100M URLs/month
100M / (30  86,400) = ~40 WPS
```

**Read QPS**:
```
40 WPS  100 = 4,000 RPS
Peak = 12,000 RPS
```

### Storage Estimation

```
Each URL entry:
- Short code: 6 bytes (base62)
- Original URL: 500 bytes average
- Metadata: 50 bytes
Total: ~600 bytes

100M URLs/month  600 bytes = 60GB/month
10 years = 60GB  12  10 = 7.2TB
```

### Short Code Space

```
Base62: [a-zA-Z0-9] = 62 characters
6 characters = 62^6 = 56.8 billion combinations

At 100M URLs/month:
56.8B / 100M = 568 months = ~47 years
```

### Bandwidth

**Incoming**:
```
40 WPS  600 bytes = 24KB/second
```

**Outgoing**:
```
4K RPS  600 bytes = 2.4MB/second
Peak = 7.2MB/second
```

### Cache

```
80/20 rule: Cache 20% of URLs

Daily URLs = 100M / 30 = 3.3M
Cache = 3.3M  20%  600 bytes = ~400MB
Very manageable
```

## Estimation Shortcuts

### Rule of 72

Doubling time = 72 / growth_rate

If traffic grows 10% per year: 72/10 = 7.2 years to double

### 86,400 Seconds Per Day

Easy conversions:
- Per second to per day: 86,400 (~100K)
- Per day to per second: ,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,400

### Memory Rule of Thumb

1M items  1KB each = 1GB

### Database Sizing

- Transactional DB: 1-10TB per server
- Analytics DB: 10-100TB per server
- NoSQL: 100TB+ per server possible

### Network

- 1Gbps link = 125MB/second
- 10Gbps link = 1.25GB/second

## Common Pitfalls

 **Forgetting peak traffic**: Use 2-3x average for capacity

 **Ignoring replication**: Storage needs 2-3x for redundancy

 **No growth factor**: Plan for 2-3 years of growth

 **Mixing units**: Be consistent (bytes vs bits, MB vs MiB)

 **Over-precision**: Round to 1-2 significant figures

## Best Practices

**State assumptions clearly**: Write them down

**Use round numbers**: 100M not 97.3M

**Check reasonableness**: Does 1PB cache make sense?

**Consider costs**: Cloud storage ~$0.02/GB/month

**Plan for growth**: 2-3x expected growth

## Real-World Examples

**YouTube**: 500 hours uploaded per minute = 30K hours/hour. At 1GB/hour = 30TB/hour ingress

**Netflix**: 200M subscribers  2 hours/day  3GB/hour = 1.2PB/day egress

**WhatsApp**: 2B users  50 messages/day  100 bytes = 10TB/day messages

**Instagram**: 1B users  10 photos/day  10%  2MB = 2PB/day

## Related Topics

- [Database Scaling Strategies](03-database-scaling-strategies.md)
- [Caching Strategies](02-caching-strategies.md)
- [Load Balancing](01-load-balancing.md)
- [CDN and Content Delivery](04-cdn-content-delivery.md)
