# Rate Limiting Policies Guide for Application-Level Implementation

## Overview
This document provides a comprehensive analysis of major rate limiting policies available for application-level implementation in the Subscription Adapter API. Each policy type has distinct characteristics, advantages, and optimal use cases.

## Table of Contents
- [Policy Classification](#policy-classification)
- [Fixed Window Limiter](#fixed-window-limiter)
- [Sliding Window Limiter](#sliding-window-limiter)
- [Token Bucket Limiter](#token-bucket-limiter)
- [Sliding Window Log Limiter](#sliding-window-log-limiter)
- [Concurrency Limiter](#concurrency-limiter)
- [Adaptive Rate Limiter](#adaptive-rate-limiter)
- [Hierarchical Rate Limiter](#hierarchical-rate-limiter)
- [Policy Comparison Matrix](#policy-comparison-matrix)
- [Implementation Recommendations](#implementation-recommendations)

## Policy Classification

### Time-Based Policies
- **Fixed Window**: Rate limits based on fixed time windows
- **Sliding Window**: Rate limits with rolling time windows
- **Sliding Window Log**: Precise tracking of individual request timestamps

### Token-Based Policies
- **Token Bucket**: Credit-based system with refill rates
- **Leaky Bucket**: Queue-based processing with steady output

### Resource-Based Policies
- **Concurrency Limiter**: Limits simultaneous active requests
- **Memory-Based**: Limits based on resource consumption

### Adaptive Policies
- **Circuit Breaker Integration**: Dynamic limits based on downstream health
- **Load-Based**: Adjusts limits based on system load

## Fixed Window Limiter

### Description
The Fixed Window Limiter divides time into discrete, non-overlapping windows of fixed duration. Each window has a request counter that resets to zero at the start of each new window.

### Architecture Diagram
```
Time Windows (1 minute each):
┌─────────────┬─────────────┬─────────────┬─────────────┐
│ Window 1    │ Window 2    │ Window 3    │ Window 4    │
│ 09:00-09:01 │ 09:01-09:02 │ 09:02-09:03 │ 09:03-09:04 │
├─────────────┼─────────────┼─────────────┼─────────────┤
│ Count: 45   │ Count: 32   │ Count: 18   │ Count: 0    │
│ Limit: 50   │ Limit: 50   │ Limit: 50   │ Limit: 50   │
│ Status: OK  │ Status: OK  │ Status: OK  │ Status: OK  │
└─────────────┴─────────────┴─────────────┴─────────────┘
```

### Implementation Example
```csharp
public class FixedWindowRateLimiter : IRateLimitPolicy
{
    private readonly IMemoryCache _cache;
    private readonly TimeSpan _windowSize;
    private readonly int _requestLimit;

    public FixedWindowRateLimiter(IMemoryCache cache, TimeSpan windowSize, int requestLimit)
    {
        _cache = cache;
        _windowSize = windowSize;
        _requestLimit = requestLimit;
    }

    public async Task<RateLimitResult> CheckLimitAsync(string key)
    {
        var windowStart = GetCurrentWindowStart();
        var cacheKey = $"{key}:{windowStart.Ticks}";
        
        var currentCount = _cache.Get<int>(cacheKey);
        
        if (currentCount >= _requestLimit)
        {
            return RateLimitResult.Denied(
                $"Fixed window limit exceeded: {currentCount}/{_requestLimit}",
                GetWindowReset(windowStart)
            );
        }

        // Increment counter
        _cache.Set(cacheKey, currentCount + 1, GetWindowExpiry(windowStart));
        
        return RateLimitResult.Allowed($"Request {currentCount + 1}/{_requestLimit}");
    }

    private DateTimeOffset GetCurrentWindowStart()
    {
        var now = DateTimeOffset.UtcNow;
        var windowTicks = _windowSize.Ticks;
        var windowStart = new DateTimeOffset(now.Ticks / windowTicks * windowTicks, TimeSpan.Zero);
        return windowStart;
    }

    private TimeSpan GetWindowReset(DateTimeOffset windowStart)
    {
        return windowStart.Add(_windowSize) - DateTimeOffset.UtcNow;
    }

    private DateTimeOffset GetWindowExpiry(DateTimeOffset windowStart)
    {
        return windowStart.Add(_windowSize).Add(TimeSpan.FromMinutes(1)); // Grace period
    }
}
```

### Characteristics
- **Memory Efficiency**: Very low memory usage, only stores one counter per window
- **Performance**: Excellent, O(1) lookup and update operations
- **Accuracy**: Good for most use cases, but can have edge case bursts
- **Reset Behavior**: Hard reset at window boundaries

### Advantages
- Simple to implement and understand
- Low memory and computational overhead
- Predictable behavior
- Easy to configure and tune
- Works well with distributed systems

### Disadvantages
- **Burst Problem**: Allows 2x limit across window boundaries
- **Edge Case Issues**: High traffic at window boundaries
- **Reset Spikes**: All clients can burst at window reset

### Use Cases
- **API Gateway Rate Limiting**: Basic protection against abuse
- **Public API Endpoints**: Simple, predictable limits for external clients
- **Resource Protection**: Protecting databases or external services
- **Billing-Based Limits**: Aligning with billing periods

### Configuration Example
```json
{
  "FixedWindow": {
    "WindowSize": "00:01:00",
    "RequestLimit": 100,
    "EnableBurstProtection": false,
    "GracePeriod": "00:00:30"
  }
}
```

## Sliding Window Limiter

### Description
The Sliding Window Limiter maintains a rolling time window that moves continuously with each request. It provides more accurate rate limiting by considering request distribution over time.

### Architecture Diagram
```
Sliding Window (60 seconds, moves with each request):
Current Time: 09:01:30

┌─────────────────────────────────────────────────────────────┐
│           Sliding Window (60s)                              │
│  ←─────────────────────────────────────────────────────→   │
│ 09:00:30                                           09:01:30 │
│                                                             │
│ Requests in window: 42                                      │
│ Limit: 50                                                   │
│ Status: ALLOWED                                             │
└─────────────────────────────────────────────────────────────┘

Previous requests:    ████████████████████████████
Recent requests:                                  ██████████
```

### Implementation Example
```csharp
public class SlidingWindowRateLimiter : IRateLimitPolicy
{
    private readonly IDistributedCache _cache;
    private readonly TimeSpan _windowSize;
    private readonly int _requestLimit;
    private readonly int _windowSegments;

    public SlidingWindowRateLimiter(
        IDistributedCache cache, 
        TimeSpan windowSize, 
        int requestLimit,
        int windowSegments = 10)
    {
        _cache = cache;
        _windowSize = windowSize;
        _requestLimit = requestLimit;
        _windowSegments = windowSegments;
    }

    public async Task<RateLimitResult> CheckLimitAsync(string key)
    {
        var now = DateTimeOffset.UtcNow;
        var segmentSize = TimeSpan.FromTicks(_windowSize.Ticks / _windowSegments);
        
        // Calculate current and previous window counts
        var currentWindow = GetWindowKey(key, now, segmentSize);
        var previousWindow = GetWindowKey(key, now.Subtract(segmentSize), segmentSize);
        
        var currentCount = await GetCountAsync(currentWindow);
        var previousCount = await GetCountAsync(previousWindow);
        
        // Calculate sliding window count
        var segmentProgress = GetSegmentProgress(now, segmentSize);
        var estimatedCount = (int)(previousCount * (1 - segmentProgress) + currentCount);
        
        if (estimatedCount >= _requestLimit)
        {
            return RateLimitResult.Denied(
                $"Sliding window limit exceeded: {estimatedCount}/{_requestLimit}",
                CalculateRetryAfter(now, segmentSize)
            );
        }

        // Increment current window
        await IncrementCountAsync(currentWindow, segmentSize);
        
        return RateLimitResult.Allowed($"Request {estimatedCount + 1}/{_requestLimit}");
    }

    private string GetWindowKey(string key, DateTimeOffset time, TimeSpan segmentSize)
    {
        var windowStart = new DateTimeOffset(
            time.Ticks / segmentSize.Ticks * segmentSize.Ticks, 
            TimeSpan.Zero
        );
        return $"{key}:sw:{windowStart.Ticks}";
    }

    private async Task<int> GetCountAsync(string windowKey)
    {
        var value = await _cache.GetStringAsync(windowKey);
        return int.TryParse(value, out var count) ? count : 0;
    }

    private async Task IncrementCountAsync(string windowKey, TimeSpan segmentSize)
    {
        var options = new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = _windowSize.Add(segmentSize)
        };
        
        // This is a simplified increment - in production, use atomic operations
        var current = await GetCountAsync(windowKey);
        await _cache.SetStringAsync(windowKey, (current + 1).ToString(), options);
    }

    private double GetSegmentProgress(DateTimeOffset now, TimeSpan segmentSize)
    {
        var segmentTicks = segmentSize.Ticks;
        var ticksIntoSegment = now.Ticks % segmentTicks;
        return (double)ticksIntoSegment / segmentTicks;
    }

    private TimeSpan CalculateRetryAfter(DateTimeOffset now, TimeSpan segmentSize)
    {
        var segmentProgress = GetSegmentProgress(now, segmentSize);
        return TimeSpan.FromTicks((long)(segmentSize.Ticks * (1 - segmentProgress)));
    }
}
```

### Characteristics
- **Memory Efficiency**: Moderate, stores multiple window segments
- **Performance**: Good, O(log n) operations with some computation
- **Accuracy**: High, provides smooth rate limiting
- **Reset Behavior**: Gradual, no hard resets

### Advantages
- Eliminates burst problems at window boundaries
- More accurate request distribution
- Smoother user experience
- Better protection against traffic spikes

### Disadvantages
- Higher computational complexity
- More memory usage than fixed window
- Implementation complexity
- Requires careful tuning of segments

### Use Cases
- **High-Traffic APIs**: Where smooth limiting is critical
- **User-Facing Applications**: Better user experience
- **SLA Enforcement**: Precise adherence to service agreements
- **Premium Services**: Where accuracy matters

### Configuration Example
```json
{
  "SlidingWindow": {
    "WindowSize": "00:01:00",
    "RequestLimit": 100,
    "WindowSegments": 10,
    "AccuracyLevel": "High"
  }
}
```

## Token Bucket Limiter

### Description
The Token Bucket Limiter maintains a bucket of tokens that are consumed by requests and refilled at a steady rate. It allows for burst traffic while maintaining an average rate over time.

### Architecture Diagram
```
Token Bucket Visualization:

┌─────────────────────────────────────────┐
│           Token Bucket                  │
│  Capacity: 100 tokens                   │
│  Current: 75 tokens                     │
│  ┌─────────────────────────────────┐    │
│  │ ████████████████████████░░░░░░░ │    │
│  └─────────────────────────────────┘    │
│                                         │
│  Refill Rate: 10 tokens/second          │
│  Last Refill: 2 seconds ago             │
│                                         │
│  ▲ Tokens added continuously            │
│  ▼ Tokens consumed per request          │
└─────────────────────────────────────────┘

Request Processing:
Incoming Request → Check Token Available? → Yes: Consume Token & Allow
                                        → No: Deny Request
```

### Implementation Example
```csharp
public class TokenBucketRateLimiter : IRateLimitPolicy
{
    private readonly IDistributedCache _cache;
    private readonly int _bucketCapacity;
    private readonly double _refillRate; // tokens per second
    private readonly ILogger<TokenBucketRateLimiter> _logger;

    public TokenBucketRateLimiter(
        IDistributedCache cache,
        int bucketCapacity,
        double refillRate,
        ILogger<TokenBucketRateLimiter> logger)
    {
        _cache = cache;
        _bucketCapacity = bucketCapacity;
        _refillRate = refillRate;
        _logger = logger;
    }

    public async Task<RateLimitResult> CheckLimitAsync(string key)
    {
        var bucket = await GetBucketStateAsync(key);
        var now = DateTimeOffset.UtcNow;
        
        // Calculate tokens to add based on time elapsed
        var timeSinceLastRefill = now - bucket.LastRefillTime;
        var tokensToAdd = timeSinceLastRefill.TotalSeconds * _refillRate;
        
        // Update bucket state
        bucket.TokenCount = Math.Min(_bucketCapacity, bucket.TokenCount + tokensToAdd);
        bucket.LastRefillTime = now;
        
        if (bucket.TokenCount < 1)
        {
            await SaveBucketStateAsync(key, bucket);
            
            var retryAfter = TimeSpan.FromSeconds((1 - bucket.TokenCount) / _refillRate);
            return RateLimitResult.Denied(
                $"Token bucket empty: {bucket.TokenCount:F2}/{_bucketCapacity}",
                retryAfter
            );
        }

        // Consume token
        bucket.TokenCount -= 1;
        await SaveBucketStateAsync(key, bucket);
        
        return RateLimitResult.Allowed($"Token consumed: {bucket.TokenCount:F2}/{_bucketCapacity} remaining");
    }

    private async Task<TokenBucketState> GetBucketStateAsync(string key)
    {
        var cacheKey = $"token_bucket:{key}";
        var cachedData = await _cache.GetStringAsync(cacheKey);
        
        if (string.IsNullOrEmpty(cachedData))
        {
            return new TokenBucketState
            {
                TokenCount = _bucketCapacity,
                LastRefillTime = DateTimeOffset.UtcNow
            };
        }

        try
        {
            return JsonSerializer.Deserialize<TokenBucketState>(cachedData) 
                   ?? new TokenBucketState { TokenCount = _bucketCapacity, LastRefillTime = DateTimeOffset.UtcNow };
        }
        catch (JsonException ex)
        {
            _logger.LogWarning(ex, "Failed to deserialize bucket state for key {Key}", key);
            return new TokenBucketState
            {
                TokenCount = _bucketCapacity,
                LastRefillTime = DateTimeOffset.UtcNow
            };
        }
    }

    private async Task SaveBucketStateAsync(string key, TokenBucketState bucket)
    {
        var cacheKey = $"token_bucket:{key}";
        var serializedData = JsonSerializer.Serialize(bucket);
        
        var options = new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10)
        };
        
        await _cache.SetStringAsync(cacheKey, serializedData, options);
    }

    public class TokenBucketState
    {
        public double TokenCount { get; set; }
        public DateTimeOffset LastRefillTime { get; set; }
    }
}
```

### Characteristics
- **Memory Efficiency**: Moderate, stores bucket state per client
- **Performance**: Good, O(1) operations with time calculations
- **Accuracy**: High for burst handling, good for average rate
- **Reset Behavior**: Continuous refill, no hard resets

### Advantages
- Excellent burst handling capabilities
- Smooth traffic shaping
- Intuitive credit-based model
- Naturally handles varying load patterns
- Self-healing (tokens replenish over time)

### Disadvantages
- More complex implementation
- Requires precise time tracking
- State management complexity
- Can be hard to predict exact behavior

### Use Cases
- **Bursty Workloads**: APIs with variable traffic patterns
- **Interactive Applications**: Where occasional bursts are acceptable
- **File Upload Services**: Allowing burst uploads within limits
- **Notification Systems**: Burst notifications with rate control

### Configuration Example
```json
{
  "TokenBucket": {
    "BucketCapacity": 100,
    "RefillRate": 10.0,
    "RefillInterval": "00:00:01",
    "InitialTokens": 100
  }
}
```

## Sliding Window Log Limiter

### Description
The Sliding Window Log Limiter maintains a precise log of all request timestamps within the time window. It provides the most accurate rate limiting but at the cost of higher memory usage.

### Architecture Diagram
```
Sliding Window Log (60-second window):
Current Time: 09:01:30

Request Log:
┌─────────────────────────────────────────────────────────────┐
│ Timestamp        │ Action    │ Window Status                 │
├─────────────────────────────────────────────────────────────┤
│ 09:00:35.123     │ Request   │ In Window ✓                   │
│ 09:00:42.456     │ Request   │ In Window ✓                   │
│ 09:00:55.789     │ Request   │ In Window ✓                   │
│ 09:01:05.012     │ Request   │ In Window ✓                   │
│ 09:01:15.345     │ Request   │ In Window ✓                   │
│ 09:01:25.678     │ Request   │ In Window ✓                   │
│ 09:01:30.000     │ Current   │ Window End                    │
└─────────────────────────────────────────────────────────────┘

Window: [09:00:30.000 - 09:01:30.000]
Requests in window: 6
Limit: 50
Status: ALLOWED
```

### Implementation Example
```csharp
public class SlidingWindowLogRateLimiter : IRateLimitPolicy
{
    private readonly IDistributedCache _cache;
    private readonly TimeSpan _windowSize;
    private readonly int _requestLimit;
    private readonly ILogger<SlidingWindowLogRateLimiter> _logger;

    public SlidingWindowLogRateLimiter(
        IDistributedCache cache,
        TimeSpan windowSize,
        int requestLimit,
        ILogger<SlidingWindowLogRateLimiter> logger)
    {
        _cache = cache;
        _windowSize = windowSize;
        _requestLimit = requestLimit;
        _logger = logger;
    }

    public async Task<RateLimitResult> CheckLimitAsync(string key)
    {
        var now = DateTimeOffset.UtcNow;
        var windowStart = now.Subtract(_windowSize);
        
        var requestLog = await GetRequestLogAsync(key);
        
        // Clean up old entries outside the window
        requestLog.RemoveAll(timestamp => timestamp < windowStart);
        
        if (requestLog.Count >= _requestLimit)
        {
            await SaveRequestLogAsync(key, requestLog);
            
            var oldestRequest = requestLog.Min();
            var retryAfter = oldestRequest.Add(_windowSize) - now;
            
            return RateLimitResult.Denied(
                $"Sliding window log limit exceeded: {requestLog.Count}/{_requestLimit}",
                retryAfter > TimeSpan.Zero ? retryAfter : TimeSpan.FromSeconds(1)
            );
        }

        // Add current request to log
        requestLog.Add(now);
        await SaveRequestLogAsync(key, requestLog);
        
        return RateLimitResult.Allowed($"Request logged: {requestLog.Count}/{_requestLimit}");
    }

    private async Task<List<DateTimeOffset>> GetRequestLogAsync(string key)
    {
        var cacheKey = $"window_log:{key}";
        var cachedData = await _cache.GetStringAsync(cacheKey);
        
        if (string.IsNullOrEmpty(cachedData))
        {
            return new List<DateTimeOffset>();
        }

        try
        {
            var timestamps = JsonSerializer.Deserialize<long[]>(cachedData);
            return timestamps?.Select(ticks => new DateTimeOffset(ticks, TimeSpan.Zero)).ToList() 
                   ?? new List<DateTimeOffset>();
        }
        catch (JsonException ex)
        {
            _logger.LogWarning(ex, "Failed to deserialize request log for key {Key}", key);
            return new List<DateTimeOffset>();
        }
    }

    private async Task SaveRequestLogAsync(string key, List<DateTimeOffset> requestLog)
    {
        var cacheKey = $"window_log:{key}";
        var timestamps = requestLog.Select(dt => dt.Ticks).ToArray();
        var serializedData = JsonSerializer.Serialize(timestamps);
        
        var options = new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = _windowSize.Add(TimeSpan.FromMinutes(5))
        };
        
        await _cache.SetStringAsync(cacheKey, serializedData, options);
    }
}
```

### Characteristics
- **Memory Efficiency**: Poor, stores every request timestamp
- **Performance**: Good for low traffic, degrades with high request counts
- **Accuracy**: Perfect, exact request tracking
- **Reset Behavior**: Gradual expiration of individual requests

### Advantages
- Perfect accuracy in rate limiting
- No edge cases or approximations
- Precise request distribution analysis
- Deterministic behavior

### Disadvantages
- High memory usage (O(n) where n = requests in window)
- Performance degrades with request volume
- Complex cleanup and maintenance
- Not suitable for high-traffic scenarios

### Use Cases
- **Low-Traffic Premium APIs**: Where perfect accuracy is required
- **Debugging and Analysis**: Understanding exact request patterns
- **Compliance Requirements**: Where precise tracking is mandatory
- **Research and Development**: Analyzing rate limiting effectiveness

### Configuration Example
```json
{
  "SlidingWindowLog": {
    "WindowSize": "00:01:00",
    "RequestLimit": 100,
    "MaxLogSize": 1000,
    "CleanupInterval": "00:00:30"
  }
}
```

## Concurrency Limiter

### Description
The Concurrency Limiter controls the number of simultaneous active requests rather than the rate of requests. It's ideal for protecting resources that have a limited capacity for concurrent operations.

### Architecture Diagram
```
Concurrency Control:

┌─────────────────────────────────────────────┐
│           Active Request Pool               │
│  Max Concurrent: 10                         │
│  Current Active: 7                          │
│                                             │
│  ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ │
│  │ 1 │ │ 2 │ │ 3 │ │ 4 │ │ 5 │ │ 6 │ │ 7 │ │
│  └───┘ └───┘ └───┘ └───┘ └───┘ └───┘ └───┘ │
│                                             │
│  Available Slots: 3                         │
└─────────────────────────────────────────────┘

Request Flow:
Incoming Request → Check Available Slots → Yes: Acquire Slot & Process
                                       → No: Queue or Reject

Request Complete → Release Slot → Next Queued Request
```

### Implementation Example
```csharp
public class ConcurrencyRateLimiter : IRateLimitPolicy
{
    private readonly IDistributedCache _cache;
    private readonly IDistributedLockProvider _lockProvider;
    private readonly int _maxConcurrency;
    private readonly TimeSpan _requestTimeout;
    private readonly ILogger<ConcurrencyRateLimiter> _logger;

    public ConcurrencyRateLimiter(
        IDistributedCache cache,
        IDistributedLockProvider lockProvider,
        int maxConcurrency,
        TimeSpan requestTimeout,
        ILogger<ConcurrencyRateLimiter> logger)
    {
        _cache = cache;
        _lockProvider = lockProvider;
        _maxConcurrency = maxConcurrency;
        _requestTimeout = requestTimeout;
        _logger = logger;
    }

    public async Task<RateLimitResult> CheckLimitAsync(string key)
    {
        var requestId = Guid.NewGuid().ToString();
        var lockKey = $"concurrency_lock:{key}";
        
        using var distributedLock = await _lockProvider.AcquireLockAsync(lockKey, TimeSpan.FromSeconds(5));
        
        if (distributedLock == null)
        {
            return RateLimitResult.Denied("Unable to acquire concurrency lock", TimeSpan.FromSeconds(1));
        }

        var activeRequests = await GetActiveRequestsAsync(key);
        
        // Clean up expired requests
        await CleanupExpiredRequestsAsync(key, activeRequests);
        
        if (activeRequests.Count >= _maxConcurrency)
        {
            return RateLimitResult.Denied(
                $"Concurrency limit exceeded: {activeRequests.Count}/{_maxConcurrency}",
                TimeSpan.FromSeconds(1)
            );
        }

        // Register this request as active
        var activeRequest = new ActiveRequest
        {
            RequestId = requestId,
            StartTime = DateTimeOffset.UtcNow,
            ExpireTime = DateTimeOffset.UtcNow.Add(_requestTimeout)
        };

        activeRequests.Add(activeRequest);
        await SaveActiveRequestsAsync(key, activeRequests);
        
        return RateLimitResult.Allowed($"Slot acquired: {activeRequests.Count}/{_maxConcurrency}");
    }

    public async Task ReleaseSlotAsync(string key, string requestId)
    {
        var lockKey = $"concurrency_lock:{key}";
        
        using var distributedLock = await _lockProvider.AcquireLockAsync(lockKey, TimeSpan.FromSeconds(5));
        
        if (distributedLock == null)
        {
            _logger.LogWarning("Unable to acquire lock to release slot for request {RequestId}", requestId);
            return;
        }

        var activeRequests = await GetActiveRequestsAsync(key);
        var requestToRemove = activeRequests.FirstOrDefault(r => r.RequestId == requestId);
        
        if (requestToRemove != null)
        {
            activeRequests.Remove(requestToRemove);
            await SaveActiveRequestsAsync(key, activeRequests);
            
            _logger.LogDebug("Released concurrency slot for request {RequestId}", requestId);
        }
    }

    private async Task<List<ActiveRequest>> GetActiveRequestsAsync(string key)
    {
        var cacheKey = $"active_requests:{key}";
        var cachedData = await _cache.GetStringAsync(cacheKey);
        
        if (string.IsNullOrEmpty(cachedData))
        {
            return new List<ActiveRequest>();
        }

        try
        {
            return JsonSerializer.Deserialize<List<ActiveRequest>>(cachedData) 
                   ?? new List<ActiveRequest>();
        }
        catch (JsonException ex)
        {
            _logger.LogWarning(ex, "Failed to deserialize active requests for key {Key}", key);
            return new List<ActiveRequest>();
        }
    }

    private async Task SaveActiveRequestsAsync(string key, List<ActiveRequest> activeRequests)
    {
        var cacheKey = $"active_requests:{key}";
        var serializedData = JsonSerializer.Serialize(activeRequests);
        
        var options = new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = _requestTimeout.Add(TimeSpan.FromMinutes(5))
        };
        
        await _cache.SetStringAsync(cacheKey, serializedData, options);
    }

    private async Task CleanupExpiredRequestsAsync(string key, List<ActiveRequest> activeRequests)
    {
        var now = DateTimeOffset.UtcNow;
        var expiredRequests = activeRequests.Where(r => r.ExpireTime < now).ToList();
        
        foreach (var expired in expiredRequests)
        {
            activeRequests.Remove(expired);
            _logger.LogDebug("Cleaned up expired request {RequestId}", expired.RequestId);
        }

        if (expiredRequests.Any())
        {
            await SaveActiveRequestsAsync(key, activeRequests);
        }
    }

    public class ActiveRequest
    {
        public string RequestId { get; set; } = string.Empty;
        public DateTimeOffset StartTime { get; set; }
        public DateTimeOffset ExpireTime { get; set; }
    }
}
```

### Characteristics
- **Memory Efficiency**: Moderate, stores active request information
- **Performance**: Good, but requires distributed locking
- **Accuracy**: High for resource protection
- **Reset Behavior**: Immediate upon request completion

### Advantages
- Perfect for protecting limited resources
- Prevents resource exhaustion
- Works well with long-running requests
- Intuitive for capacity management

### Disadvantages
- Requires request completion tracking
- Complex state management
- Distributed locking overhead
- Not suitable for pure rate limiting

### Use Cases
- **Database Connection Pooling**: Limiting concurrent database queries
- **File Processing**: Limiting simultaneous file operations
- **External API Calls**: Respecting downstream concurrency limits
- **Resource-Intensive Operations**: CPU or memory intensive tasks

### Configuration Example
```json
{
  "Concurrency": {
    "MaxConcurrentRequests": 10,
    "RequestTimeout": "00:05:00",
    "CleanupInterval": "00:00:30",
    "LockTimeout": "00:00:05"
  }
}
```

## Adaptive Rate Limiter

### Description
The Adaptive Rate Limiter dynamically adjusts rate limits based on system conditions, downstream health, and observed traffic patterns. It provides intelligent rate limiting that responds to changing conditions.

### Architecture Diagram
```
Adaptive Rate Limiting System:

┌─────────────────────────────────────────────────────────────┐
│                System Health Monitor                         │
├─────────────────────────────────────────────────────────────┤
│ CPU Usage: 45%        Memory: 60%        Latency: 120ms     │
│ Error Rate: 2%        Success Rate: 98%  Queue Depth: 5     │
└─────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────┐
│               Adaptive Algorithm                             │
├─────────────────────────────────────────────────────────────┤
│ Base Limit: 100 req/min                                     │
│ Health Factor: 0.8 (Good)                                   │
│ Load Factor: 1.2 (High)                                     │
│ Error Factor: 0.9 (Low Errors)                              │
│                                                             │
│ Calculated Limit: 100 × 0.8 × 1.2 × 0.9 = 86 req/min      │
└─────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────┐
│              Rate Limit Enforcement                          │
└─────────────────────────────────────────────────────────────┘
```

### Implementation Example
```csharp
public class AdaptiveRateLimiter : IRateLimitPolicy
{
    private readonly IRateLimitPolicy _basePolicy;
    private readonly ISystemHealthMonitor _healthMonitor;
    private readonly IConfiguration _config;
    private readonly ILogger<AdaptiveRateLimiter> _logger;

    public AdaptiveRateLimiter(
        IRateLimitPolicy basePolicy,
        ISystemHealthMonitor healthMonitor,
        IConfiguration config,
        ILogger<AdaptiveRateLimiter> logger)
    {
        _basePolicy = basePolicy;
        _healthMonitor = healthMonitor;
        _config = config;
        _logger = logger;
    }

    public async Task<RateLimitResult> CheckLimitAsync(string key)
    {
        var systemHealth = await _healthMonitor.GetCurrentHealthAsync();
        var adaptiveFactor = CalculateAdaptiveFactor(systemHealth);
        
        // Modify the base policy's limit based on adaptive factor
        var adaptedPolicy = CreateAdaptedPolicy(adaptiveFactor);
        var result = await adaptedPolicy.CheckLimitAsync(key);
        
        // Log adaptation decisions
        if (Math.Abs(adaptiveFactor - 1.0) > 0.1) // Significant adaptation
        {
            _logger.LogInformation(
                "Adaptive rate limiting: Factor {Factor:F2}, Health {Health}, Key {Key}",
                adaptiveFactor, systemHealth.OverallScore, key
            );
        }

        return result;
    }

    private double CalculateAdaptiveFactor(SystemHealth health)
    {
        var factors = new List<double>();

        // CPU-based adjustment
        if (health.CpuUsage > 0.8)
        {
            factors.Add(0.5); // Severely reduce limits under high CPU
        }
        else if (health.CpuUsage > 0.6)
        {
            factors.Add(0.7); // Moderately reduce limits
        }
        else
        {
            factors.Add(1.0); // Normal limits
        }

        // Memory-based adjustment
        if (health.MemoryUsage > 0.9)
        {
            factors.Add(0.3); // Drastically reduce under memory pressure
        }
        else if (health.MemoryUsage > 0.7)
        {
            factors.Add(0.6); // Reduce limits
        }
        else
        {
            factors.Add(1.0); // Normal limits
        }

        // Error rate adjustment
        if (health.ErrorRate > 0.1) // 10% error rate
        {
            factors.Add(0.4); // Reduce limits significantly
        }
        else if (health.ErrorRate > 0.05) // 5% error rate
        {
            factors.Add(0.7); // Moderate reduction
        }
        else
        {
            factors.Add(1.0); // Normal limits
        }

        // Response time adjustment
        if (health.AverageResponseTime.TotalMilliseconds > 1000)
        {
            factors.Add(0.6); // Reduce limits for slow responses
        }
        else if (health.AverageResponseTime.TotalMilliseconds > 500)
        {
            factors.Add(0.8); // Slight reduction
        }
        else
        {
            factors.Add(1.1); // Allow slight increase for fast responses
        }

        // Calculate final factor (minimum of all factors for safety)
        var finalFactor = factors.Min();
        
        // Apply bounds
        var minFactor = _config.GetValue<double>("AdaptiveRateLimit:MinFactor", 0.1);
        var maxFactor = _config.GetValue<double>("AdaptiveRateLimit:MaxFactor", 2.0);
        
        return Math.Max(minFactor, Math.Min(maxFactor, finalFactor));
    }

    private IRateLimitPolicy CreateAdaptedPolicy(double factor)
    {
        // This is a simplified example - in practice, you'd modify the underlying policy
        if (_basePolicy is FixedWindowRateLimiter fixedWindow)
        {
            var adaptedLimit = (int)(fixedWindow.RequestLimit * factor);
            return new FixedWindowRateLimiter(
                fixedWindow.Cache,
                fixedWindow.WindowSize,
                adaptedLimit
            );
        }
        
        // Fallback to original policy
        return _basePolicy;
    }
}

public interface ISystemHealthMonitor
{
    Task<SystemHealth> GetCurrentHealthAsync();
}

public class SystemHealth
{
    public double CpuUsage { get; set; }
    public double MemoryUsage { get; set; }
    public double ErrorRate { get; set; }
    public TimeSpan AverageResponseTime { get; set; }
    public int ActiveConnections { get; set; }
    public double OverallScore { get; set; }
}

public class SystemHealthMonitor : ISystemHealthMonitor
{
    private readonly IMemoryCache _cache;
    private readonly ILogger<SystemHealthMonitor> _logger;

    public SystemHealthMonitor(IMemoryCache cache, ILogger<SystemHealthMonitor> logger)
    {
        _cache = cache;
        _logger = logger;
    }

    public async Task<SystemHealth> GetCurrentHealthAsync()
    {
        const string cacheKey = "system_health";
        
        if (_cache.TryGetValue(cacheKey, out SystemHealth? cached) && cached != null)
        {
            return cached;
        }

        var health = await CollectHealthMetricsAsync();
        
        _cache.Set(cacheKey, health, TimeSpan.FromSeconds(30)); // Cache for 30 seconds
        
        return health;
    }

    private async Task<SystemHealth> CollectHealthMetricsAsync()
    {
        // In a real implementation, collect actual system metrics
        var health = new SystemHealth
        {
            CpuUsage = await GetCpuUsageAsync(),
            MemoryUsage = await GetMemoryUsageAsync(),
            ErrorRate = await GetErrorRateAsync(),
            AverageResponseTime = await GetAverageResponseTimeAsync(),
            ActiveConnections = await GetActiveConnectionsAsync()
        };

        health.OverallScore = CalculateOverallScore(health);
        
        return health;
    }

    private async Task<double> GetCpuUsageAsync()
    {
        // Placeholder - implement actual CPU monitoring
        return 0.45; // 45% CPU usage
    }

    private async Task<double> GetMemoryUsageAsync()
    {
        // Placeholder - implement actual memory monitoring
        return 0.60; // 60% memory usage
    }

    private async Task<double> GetErrorRateAsync()
    {
        // Placeholder - implement actual error rate calculation
        return 0.02; // 2% error rate
    }

    private async Task<TimeSpan> GetAverageResponseTimeAsync()
    {
        // Placeholder - implement actual response time monitoring
        return TimeSpan.FromMilliseconds(120);
    }

    private async Task<int> GetActiveConnectionsAsync()
    {
        // Placeholder - implement actual connection monitoring
        return 150;
    }

    private double CalculateOverallScore(SystemHealth health)
    {
        var cpuScore = 1.0 - health.CpuUsage;
        var memoryScore = 1.0 - health.MemoryUsage;
        var errorScore = 1.0 - Math.Min(1.0, health.ErrorRate * 10); // Scale error rate
        var responseScore = Math.Max(0, 1.0 - (health.AverageResponseTime.TotalMilliseconds / 1000.0));

        return (cpuScore + memoryScore + errorScore + responseScore) / 4.0;
    }
}
```

### Characteristics
- **Memory Efficiency**: Moderate, caches health metrics
- **Performance**: Variable, depends on health monitoring overhead
- **Accuracy**: High adaptability to conditions
- **Reset Behavior**: Dynamic based on system state

### Advantages
- Responds intelligently to system conditions
- Prevents cascade failures
- Optimizes resource utilization
- Self-healing characteristics

### Disadvantages
- Complex implementation and tuning
- Requires comprehensive monitoring
- Can be unpredictable
- Higher computational overhead

### Use Cases
- **Microservices Architectures**: Adapting to downstream service health
- **Auto-scaling Environments**: Coordinating with scaling events
- **Variable Load Systems**: Systems with unpredictable traffic patterns
- **Critical Systems**: Where availability is paramount

### Configuration Example
```json
{
  "AdaptiveRateLimit": {
    "BasePolicy": "FixedWindow",
    "MinFactor": 0.1,
    "MaxFactor": 2.0,
    "HealthCheckInterval": "00:00:30",
    "AdaptationThreshold": 0.1,
    "Factors": {
      "CpuWeight": 0.3,
      "MemoryWeight": 0.3,
      "ErrorRateWeight": 0.2,
      "ResponseTimeWeight": 0.2
    }
  }
}
```

## Policy Comparison Matrix

| Policy Type | Memory Usage | CPU Overhead | Accuracy | Burst Handling | Complexity | Best For |
|-------------|--------------|--------------|-----------|----------------|------------|----------|
| **Fixed Window** | Very Low | Very Low | Good | Poor | Low | Simple APIs, Basic protection |
| **Sliding Window** | Low | Low | High | Good | Medium | General purpose, Smooth limiting |
| **Token Bucket** | Low | Low | High | Excellent | Medium | Bursty traffic, Interactive apps |
| **Sliding Window Log** | High | Medium | Perfect | Excellent | High | Low traffic, Perfect accuracy |
| **Concurrency** | Medium | Medium | Perfect | N/A | High | Resource protection |
| **Adaptive** | Medium | High | Variable | Variable | Very High | Dynamic environments |

## Implementation Recommendations

### For Subscription Adapter API

#### Primary Recommendation: **Hybrid Approach**
Combine multiple policies for optimal protection:

```csharp
public class HybridRateLimiter : IRateLimitPolicy
{
    private readonly TokenBucketRateLimiter _burstLimiter;
    private readonly SlidingWindowRateLimiter _smoothLimiter;
    private readonly ConcurrencyRateLimiter _resourceLimiter;

    public async Task<RateLimitResult> CheckLimitAsync(string key)
    {
        // Check concurrency first (most critical)
        var concurrencyResult = await _resourceLimiter.CheckLimitAsync(key);
        if (!concurrencyResult.IsAllowed)
            return concurrencyResult;

        // Check burst capacity
        var burstResult = await _burstLimiter.CheckLimitAsync(key);
        if (!burstResult.IsAllowed)
            return burstResult;

        // Check smooth rate
        var smoothResult = await _smoothLimiter.CheckLimitAsync(key);
        return smoothResult;
    }
}
```

#### Configuration by Client Tier
```json
{
  "RateLimitPolicies": {
    "Critical": {
      "Type": "TokenBucket",
      "BucketCapacity": 200,
      "RefillRate": 50.0,
      "ConcurrencyLimit": 50
    },
    "Premium": {
      "Type": "SlidingWindow",
      "WindowSize": "00:01:00",
      "RequestLimit": 100,
      "ConcurrencyLimit": 20
    },
    "Standard": {
      "Type": "FixedWindow",
      "WindowSize": "00:01:00",
      "RequestLimit": 50,
      "ConcurrencyLimit": 10
    },
    "Trial": {
      "Type": "FixedWindow",
      "WindowSize": "00:01:00",
      "RequestLimit": 10,
      "ConcurrencyLimit": 5
    }
  }
}
```

### Selection Guidelines

#### Choose **Fixed Window** when:
- Simple rate limiting requirements
- Low computational overhead needed
- Predictable traffic patterns
- Easy configuration and monitoring

#### Choose **Sliding Window** when:
- Smooth rate limiting required
- Moderate traffic volumes
- User experience is important
- Some computational overhead acceptable

#### Choose **Token Bucket** when:
- Bursty traffic patterns expected
- Interactive user applications
- Flexibility in traffic shaping needed
- Credit-based model makes sense

#### Choose **Concurrency Limiter** when:
- Protecting limited resources
- Long-running requests
- Resource exhaustion prevention needed
- Request completion tracking possible

#### Choose **Adaptive** when:
- Dynamic environments
- Complex system dependencies
- High availability requirements
- Advanced monitoring available

### Performance Considerations

#### Memory Usage Optimization
```csharp
public class OptimizedRateLimitStore
{
    private readonly IMemoryCache _localCache;
    private readonly IDistributedCache _distributedCache;
    private readonly TimeSpan _localCacheExpiry = TimeSpan.FromSeconds(30);

    public async Task<T> GetAsync<T>(string key)
    {
        // Try local cache first
        if (_localCache.TryGetValue(key, out T localValue))
            return localValue;

        // Fallback to distributed cache
        var distributedValue = await _distributedCache.GetAsync<T>(key);
        if (distributedValue != null)
        {
            _localCache.Set(key, distributedValue, _localCacheExpiry);
        }

        return distributedValue;
    }
}
```

#### CPU Optimization
```csharp
public class BatchedRateLimiter
{
    private readonly Timer _batchTimer;
    private readonly ConcurrentQueue<RateLimitCheck> _pendingChecks;

    public BatchedRateLimiter()
    {
        _pendingChecks = new ConcurrentQueue<RateLimitCheck>();
        _batchTimer = new Timer(ProcessBatch, null, TimeSpan.FromMilliseconds(100), TimeSpan.FromMilliseconds(100));
    }

    private async void ProcessBatch(object state)
    {
        var batch = new List<RateLimitCheck>();
        while (_pendingChecks.TryDequeue(out var check) && batch.Count < 100)
        {
            batch.Add(check);
        }

        if (batch.Any())
        {
            await ProcessRateLimitBatch(batch);
        }
    }
}
```

This comprehensive guide provides the foundation for implementing sophisticated rate limiting policies in your Subscription Adapter API. Choose the appropriate policy based on your specific requirements, traffic patterns, and system constraints.
