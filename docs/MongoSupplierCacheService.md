# MongoSupplierCacheService for Bus Booking System

This document provides a production-ready MongoDB-based hybrid cache design for an ASP.NET bus booking system using:

- `IMemoryCache` as L1 cache
- MongoDB as L2 shared cache store
- Local request locking to reduce duplicate supplier API calls
- Negative caching for short-lived empty results

## 1. Recommended Architecture

Use a hybrid cache approach:

- **L1 Cache:** `IMemoryCache`
  - Ultra-fast local reads
  - Best for hot data
- **L2 Cache:** MongoDB TTL collection
  - Shared cache across app instances
  - Stores cache documents with expiry metadata
- **Origin:** Supplier API / database

### Request Flow

1. Check `IMemoryCache`
2. If miss, check MongoDB cache collection
3. If miss, acquire local request lock
4. Recheck memory and MongoDB
5. Fetch from supplier API
6. Store in MongoDB
7. Store in `IMemoryCache`
8. Return result

## 2. Recommended Folder Structure

```text
Caching/
├── Interfaces/
│   ├── ISupplierCacheService.cs
├── Models/
│   ├── CachePolicy.cs
│   ├── CacheDocument.cs
├── Helpers/
│   ├── CacheKeyHelper.cs
├── Locks/
│   ├── RequestLockProvider.cs
├── Services/
│   ├── MongoSupplierCacheService.cs
```

## 3. Interface

```csharp
public interface ISupplierCacheService
{
    Task<T?> GetOrCreateAsync<T>(
        string cacheKey,
        string supplierId,
        string dataType,
        Func<CancellationToken, Task<T?>> factory,
        CachePolicy policy,
        CancellationToken cancellationToken = default);

    Task RemoveAsync(
        string cacheKey,
        CancellationToken cancellationToken = default);
}
```

## 4. Cache Policy

```csharp
public sealed class CachePolicy
{
    public TimeSpan MemoryTtl { get; init; } = TimeSpan.FromMinutes(1);
    public TimeSpan MongoTtl { get; init; } = TimeSpan.FromMinutes(5);
    public TimeSpan? NegativeCacheTtl { get; init; } = TimeSpan.FromSeconds(30);
    public bool CacheEmptyResults { get; init; } = true;
}
```

## 5. Cache Document Model

```csharp
using MongoDB.Bson;

public sealed class CacheDocument
{
    public string Id { get; set; } = default!;
    public string SupplierId { get; set; } = default!;
    public string DataType { get; set; } = default!;
    public string SchemaVersion { get; set; } = "v1";
    public DateTime CreatedAtUtc { get; set; }
    public DateTime ExpireAtUtc { get; set; }
    public bool IsNegativeCache { get; set; }
    public BsonDocument? Payload { get; set; }
}
```

## 6. Request Lock Provider

```csharp
using System.Collections.Concurrent;

public static class RequestLockProvider
{
    private static readonly ConcurrentDictionary<string, SemaphoreSlim> Locks = new();

    public static SemaphoreSlim GetLock(string key)
        => Locks.GetOrAdd(key, _ => new SemaphoreSlim(1, 1));
}
```

## 7. MongoSupplierCacheService

```csharp
using Microsoft.Extensions.Caching.Memory;
using Microsoft.Extensions.Logging;
using MongoDB.Bson;
using MongoDB.Bson.Serialization;
using MongoDB.Driver;

public sealed class MongoSupplierCacheService : ISupplierCacheService
{
    private readonly IMemoryCache _memoryCache;
    private readonly IMongoCollection<CacheDocument> _collection;
    private readonly ILogger<MongoSupplierCacheService> _logger;

    public MongoSupplierCacheService(
        IMemoryCache memoryCache,
        IMongoDatabase mongoDatabase,
        ILogger<MongoSupplierCacheService> logger)
    {
        _memoryCache = memoryCache;
        _collection = mongoDatabase.GetCollection<CacheDocument>("cache_entries");
        _logger = logger;
    }

    public async Task<T?> GetOrCreateAsync<T>(
        string cacheKey,
        string supplierId,
        string dataType,
        Func<CancellationToken, Task<T?>> factory,
        CachePolicy policy,
        CancellationToken cancellationToken = default)
    {
        if (_memoryCache.TryGetValue(cacheKey, out T? memoryValue))
        {
            _logger.LogInformation("Memory cache hit. Key: {CacheKey}", cacheKey);
            return memoryValue;
        }

        var mongoValue = await TryGetMongoValueAsync<T>(cacheKey, cancellationToken);
        if (mongoValue.HasValue)
        {
            _memoryCache.Set(cacheKey, mongoValue.Value, policy.MemoryTtl);
            _logger.LogInformation("Mongo cache hit. Key: {CacheKey}", cacheKey);
            return mongoValue.Value;
        }

        var semaphore = RequestLockProvider.GetLock(cacheKey);
        await semaphore.WaitAsync(cancellationToken);

        try
        {
            if (_memoryCache.TryGetValue(cacheKey, out memoryValue))
            {
                _logger.LogInformation("Memory cache hit after lock. Key: {CacheKey}", cacheKey);
                return memoryValue;
            }

            mongoValue = await TryGetMongoValueAsync<T>(cacheKey, cancellationToken);
            if (mongoValue.HasValue)
            {
                _memoryCache.Set(cacheKey, mongoValue.Value, policy.MemoryTtl);
                _logger.LogInformation("Mongo cache hit after lock. Key: {CacheKey}", cacheKey);
                return mongoValue.Value;
            }

            _logger.LogInformation("Cache miss. Fetching source. Key: {CacheKey}", cacheKey);

            var result = await factory(cancellationToken);

            if (result is null)
            {
                if (policy.CacheEmptyResults && policy.NegativeCacheTtl.HasValue)
                {
                    await UpsertAsync(
                        cacheKey: cacheKey,
                        supplierId: supplierId,
                        dataType: dataType,
                        payload: null,
                        ttl: policy.NegativeCacheTtl.Value,
                        isNegativeCache: true,
                        cancellationToken: cancellationToken);

                    _logger.LogInformation("Negative cache stored. Key: {CacheKey}", cacheKey);
                }

                return default;
            }

            await UpsertAsync(
                cacheKey: cacheKey,
                supplierId: supplierId,
                dataType: dataType,
                payload: result,
                ttl: policy.MongoTtl,
                isNegativeCache: false,
                cancellationToken: cancellationToken);

            _memoryCache.Set(cacheKey, result, policy.MemoryTtl);

            _logger.LogInformation("Cache populated. Key: {CacheKey}", cacheKey);

            return result;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Cache error. Key: {CacheKey}", cacheKey);
            throw;
        }
        finally
        {
            semaphore.Release();
        }
    }

    public async Task RemoveAsync(
        string cacheKey,
        CancellationToken cancellationToken = default)
    {
        _memoryCache.Remove(cacheKey);

        var filter = Builders<CacheDocument>.Filter.Eq(x => x.Id, cacheKey);
        await _collection.DeleteOneAsync(filter, cancellationToken);

        _logger.LogInformation("Cache removed. Key: {CacheKey}", cacheKey);
    }

    private async Task<CacheReadResult<T>> TryGetMongoValueAsync<T>(
        string cacheKey,
        CancellationToken cancellationToken)
    {
        var filter = Builders<CacheDocument>.Filter.Eq(x => x.Id, cacheKey);
        var doc = await _collection.Find(filter).FirstOrDefaultAsync(cancellationToken);

        if (doc is null)
            return CacheReadResult<T>.Miss();

        if (doc.ExpireAtUtc <= DateTime.UtcNow)
            return CacheReadResult<T>.Miss();

        if (doc.IsNegativeCache)
            return CacheReadResult<T>.Hit(default);

        if (doc.Payload is null)
            return CacheReadResult<T>.Miss();

        try
        {
            var value = BsonSerializer.Deserialize<T>(doc.Payload);
            return CacheReadResult<T>.Hit(value);
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "Failed to deserialize cache entry. Key: {CacheKey}", cacheKey);
            return CacheReadResult<T>.Miss();
        }
    }

    private async Task UpsertAsync<T>(
        string cacheKey,
        string supplierId,
        string dataType,
        T? payload,
        TimeSpan ttl,
        bool isNegativeCache,
        CancellationToken cancellationToken)
    {
        var now = DateTime.UtcNow;

        var doc = new CacheDocument
        {
            Id = cacheKey,
            SupplierId = supplierId,
            DataType = dataType,
            SchemaVersion = "v1",
            CreatedAtUtc = now,
            ExpireAtUtc = now.Add(ttl),
            IsNegativeCache = isNegativeCache,
            Payload = payload is null ? null : payload.ToBsonDocument()
        };

        var filter = Builders<CacheDocument>.Filter.Eq(x => x.Id, cacheKey);

        await _collection.ReplaceOneAsync(
            filter,
            doc,
            new ReplaceOptions { IsUpsert = true },
            cancellationToken);
    }

    private readonly struct CacheReadResult<T>
    {
        public bool HasValue { get; }
        public T? Value { get; }

        private CacheReadResult(bool hasValue, T? value)
        {
            HasValue = hasValue;
            Value = value;
        }

        public static CacheReadResult<T> Hit(T? value) => new(true, value);
        public static CacheReadResult<T> Miss() => new(false, default);
    }
}
```

## 8. Cache Key Helper

```csharp
public static class CacheKeyHelper
{
    public static string GenerateBusSearchKey(
        string version,
        string supplierId,
        string sourceId,
        string destinationId,
        DateTime journeyDate,
        string? seatType,
        string? currency)
    {
        return string.Join(":",
            "bussearch",
            Normalize(version),
            Normalize(supplierId),
            Normalize(sourceId),
            Normalize(destinationId),
            journeyDate.ToString("yyyyMMdd"),
            NormalizeOrDefault(seatType),
            NormalizeOrDefault(currency));
    }

    private static string Normalize(string value)
        => value.Trim().ToLowerInvariant();

    private static string NormalizeOrDefault(string? value)
        => string.IsNullOrWhiteSpace(value) ? "_" : value.Trim().ToLowerInvariant();
}
```

## 9. MongoDB Indexes

Create indexes for cache expiry and filtering:

```javascript
db.cache_entries.createIndex(
  { ExpireAtUtc: 1 },
  { expireAfterSeconds: 0 }
)

db.cache_entries.createIndex({ SupplierId: 1, DataType: 1 })
```

## 10. Dependency Injection Setup

```csharp
using MongoDB.Driver;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddMemoryCache();

builder.Services.AddSingleton<IMongoClient>(_ =>
    new MongoClient(builder.Configuration.GetConnectionString("MongoDb")));

builder.Services.AddSingleton(sp =>
{
    var client = sp.GetRequiredService<IMongoClient>();
    return client.GetDatabase("BusBookingDb");
});

builder.Services.AddScoped<ISupplierCacheService, MongoSupplierCacheService>();
```

### appsettings.json

```json
{
  "ConnectionStrings": {
    "MongoDb": "mongodb://localhost:27017"
  }
}
```

## 11. Example Usage in Supplier Search

```csharp
public sealed class SupplierSearchService
{
    private readonly ISupplierCacheService _cacheService;
    private readonly ISupplierApi _supplierApi;

    public SupplierSearchService(
        ISupplierCacheService cacheService,
        ISupplierApi supplierApi)
    {
        _cacheService = cacheService;
        _supplierApi = supplierApi;
    }

    public Task<List<BusTrip>?> SearchAsync(
        SearchRequest request,
        CancellationToken cancellationToken = default)
    {
        var cacheKey = CacheKeyHelper.GenerateBusSearchKey(
            version: "v1",
            supplierId: request.SupplierId,
            sourceId: request.SourceId,
            destinationId: request.DestinationId,
            journeyDate: request.JourneyDate,
            seatType: request.SeatType,
            currency: request.Currency);

        var policy = new CachePolicy
        {
            MemoryTtl = TimeSpan.FromMinutes(1),
            MongoTtl = TimeSpan.FromMinutes(5),
            NegativeCacheTtl = TimeSpan.FromSeconds(30),
            CacheEmptyResults = true
        };

        return _cacheService.GetOrCreateAsync(
            cacheKey,
            request.SupplierId,
            "bussearch",
            ct => _supplierApi.SearchAsync(request, ct),
            policy,
            cancellationToken);
    }
}
```

## 12. Suggested TTLs

- **Bus search:** memory 30-60 sec, MongoDB 3-5 min
- **Seat map:** memory 10-20 sec, MongoDB 30-45 sec
- **Boarding points:** memory 5-10 min, MongoDB 30-60 min
- **Static supplier metadata:** memory 30 min, MongoDB 6-24 hr

## 13. Limitations and Suggestions

### Limitations
- MongoDB is slower than Redis for very high-QPS caching
- TTL cleanup is not immediate; always validate `ExpireAtUtc` in code
- `SemaphoreSlim` lock is local to one application instance

### Suggestions
- Cache only transformed DTOs, not full raw supplier payloads
- Keep memory TTL shorter than MongoDB TTL
- Add metrics for hit/miss ratio and supplier latency
- For high traffic later, replace MongoDB L2 with Redis without changing the interface

## 14. Final Recommendation

`MongoSupplierCacheService` is a good starting point if:

- You already use MongoDB
- You want a shared cache without introducing Redis immediately
- Your traffic is moderate

For very high traffic or multi-node scale, consider moving the L2 cache to Redis later while keeping the same `ISupplierCacheService` abstraction.
