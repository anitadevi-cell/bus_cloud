# Hybrid Caching Strategy for ASP.NET Bus Booking System

This document describes a practical hybrid caching strategy for a bus booking platform to reduce supplier API calls, improve response time, and handle repeated search traffic efficiently.

## 1. What Hybrid Cache Means

Hybrid cache means using two cache layers together:

- **L1 Cache:** `IMemoryCache`
  - Fast local in-process cache
  - Best for hot and repeated requests on the same application instance
- **L2 Cache:** Shared distributed-style cache store
  - Can be MongoDB or Redis
  - Shared across application instances

## 2. Recommended Flow

User Search Request  
↓  
Generate normalized cache key  
↓  
Check `IMemoryCache`  
↓  
If miss, check shared cache store  
↓  
If miss, acquire request lock  
↓  
Call supplier API  
↓  
Store in shared cache  
↓  
Store in memory cache  
↓  
Return response

## 3. Recommended Cache Layers

### Option A: Best for Scale
- `IMemoryCache`
- Redis

### Option B: Good Starting Point
- `IMemoryCache`
- MongoDB TTL collection

## 4. Benefits

- Reduces repeated supplier API calls
- Improves response time
- Reduces CPU and database load
- Improves hot route performance
- Helps protect against supplier rate limits

## 5. Cache Domains

Use different cache policies for different data types:

- **Bus search results**: short TTL
- **Seat map / inventory**: very short TTL
- **Boarding points**: medium TTL
- **Static supplier metadata**: long TTL

## 6. Example Cache Key

```text
bussearch:v1:abhibus:1001:2002:20260525:sleeper:inr
```

## 7. Practical Recommendations

- Keep memory cache TTL shorter than shared cache TTL
- Normalize keys using lowercase and trimmed values
- Cache only transformed DTOs
- Use short negative caching for empty results
- Avoid caching full raw supplier payloads
- Add logging and metrics for hit/miss ratios

## 8. Suggested TTLs

| Data Type | Memory TTL | Shared Cache TTL |
| --- | --- | --- |
| Bus search | 30-60 sec | 3-5 min |
| Seat map | 10-20 sec | 30-45 sec |
| Boarding points | 5-10 min | 30-60 min |
| Supplier metadata | 30 min | 6-24 hr |

## 9. Limitations

- `IMemoryCache` is local to one app instance
- MongoDB is slower than Redis for high-QPS cache workloads
- Local request locks do not prevent duplicate requests across multiple servers

## 10. Final Recommendation

If you already use MongoDB, start with:

- `IMemoryCache` as L1
- MongoDB as L2

If traffic grows significantly later, move L2 to Redis while keeping the same cache service abstraction.
