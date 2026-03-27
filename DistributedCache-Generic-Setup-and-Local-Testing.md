# Distributed Cache (Redis) Generic Beginner + Production Guide


This guide is intentionally beginner-first.

It assumes you are brand new to distributed caching and want a copy-paste path to get a working local setup in .NET, then safely deploy to production.

Goal:
- Start Redis locally
- Connect a .NET API to Redis using IDistributedCache
- Store and read values
- Confirm TTL expiration works
- Verify behavior when Redis is down
- Deploy and operate Redis safely in production

---

## 0) What distributed cache is (in plain English)

A distributed cache is a shared memory store that sits outside your app.

Why use it:
- Faster reads than hitting the database for every request
- Shared across all app instances (important for load-balanced apps)
- Helps reduce database load and cost

Why Redis:
- Very common for distributed caching
- Fast and mature
- Works well with .NET built-in caching abstractions

---

## 1) Prerequisites

### 1.1 Install .NET SDK (8.0+ recommended)

Why:
- You need it to build and run the sample API.

Check:
- Run: dotnet --version
- If command fails, install from Microsoft .NET downloads.

### 1.2 Install Docker Desktop

Why:
- Easiest way to run Redis locally without manual server installation.

Check:
- Run: docker --version
- Run: docker ps

If docker ps fails:
- Start Docker Desktop app and wait until it says running.

### 1.3 (Optional but helpful) Install RedisInsight GUI

Why:
- Lets you see keys and TTL visually.

You can skip this and use redis-cli commands instead.

---

## 2) Run Redis locally

### 2.1 Pull and start Redis container

Command:

  docker run --name redis-local -p 6379:6379 -d redis:7

Why:
- Starts Redis in background and maps local port 6379.

### 2.2 Verify Redis is alive

Command:

  docker exec -it redis-local redis-cli ping

Expected output:
- PONG

Why:
- Confirms Redis server is reachable and healthy.

### 2.3 If container already exists

Use one of these:

- Start existing container:

  docker start redis-local

- Remove and recreate:

  docker rm -f redis-local
  docker run --name redis-local -p 6379:6379 -d redis:7

Why:
- Avoids confusion from stale container state.

---

## 3) Create a brand new .NET Web API

### 3.1 Create project

Commands:

  mkdir RedisCacheDemo
  cd RedisCacheDemo
  dotnet new webapi

Why:
- Creates a clean baseline app where you can test caching quickly.

### 3.2 Add required packages

Commands:

  dotnet add package Microsoft.Extensions.Caching.StackExchangeRedis
  dotnet add package Microsoft.Extensions.Caching.Abstractions

Why:
- First package integrates Redis with .NET distributed cache.
- Second package contains cache abstractions.

---

## 4) Add Redis settings

Edit appsettings.json and add:

{
  "Redis": {
    "ConnectionString": "localhost:6379",
    "InstanceName": "RedisCacheDemo-local"
  }
}

Why:
- Keeps Redis details in config instead of hardcoding in code.
- InstanceName prefixes keys, helps avoid collisions.

---

## 5) Register Redis distributed cache in DI

In Program.cs, add service registration before app build:

builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration["Redis:ConnectionString"];
    options.InstanceName = builder.Configuration["Redis:InstanceName"];
});

Why:
- Registers IDistributedCache so your services/controllers can inject it.
- InstanceName isolates your app keys.

---

## 6) Add strongly typed helper extensions (recommended)

Create file: DistributedCacheExtensions.cs

Content:

using Microsoft.Extensions.Caching.Distributed;
using System.Text;
using System.Text.Json;

public static class DistributedCacheExtensions
{
    public static async Task SetJsonAsync<T>(
        this IDistributedCache cache,
        string key,
        T value,
        DistributedCacheEntryOptions options,
        CancellationToken cancellationToken = default)
    {
        var bytes = Encoding.UTF8.GetBytes(JsonSerializer.Serialize(value));
        await cache.SetAsync(key, bytes, options, cancellationToken);
    }

    public static async Task<T?> GetJsonAsync<T>(
        this IDistributedCache cache,
        string key,
        CancellationToken cancellationToken = default)
    {
        var bytes = await cache.GetAsync(key, cancellationToken);
        if (bytes == null) return default;
        return JsonSerializer.Deserialize<T>(Encoding.UTF8.GetString(bytes));
    }
}

Why:
- IDistributedCache works with byte arrays by default.
- These helpers let you store/read typed objects directly.

---

## 7) Add a simple cache-aside demo service

Create file: ProductCacheService.cs

Content:

using Microsoft.Extensions.Caching.Distributed;

public record Product(int Id, string Name, decimal Price);

public class ProductCacheService
{
    private readonly IDistributedCache _cache;

    public ProductCacheService(IDistributedCache cache)
    {
        _cache = cache;
    }

    public async Task<Product> GetProductAsync(int id, CancellationToken ct)
    {
        var key = $"products:id:{id}";

        var cached = await _cache.GetJsonAsync<Product>(key, ct);
        if (cached != null)
        {
            return cached;
        }

        // Simulate expensive DB call
        await Task.Delay(500, ct);
        var productFromDb = new Product(id, $"Product-{id}", 9.99m + id);

        await _cache.SetJsonAsync(
            key,
            productFromDb,
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromSeconds(20)
            },
            ct);

        return productFromDb;
    }
}

Why:
- Demonstrates cache-aside pattern: read cache -> miss -> load source -> set cache.

---

## 8) Register and expose API endpoint

### 8.1 Register service in Program.cs

Add:

builder.Services.AddScoped<ProductCacheService>();

Why:
- Makes service available via dependency injection.

### 8.2 Add endpoint in Program.cs

Add after app creation:

app.MapGet("/products/{id:int}", async (int id, ProductCacheService service, CancellationToken ct) =>
{
    var start = DateTime.UtcNow;
    var product = await service.GetProductAsync(id, ct);
    var elapsedMs = (DateTime.UtcNow - start).TotalMilliseconds;

    return Results.Ok(new
    {
        Product = product,
        ElapsedMs = elapsedMs
    });
});

Why:
- Gives you a quick way to see first-call (cache miss) vs second-call (cache hit) timing.

---

## 9) Run and test locally

### 9.1 Start API

Command:

  dotnet run

Why:
- Runs your app and hosts the test endpoint.

### 9.2 Call endpoint first time

PowerShell command:

  Invoke-RestMethod "http://localhost:5000/products/42"

Expected:
- Higher ElapsedMs (because simulated DB delay on cache miss).

### 9.3 Call endpoint second time quickly

PowerShell command:

  Invoke-RestMethod "http://localhost:5000/products/42"

Expected:
- Lower ElapsedMs (cache hit).

Why:
- Confirms data is being served from Redis.

### 9.4 Wait for TTL to expire and call again

Wait about 20+ seconds, then call again.

Expected:
- Higher ElapsedMs again (entry expired, cache miss).

Why:
- Confirms expiration behavior works.

---

## 10) Verify keys exist in Redis

### 10.1 Open redis-cli inside container

Command:

  docker exec -it redis-local redis-cli

### 10.2 Find keys

Command in redis-cli:

  KEYS *

You should see something like:
- RedisCacheDemo-localproducts:id:42

Why:
- Confirms InstanceName prefix and key naming.

### 10.3 Check remaining TTL

Command in redis-cli:

  TTL RedisCacheDemo-localproducts:id:42

Why:
- Shows seconds until expiration.

### 10.4 Exit redis-cli

Command:

  exit

---

## 11) Test failure behavior (important)

### 11.1 Stop Redis while API is running

Command:

  docker stop redis-local

### 11.2 Call API again

Expected:
- Without fallback handling, API may throw errors.

Why:
- You must define what your app should do when cache is unavailable.

Recommended behavior:
- For non-critical cache paths: catch cache exceptions and continue with DB.
- For consistency-sensitive paths: fail safe (for example force primary reads).

### 11.3 Restart Redis

Command:

  docker start redis-local

---

## 12) Add timeout protection (production-friendly)

Use a short timeout for cache calls so Redis latency does not block requests.

Pattern:

var timeoutCts = new CancellationTokenSource();
timeoutCts.CancelAfter(TimeSpan.FromMilliseconds(100));
var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(timeoutCts.Token, requestCt);

Then use linkedCts.Token in cache get/set calls.

Why:
- Keeps API responsive even if Redis is slow.

---

## 13) Local test checklist

You are done when all are true:

1. Redis ping returns PONG.
2. First API call is slower than second call.
3. After TTL expires, call is slow again.
4. Redis key is visible with expected prefix.
5. TTL command shows countdown.
6. With Redis stopped, app fallback behavior is understood and acceptable.

---

## 14) Beginner troubleshooting

### Problem: Connection refused to Redis

Checks:
- Is Docker Desktop running?
- Does docker ps show redis-local?
- Is connection string localhost:6379?

### Problem: No key appears in Redis

Checks:
- Did your endpoint actually call SetJsonAsync?
- Is key format exactly what you are searching?
- Is TTL too short and key already expired?

### Problem: Every request is slow

Checks:
- Is cache read code path reached before DB call?
- Are you generating a different key every request?
- Is serialization throwing and being swallowed?

### Problem: Data stale too long

Fixes:
- Reduce TTL.
- Add explicit invalidation on update events.
- Consider versioned keys.

---

## 15) Suggested next upgrades

1. Add logging for hit, miss, set, timeout, and exception.
2. Add metrics dashboard (hit ratio, Redis latency).
3. Add integration tests that spin up Redis with Docker.
4. Add key naming conventions doc for team consistency.
5. Add resilience policy and fallback standards per endpoint criticality.

---

## 16) One-screen quick copy setup

If you only want minimum to start:

1. Start Redis: docker run --name redis-local -p 6379:6379 -d redis:7
2. Add package: dotnet add package Microsoft.Extensions.Caching.StackExchangeRedis
3. Register in Program.cs with localhost:6379
4. Inject IDistributedCache in service
5. Implement cache-aside pattern
6. Set absolute TTL (start with 20-60 seconds)
7. Call endpoint twice and compare timing

That is enough to prove distributed caching is working locally.

---

## 17) Production path (from local to real servers)

You now have local proof. Next is production hardening.

Rule of thumb:
- Local proves correctness.
- Production setup proves reliability and security.

---

## 18) Pick a production Redis model

### 18.1 Managed Redis (recommended for most teams)

Examples:
- Azure Cache for Redis / Azure Managed Redis
- AWS ElastiCache Redis
- GCP Memorystore Redis

Why:
- Managed services reduce operational burden (patching, failover, backup).

### 18.2 Self-hosted Redis on VMs

Why:
- More control, but much more operational responsibility.

Use only if you have clear requirements that managed Redis cannot satisfy.

### 18.3 Redis on Kubernetes

Why:
- Works if your platform standard is Kubernetes.

Caution:
- Stateful workloads in Kubernetes require strong SRE practices.

---

## 19) Security baseline (must do)

### 19.1 Private network only

Do:
- Keep Redis private (VNET/VPC/subnet only).
- Allow access only from application subnets.

Why:
- Public Redis endpoints are high-risk.

### 19.2 TLS in transit

Do:
- Enable TLS for production Redis connections.
- Use secure port (commonly 6380 in managed Redis).

Why:
- Protects credentials and payloads in transit.

### 19.3 Secrets in a secret manager

Do:
- Store Redis credentials in Key Vault/Secrets Manager.
- Grant read access only to app runtime identity.

Why:
- Avoids leaking secrets in source code or config files.

### 19.4 Environment isolation

Do:
- Use separate credentials per environment.
- Use environment-specific `InstanceName` prefixes.

Why:
- Prevents cross-environment key collisions and mistakes.

---

## 20) Capacity and reliability planning

### 20.1 Memory planning

Use rough estimate:

Estimated Memory = (Avg Value Size + Key Size + Overhead) * Key Count * Safety Factor

Start with safety factor 1.3 to 1.5.

Why:
- Under-sized Redis causes evictions and unstable hit rate.

### 20.2 Eviction policy

Common choices:
- allkeys-lru
- volatile-lru
- noeviction

For general cache use, `allkeys-lru` is usually practical.

Why:
- Keeps cache functioning under memory pressure.

### 20.3 High availability and backups

Do:
- Enable replication/failover.
- Enable backups.
- Test restore process regularly.

Why:
- Single-node + no restore test is operationally fragile.

---

## 21) Production .NET configuration

### 21.1 Connection string pattern

Example:

mycache.redis.cache.windows.net:6380,password=<secret>,ssl=True,abortConnect=False

Why:
- Includes TLS and auth required in production.

### 21.2 Program.cs registration

```csharp
builder.Services.AddStackExchangeRedisCache(options =>
{
  options.Configuration = builder.Configuration["Redis:ConnectionString"];
  options.InstanceName = builder.Configuration["Redis:InstanceName"];
});
```

Why:
- Standard DI registration for `IDistributedCache`.

### 21.3 Timeout + fallback policy

Use short timeout with linked cancellation token and define fallback policy per endpoint.

```csharp
var timeoutCts = new CancellationTokenSource();
timeoutCts.CancelAfter(TimeSpan.FromMilliseconds(redisTimeoutMs));
var linked = CancellationTokenSource.CreateLinkedTokenSource(timeoutCts.Token, requestCt);
```

Fallback policy:
- Non-critical endpoints: fail-open (ignore cache error, use source-of-truth).
- Consistency-sensitive endpoints: fail-safe (choose stricter path).

Why:
- Prevents cache outages from turning into API outages.

---

## 22) Step-by-step production rollout

### 22.1 Step 1: Provision Redis securely

Do:
1. Create production Redis instance.
2. Enable private networking.
3. Enable TLS and auth.
4. Configure maxmemory and eviction policy.
5. Enable backups.

Why:
- Creates the secure operational base.

### 22.2 Step 2: Add secrets

Do:
1. Save connection string in secret manager.
2. Grant app identity read-only access.

Why:
- Avoids plaintext secrets in app config.

### 22.3 Step 3: Configure app settings

Do:
1. Set `Redis:ConnectionString` from secret reference.
2. Set `Redis:InstanceName` for prod key namespace.
3. Set timeout and TTL values.

Why:
- Keeps behavior explicit and controlled per environment.

### 22.4 Step 4: Deploy to staging first

Do:
1. Deploy same config model to staging.
2. Validate hit/miss/expiration behavior.
3. Validate fallback behavior by forcing Redis failure.

Why:
- Reduces production rollout risk.

### 22.5 Step 5: Canary to production

Do:
1. Route small traffic percent to new release.
2. Watch latency, errors, hit ratio, evictions.
3. Increase traffic gradually.

Why:
- Limits blast radius if config is wrong.

### 22.6 Step 6: Full cutover and watch window

Do:
1. Move full traffic.
2. Monitor aggressively for first hours.
3. Keep rollback plan ready.

Why:
- Most incidents show up shortly after cutover.

---

## 23) Production verification checklist

All should be true:

1. Redis is private-only.
2. TLS is enforced.
3. Secrets come from secret manager.
4. InstanceName is environment-specific.
5. Hit/miss/TTL behavior validated in staging.
6. Fallback behavior validated with forced Redis outage.
7. Monitoring and alerts are active.
8. Backup restore process is tested.

---

## 24) What to monitor in production

Track:

1. Cache hit ratio
2. Cache miss ratio
3. Cache latency (P50/P95/P99)
4. Timeout count
5. Exception count
6. Memory usage
7. Eviction count

Why:
- You cannot tune a cache you cannot see.

---

## 25) Production runbooks you should have

Create runbooks for:

1. Redis unreachable
2. High Redis latency
3. Memory pressure / eviction spike
4. Credential rotation
5. Backup restore

Each runbook should define:
- Detection signals
- Immediate mitigation
- Rollback criteria
- Communication path

Why:
- During incidents, clear runbooks prevent slow or risky decisions.

---

## 26) Common production mistakes

Avoid these:

1. Public Redis endpoint
2. No TLS/auth in production
3. Reusing same namespace across environments
4. No timeout on cache calls
5. Treating cache as guaranteed source-of-truth
6. No fallback behavior
7. No eviction/hit-ratio monitoring

---

## 27) Recommended starting values

Starting points:

1. Redis timeout:
- 25 to 100 ms on hot APIs

2. TTL:
- 5 to 30 seconds for dynamic responses
- 5 to 30 minutes for reference data

3. Eviction:
- `allkeys-lru` for cache-heavy workloads

4. Key naming:
- `<env>:<service>:<domain>:<entity>:<id>`
- Example: `prod:customer-api:customer:get:12345`

---

## 28) Single-guide completion checklist

You are done if you can prove all of this:

1. Local Redis flow works end-to-end.
2. TTL and cache hit/miss behavior is verified.
3. App remains usable when Redis is down (by design).
4. Production deployment path is documented and rehearsed.
5. Monitoring and alerts are active before go-live.

If you can check all five, you are in a strong place to ship distributed caching safely.
