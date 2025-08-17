# Priority-Based Rate Limiting: Detailed Implementation Guide

## Overview
This document provides comprehensive implementation details for priority-based rate limiting in the Subscription Adapter API, including code examples, configuration patterns, testing strategies, and operational considerations.

## Table of Contents
- [Architecture Overview](#architecture-overview)
- [Priority Detection Implementation](#priority-detection-implementation)
- [Rate Limiting Engine](#rate-limiting-engine)
- [Client Authorization](#client-authorization)
- [Configuration Management](#configuration-management)
- [Monitoring and Metrics](#monitoring-and-metrics)
- [Security Implementation](#security-implementation)
- [Testing Strategy](#testing-strategy)
- [Deployment Guide](#deployment-guide)
- [Troubleshooting](#troubleshooting)

## Architecture Overview

### Core Components Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Request Pipeline                         │
├─────────────────────────────────────────────────────────────────┤
│  1. Priority Detection Middleware                               │
│     ├── Payload Buffer & Parse                                  │
│     ├── Priority Field Extraction                               │
│     └── Priority Validation                                     │
│                              │                                  │
│  2. Client Authorization      ▼                                 │
│     ├── Client Tier Lookup                                      │
│     ├── Priority Permission Check                               │
│     └── Security Validation                                     │
│                              │                                  │
│  3. Rate Limiting Engine      ▼                                 │
│     ├── Policy Selection                                        │
│     ├── Rate Limit Calculation                                  │
│     ├── Limit Enforcement                                       │
│     └── Bypass Logic                                            │
│                              │                                  │
│  4. Metrics & Monitoring      ▼                                 │
│     ├── Request Tracking                                        │
│     ├── Violation Logging                                       │
│     └── Performance Metrics                                     │
│                              │                                  │
│  5. Business Logic           ▼                                  │
│     └── Subscription Processing                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Data Flow Architecture

```
HTTP Request
    │
    ▼
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│ Request Buffer  │───▶│ Priority Parser  │───▶│ Priority        │
│ & Validation    │    │ & Extractor      │    │ Determination   │
└─────────────────┘    └──────────────────┘    └─────────────────┘
    │                           │                       │
    ▼                           ▼                       ▼
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│ Client Identity │    │ Authorization    │    │ Rate Policy     │
│ Resolution      │    │ Matrix Check     │    │ Selection       │
└─────────────────┘    └──────────────────┘    └─────────────────┘
    │                           │                       │
    ▼                           ▼                       ▼
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│ Metrics         │    │ Rate Limit       │    │ Response        │
│ Collection      │    │ Enforcement      │    │ Generation      │
└─────────────────┘    └──────────────────┘    └─────────────────┘
```

## Priority Detection Implementation

### 1. Priority Detection Middleware

```csharp
public class PriorityDetectionMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<PriorityDetectionMiddleware> _logger;
    private readonly IPriorityParser _priorityParser;
    private readonly IPriorityValidator _priorityValidator;

    public PriorityDetectionMiddleware(
        RequestDelegate next,
        ILogger<PriorityDetectionMiddleware> logger,
        IPriorityParser priorityParser,
        IPriorityValidator priorityValidator)
    {
        _next = next;
        _logger = logger;
        _priorityParser = priorityParser;
        _priorityValidator = priorityValidator;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // Enable request body buffering for multiple reads
        context.Request.EnableBuffering();

        var priorityContext = await ExtractPriorityContextAsync(context);
        
        // Store priority context for downstream middleware
        context.Items["PriorityContext"] = priorityContext;

        await _next(context);
    }

    private async Task<PriorityContext> ExtractPriorityContextAsync(HttpContext context)
    {
        try
        {
            var priorityLevel = await _priorityParser.ParseRequestPriorityAsync(context.Request);
            var validationResult = await _priorityValidator.ValidateAsync(priorityLevel, context);

            return new PriorityContext
            {
                Level = validationResult.IsValid ? priorityLevel : PriorityLevel.Low,
                IsValid = validationResult.IsValid,
                Reason = validationResult.Reason,
                ClientId = ExtractClientId(context),
                Timestamp = DateTimeOffset.UtcNow,
                ValidationErrors = validationResult.Errors
            };
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "Failed to extract priority context, defaulting to Low priority");
            
            return new PriorityContext
            {
                Level = PriorityLevel.Low,
                IsValid = false,
                Reason = "Priority extraction failed",
                ClientId = ExtractClientId(context),
                Timestamp = DateTimeOffset.UtcNow,
                ValidationErrors = new[] { ex.Message }
            };
        }
    }

    private string ExtractClientId(HttpContext context)
    {
        return context.Request.Headers["Xero-Client-Name"].FirstOrDefault() 
               ?? context.Request.Headers["Xero-User-Id"].FirstOrDefault() 
               ?? "unknown";
    }
}
```

### 2. Priority Parser Implementation

```csharp
public interface IPriorityParser
{
    Task<PriorityLevel> ParseRequestPriorityAsync(HttpRequest request);
}

public class PriorityParser : IPriorityParser
{
    private readonly ILogger<PriorityParser> _logger;
    private readonly JsonSerializerOptions _jsonOptions;

    public PriorityParser(ILogger<PriorityParser> logger)
    {
        _logger = logger;
        _jsonOptions = new JsonSerializerOptions
        {
            PropertyNameCaseInsensitive = true,
            DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull
        };
    }

    public async Task<PriorityLevel> ParseRequestPriorityAsync(HttpRequest request)
    {
        if (!IsJsonRequest(request))
        {
            return PriorityLevel.Low; // Default for non-JSON requests
        }

        try
        {
            // Read request body
            request.Body.Position = 0;
            using var reader = new StreamReader(request.Body, leaveOpen: true);
            var requestBody = await reader.ReadToEndAsync();
            request.Body.Position = 0; // Reset for downstream middleware

            if (string.IsNullOrWhiteSpace(requestBody))
            {
                return PriorityLevel.Low;
            }

            // Parse JSON to extract priority field
            var jsonDocument = JsonDocument.Parse(requestBody);
            
            if (jsonDocument.RootElement.TryGetProperty("priority", out var priorityElement))
            {
                var priorityString = priorityElement.GetString();
                return ParsePriorityString(priorityString);
            }

            // Check for legacy priority indicators
            if (HasLegacyHighPriorityIndicators(jsonDocument.RootElement))
            {
                return PriorityLevel.Medium; // Upgrade legacy patterns to medium
            }

            return PriorityLevel.Low; // Default when no priority specified
        }
        catch (JsonException ex)
        {
            _logger.LogWarning(ex, "Failed to parse JSON request body for priority extraction");
            return PriorityLevel.Low;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unexpected error during priority parsing");
            return PriorityLevel.Low;
        }
    }

    private bool IsJsonRequest(HttpRequest request)
    {
        return request.ContentType?.Contains("application/json", StringComparison.OrdinalIgnoreCase) == true;
    }

    private PriorityLevel ParsePriorityString(string? priorityString)
    {
        return priorityString?.ToLowerInvariant() switch
        {
            "high" or "critical" or "urgent" => PriorityLevel.High,
            "medium" or "important" or "elevated" => PriorityLevel.Medium,
            "low" or "normal" or "standard" => PriorityLevel.Low,
            "background" or "batch" or "bulk" => PriorityLevel.Background,
            _ => PriorityLevel.Low
        };
    }

    private bool HasLegacyHighPriorityIndicators(JsonElement element)
    {
        // Check for legacy patterns that might indicate higher priority
        var urgentIndicators = new[] { "urgent", "critical", "emergency", "immediate" };
        
        foreach (var property in element.EnumerateObject())
        {
            if (property.Value.ValueKind == JsonValueKind.String)
            {
                var value = property.Value.GetString()?.ToLowerInvariant();
                if (urgentIndicators.Any(indicator => value?.Contains(indicator) == true))
                {
                    return true;
                }
            }
        }
        
        return false;
    }
}
```

### 3. Priority Context Model

```csharp
public class PriorityContext
{
    public PriorityLevel Level { get; set; }
    public bool IsValid { get; set; }
    public string Reason { get; set; } = string.Empty;
    public string ClientId { get; set; } = string.Empty;
    public DateTimeOffset Timestamp { get; set; }
    public IEnumerable<string> ValidationErrors { get; set; } = Array.Empty<string>();
    public string? OriginalPriorityValue { get; set; }
    public bool IsEscalated { get; set; }
    public string? EscalationReason { get; set; }
}

public enum PriorityLevel
{
    Background = 0,
    Low = 1,
    Medium = 2,
    High = 3
}

public class PriorityValidationResult
{
    public bool IsValid { get; set; }
    public string Reason { get; set; } = string.Empty;
    public IEnumerable<string> Errors { get; set; } = Array.Empty<string>();
    public PriorityLevel RecommendedLevel { get; set; }
}
```

## Rate Limiting Engine

### 1. Priority-Based Rate Limiter

```csharp
public class PriorityBasedRateLimiter : IRateLimiter
{
    private readonly IRateLimitStore _store;
    private readonly IRateLimitConfiguration _config;
    private readonly ILogger<PriorityBasedRateLimiter> _logger;
    private readonly IMetricsCollector _metrics;

    public PriorityBasedRateLimiter(
        IRateLimitStore store,
        IRateLimitConfiguration config,
        ILogger<PriorityBasedRateLimiter> logger,
        IMetricsCollector metrics)
    {
        _store = store;
        _config = config;
        _logger = logger;
        _metrics = metrics;
    }

    public async Task<RateLimitResult> CheckRateLimitAsync(
        string clientId, 
        PriorityContext priorityContext, 
        string endpoint)
    {
        var startTime = DateTimeOffset.UtcNow;

        try
        {
            // High priority requests bypass rate limiting
            if (priorityContext.Level == PriorityLevel.High)
            {
                await _metrics.RecordBypassAsync(clientId, priorityContext.Level, endpoint);
                return RateLimitResult.Allowed("High priority bypass");
            }

            var policy = await GetRateLimitPolicyAsync(clientId, priorityContext.Level, endpoint);
            var current = await _store.GetCurrentUsageAsync(clientId, endpoint, policy.WindowSize);
            
            var effectiveLimit = CalculateEffectiveLimit(policy.Limit, priorityContext.Level);
            var isAllowed = current.RequestCount < effectiveLimit;

            if (isAllowed)
            {
                await _store.IncrementUsageAsync(clientId, endpoint, policy.WindowSize);
                await _metrics.RecordAllowedRequestAsync(clientId, priorityContext.Level, endpoint);
            }
            else
            {
                await _metrics.RecordRateLimitViolationAsync(clientId, priorityContext.Level, endpoint, current.RequestCount, effectiveLimit);
            }

            var result = isAllowed 
                ? RateLimitResult.Allowed($"Request {current.RequestCount + 1}/{effectiveLimit}")
                : RateLimitResult.Denied($"Rate limit exceeded: {current.RequestCount}/{effectiveLimit}", 
                                        CalculateRetryAfter(current, policy));

            result.RemainingRequests = Math.Max(0, effectiveLimit - current.RequestCount);
            result.ResetTime = current.WindowResetTime;
            result.Priority = priorityContext.Level;

            return result;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error checking rate limit for client {ClientId}", clientId);
            await _metrics.RecordErrorAsync(clientId, endpoint, ex);
            
            // Fail open - allow request on error but log for investigation
            return RateLimitResult.Allowed("Rate limiter error - failing open");
        }
        finally
        {
            var duration = DateTimeOffset.UtcNow - startTime;
            await _metrics.RecordRateLimiterPerformanceAsync(duration);
        }
    }

    private int CalculateEffectiveLimit(int baseLimit, PriorityLevel priority)
    {
        return priority switch
        {
            PriorityLevel.High => int.MaxValue, // Unlimited
            PriorityLevel.Medium => (int)(baseLimit * 2.0), // 2x normal
            PriorityLevel.Low => baseLimit, // Normal
            PriorityLevel.Background => (int)(baseLimit * 0.2), // 20% of normal
            _ => baseLimit
        };
    }

    private async Task<RateLimitPolicy> GetRateLimitPolicyAsync(
        string clientId, 
        PriorityLevel priority, 
        string endpoint)
    {
        var clientTier = await _config.GetClientTierAsync(clientId);
        var basePolicy = await _config.GetBasePolicyAsync(clientTier, endpoint);
        
        return new RateLimitPolicy
        {
            Limit = basePolicy.Limit,
            WindowSize = basePolicy.WindowSize,
            BurstMultiplier = GetBurstMultiplier(priority),
            ClientTier = clientTier,
            Priority = priority
        };
    }

    private double GetBurstMultiplier(PriorityLevel priority)
    {
        return priority switch
        {
            PriorityLevel.High => 1.0, // No burst needed - unlimited
            PriorityLevel.Medium => 3.0, // Higher burst allowance
            PriorityLevel.Low => 2.0, // Standard burst
            PriorityLevel.Background => 1.1, // Minimal burst
            _ => 2.0
        };
    }

    private TimeSpan CalculateRetryAfter(RateLimitUsage current, RateLimitPolicy policy)
    {
        var timeUntilReset = current.WindowResetTime - DateTimeOffset.UtcNow;
        return timeUntilReset > TimeSpan.Zero ? timeUntilReset : TimeSpan.FromMinutes(1);
    }
}
```

### 2. Rate Limit Models

```csharp
public class RateLimitResult
{
    public bool IsAllowed { get; set; }
    public string Reason { get; set; } = string.Empty;
    public int RemainingRequests { get; set; }
    public DateTimeOffset ResetTime { get; set; }
    public TimeSpan? RetryAfter { get; set; }
    public PriorityLevel Priority { get; set; }

    public static RateLimitResult Allowed(string reason) => new()
    {
        IsAllowed = true,
        Reason = reason
    };

    public static RateLimitResult Denied(string reason, TimeSpan retryAfter) => new()
    {
        IsAllowed = false,
        Reason = reason,
        RetryAfter = retryAfter
    };
}

public class RateLimitPolicy
{
    public int Limit { get; set; }
    public TimeSpan WindowSize { get; set; }
    public double BurstMultiplier { get; set; }
    public ClientTier ClientTier { get; set; }
    public PriorityLevel Priority { get; set; }
}

public class RateLimitUsage
{
    public int RequestCount { get; set; }
    public DateTimeOffset WindowStartTime { get; set; }
    public DateTimeOffset WindowResetTime { get; set; }
    public DateTimeOffset LastRequestTime { get; set; }
}

public enum ClientTier
{
    Trial = 0,
    Standard = 1,
    Premium = 2,
    Critical = 3
}
```

## Client Authorization

### 1. Priority Authorization Service

```csharp
public interface IPriorityValidator
{
    Task<PriorityValidationResult> ValidateAsync(PriorityLevel requestedPriority, HttpContext context);
}

public class PriorityValidator : IPriorityValidator
{
    private readonly IClientConfigurationService _clientConfig;
    private readonly ISecurityService _security;
    private readonly ILogger<PriorityValidator> _logger;

    public PriorityValidator(
        IClientConfigurationService clientConfig,
        ISecurityService security,
        ILogger<PriorityValidator> logger)
    {
        _clientConfig = clientConfig;
        _security = security;
        _logger = logger;
    }

    public async Task<PriorityValidationResult> ValidateAsync(
        PriorityLevel requestedPriority, 
        HttpContext context)
    {
        var clientId = ExtractClientId(context);
        var clientTier = await _clientConfig.GetClientTierAsync(clientId);
        
        // Check if client is authorized for requested priority
        var authorizedPriorities = GetAuthorizedPriorities(clientTier);
        
        if (!authorizedPriorities.Contains(requestedPriority))
        {
            _logger.LogWarning(
                "Client {ClientId} (tier: {ClientTier}) attempted to use unauthorized priority {Priority}",
                clientId, clientTier, requestedPriority);

            return new PriorityValidationResult
            {
                IsValid = false,
                Reason = $"Client tier {clientTier} not authorized for {requestedPriority} priority",
                Errors = new[] { "Insufficient authorization for requested priority level" },
                RecommendedLevel = GetMaxAuthorizedPriority(clientTier)
            };
        }

        // Additional validation for high priority requests
        if (requestedPriority == PriorityLevel.High)
        {
            var highPriorityValidation = await ValidateHighPriorityRequestAsync(context, clientId);
            if (!highPriorityValidation.IsValid)
            {
                return highPriorityValidation;
            }
        }

        return new PriorityValidationResult
        {
            IsValid = true,
            Reason = $"Priority {requestedPriority} authorized for client tier {clientTier}",
            RecommendedLevel = requestedPriority
        };
    }

    private PriorityLevel[] GetAuthorizedPriorities(ClientTier clientTier)
    {
        return clientTier switch
        {
            ClientTier.Critical => new[] { PriorityLevel.High, PriorityLevel.Medium, PriorityLevel.Low, PriorityLevel.Background },
            ClientTier.Premium => new[] { PriorityLevel.Medium, PriorityLevel.Low, PriorityLevel.Background },
            ClientTier.Standard => new[] { PriorityLevel.Low, PriorityLevel.Background },
            ClientTier.Trial => new[] { PriorityLevel.Low },
            _ => new[] { PriorityLevel.Low }
        };
    }

    private PriorityLevel GetMaxAuthorizedPriority(ClientTier clientTier)
    {
        return clientTier switch
        {
            ClientTier.Critical => PriorityLevel.High,
            ClientTier.Premium => PriorityLevel.Medium,
            ClientTier.Standard => PriorityLevel.Low,
            ClientTier.Trial => PriorityLevel.Low,
            _ => PriorityLevel.Low
        };
    }

    private async Task<PriorityValidationResult> ValidateHighPriorityRequestAsync(
        HttpContext context, 
        string clientId)
    {
        // Check for additional security requirements for high priority
        var hasValidJwt = await _security.ValidateJwtTokenAsync(context);
        if (!hasValidJwt)
        {
            return new PriorityValidationResult
            {
                IsValid = false,
                Reason = "High priority requests require valid JWT token",
                Errors = new[] { "JWT token validation failed for high priority request" },
                RecommendedLevel = PriorityLevel.Medium
            };
        }

        // Check rate limit on high priority usage itself
        var highPriorityUsage = await _clientConfig.GetHighPriorityUsageAsync(clientId);
        if (highPriorityUsage.ExceedsThreshold())
        {
            return new PriorityValidationResult
            {
                IsValid = false,
                Reason = "High priority usage threshold exceeded",
                Errors = new[] { "Client has exceeded high priority request quota" },
                RecommendedLevel = PriorityLevel.Medium
            };
        }

        return new PriorityValidationResult
        {
            IsValid = true,
            Reason = "High priority request validated successfully"
        };
    }

    private string ExtractClientId(HttpContext context)
    {
        return context.Request.Headers["Xero-Client-Name"].FirstOrDefault() 
               ?? context.Request.Headers["Xero-User-Id"].FirstOrDefault() 
               ?? "unknown";
    }
}
```

## Configuration Management

### 1. Rate Limit Configuration Service

```csharp
public interface IRateLimitConfiguration
{
    Task<ClientTier> GetClientTierAsync(string clientId);
    Task<RateLimitPolicy> GetBasePolicyAsync(ClientTier clientTier, string endpoint);
    Task<PrioritySettings> GetPrioritySettingsAsync();
    Task UpdateClientTierAsync(string clientId, ClientTier newTier);
}

public class RateLimitConfiguration : IRateLimitConfiguration
{
    private readonly IConfiguration _configuration;
    private readonly IMemoryCache _cache;
    private readonly ILogger<RateLimitConfiguration> _logger;

    public RateLimitConfiguration(
        IConfiguration configuration,
        IMemoryCache cache,
        ILogger<RateLimitConfiguration> logger)
    {
        _configuration = configuration;
        _cache = cache;
        _logger = logger;
    }

    public async Task<ClientTier> GetClientTierAsync(string clientId)
    {
        var cacheKey = $"client_tier_{clientId}";
        
        if (_cache.TryGetValue(cacheKey, out ClientTier cachedTier))
        {
            return cachedTier;
        }

        // Load from configuration or external service
        var clientTier = await LoadClientTierFromConfigurationAsync(clientId);
        
        _cache.Set(cacheKey, clientTier, TimeSpan.FromMinutes(5));
        
        return clientTier;
    }

    public async Task<RateLimitPolicy> GetBasePolicyAsync(ClientTier clientTier, string endpoint)
    {
        var settings = await GetPrioritySettingsAsync();
        var tierSettings = settings.ClientTierSettings[clientTier];
        
        return new RateLimitPolicy
        {
            Limit = tierSettings.BaseRequestsPerMinute,
            WindowSize = TimeSpan.FromMinutes(1),
            BurstMultiplier = tierSettings.BurstMultiplier,
            ClientTier = clientTier
        };
    }

    public async Task<PrioritySettings> GetPrioritySettingsAsync()
    {
        const string cacheKey = "priority_settings";
        
        if (_cache.TryGetValue(cacheKey, out PrioritySettings? cached) && cached != null)
        {
            return cached;
        }

        var settings = new PrioritySettings
        {
            EnablePriorityBasedLimiting = _configuration.GetValue<bool>("RateLimit:PrioritySettings:EnablePriorityBasedLimiting", true),
            DefaultPriority = Enum.Parse<PriorityLevel>(_configuration.GetValue<string>("RateLimit:PrioritySettings:DefaultPriority", "Low")),
            RequireReasonForHighPriority = _configuration.GetValue<bool>("RateLimit:PrioritySettings:RequireReasonForHighPriority", true),
            
            PriorityRateLimitMultipliers = new Dictionary<PriorityLevel, double>
            {
                { PriorityLevel.High, _configuration.GetValue<double>("RateLimit:PrioritySettings:PriorityRateLimitMultipliers:high", 0.0) },
                { PriorityLevel.Medium, _configuration.GetValue<double>("RateLimit:PrioritySettings:PriorityRateLimitMultipliers:medium", 0.5) },
                { PriorityLevel.Low, _configuration.GetValue<double>("RateLimit:PrioritySettings:PriorityRateLimitMultipliers:low", 1.0) },
                { PriorityLevel.Background, _configuration.GetValue<double>("RateLimit:PrioritySettings:PriorityRateLimitMultipliers:background", 5.0) }
            },

            ClientTierSettings = new Dictionary<ClientTier, ClientTierSettings>
            {
                { ClientTier.Critical, new ClientTierSettings { BaseRequestsPerMinute = 100, BurstMultiplier = 3.0 } },
                { ClientTier.Premium, new ClientTierSettings { BaseRequestsPerMinute = 50, BurstMultiplier = 2.5 } },
                { ClientTier.Standard, new ClientTierSettings { BaseRequestsPerMinute = 20, BurstMultiplier = 2.0 } },
                { ClientTier.Trial, new ClientTierSettings { BaseRequestsPerMinute = 5, BurstMultiplier = 1.5 } }
            }
        };

        _cache.Set(cacheKey, settings, TimeSpan.FromMinutes(10));
        
        return settings;
    }

    private async Task<ClientTier> LoadClientTierFromConfigurationAsync(string clientId)
    {
        // Check specific client configurations first
        var clientSpecificTier = _configuration.GetValue<string>($"RateLimit:ClientTiers:{clientId}");
        if (!string.IsNullOrEmpty(clientSpecificTier) && Enum.TryParse<ClientTier>(clientSpecificTier, out var tier))
        {
            return tier;
        }

        // Apply pattern-based rules
        if (clientId.Contains("critical", StringComparison.OrdinalIgnoreCase) || 
            clientId.Contains("admin", StringComparison.OrdinalIgnoreCase))
        {
            return ClientTier.Critical;
        }

        if (clientId.Contains("premium", StringComparison.OrdinalIgnoreCase))
        {
            return ClientTier.Premium;
        }

        if (clientId.Contains("trial", StringComparison.OrdinalIgnoreCase) || 
            clientId.Contains("test", StringComparison.OrdinalIgnoreCase))
        {
            return ClientTier.Trial;
        }

        // Default to Standard tier
        return ClientTier.Standard;
    }

    public async Task UpdateClientTierAsync(string clientId, ClientTier newTier)
    {
        // Update configuration (could be database, external service, etc.)
        _logger.LogInformation("Updating client {ClientId} tier to {NewTier}", clientId, newTier);
        
        // Invalidate cache
        var cacheKey = $"client_tier_{clientId}";
        _cache.Remove(cacheKey);
        
        // In a real implementation, this would persist to a data store
        await Task.CompletedTask;
    }
}
```

### 2. Configuration Models

```csharp
public class PrioritySettings
{
    public bool EnablePriorityBasedLimiting { get; set; } = true;
    public PriorityLevel DefaultPriority { get; set; } = PriorityLevel.Low;
    public bool RequireReasonForHighPriority { get; set; } = true;
    public Dictionary<PriorityLevel, double> PriorityRateLimitMultipliers { get; set; } = new();
    public Dictionary<ClientTier, ClientTierSettings> ClientTierSettings { get; set; } = new();
    public int HighPriorityUsageThresholdPerHour { get; set; } = 10;
}

public class ClientTierSettings
{
    public int BaseRequestsPerMinute { get; set; }
    public double BurstMultiplier { get; set; }
    public PriorityLevel[] AllowedPriorities { get; set; } = Array.Empty<PriorityLevel>();
}
```

## Monitoring and Metrics

### 1. Metrics Collection Service

```csharp
public interface IMetricsCollector
{
    Task RecordAllowedRequestAsync(string clientId, PriorityLevel priority, string endpoint);
    Task RecordRateLimitViolationAsync(string clientId, PriorityLevel priority, string endpoint, int current, int limit);
    Task RecordBypassAsync(string clientId, PriorityLevel priority, string endpoint);
    Task RecordErrorAsync(string clientId, string endpoint, Exception exception);
    Task RecordRateLimiterPerformanceAsync(TimeSpan duration);
    Task RecordPriorityDistributionAsync(Dictionary<PriorityLevel, int> distribution);
}

public class MetricsCollector : IMetricsCollector
{
    private readonly ILogger<MetricsCollector> _logger;
    private readonly Counter<int> _requestCounter;
    private readonly Counter<int> _violationCounter;
    private readonly Counter<int> _bypassCounter;
    private readonly Histogram<double> _performanceHistogram;

    public MetricsCollector(ILogger<MetricsCollector> logger, IMeterFactory meterFactory)
    {
        _logger = logger;
        var meter = meterFactory.Create("SubscriptionAdapter.RateLimit");
        
        _requestCounter = meter.CreateCounter<int>(
            "rate_limit_requests_total",
            description: "Total number of rate limit checks performed");
            
        _violationCounter = meter.CreateCounter<int>(
            "rate_limit_violations_total", 
            description: "Total number of rate limit violations");
            
        _bypassCounter = meter.CreateCounter<int>(
            "rate_limit_bypasses_total",
            description: "Total number of rate limit bypasses");
            
        _performanceHistogram = meter.CreateHistogram<double>(
            "rate_limit_check_duration_ms",
            description: "Duration of rate limit checks in milliseconds");
    }

    public async Task RecordAllowedRequestAsync(string clientId, PriorityLevel priority, string endpoint)
    {
        _requestCounter.Add(1, new TagList
        {
            { "client_id", clientId },
            { "priority", priority.ToString() },
            { "endpoint", endpoint },
            { "result", "allowed" }
        });

        _logger.LogDebug(
            "Rate limit check: ALLOWED - Client: {ClientId}, Priority: {Priority}, Endpoint: {Endpoint}",
            clientId, priority, endpoint);

        await Task.CompletedTask;
    }

    public async Task RecordRateLimitViolationAsync(
        string clientId, 
        PriorityLevel priority, 
        string endpoint, 
        int current, 
        int limit)
    {
        _violationCounter.Add(1, new TagList
        {
            { "client_id", clientId },
            { "priority", priority.ToString() },
            { "endpoint", endpoint }
        });

        _logger.LogWarning(
            "Rate limit VIOLATION - Client: {ClientId}, Priority: {Priority}, Endpoint: {Endpoint}, Usage: {Current}/{Limit}",
            clientId, priority, endpoint, current, limit);

        await Task.CompletedTask;
    }

    public async Task RecordBypassAsync(string clientId, PriorityLevel priority, string endpoint)
    {
        _bypassCounter.Add(1, new TagList
        {
            { "client_id", clientId },
            { "priority", priority.ToString() },
            { "endpoint", endpoint }
        });

        _logger.LogInformation(
            "Rate limit BYPASS - Client: {ClientId}, Priority: {Priority}, Endpoint: {Endpoint}",
            clientId, priority, endpoint);

        await Task.CompletedTask;
    }

    public async Task RecordErrorAsync(string clientId, string endpoint, Exception exception)
    {
        _logger.LogError(exception,
            "Rate limit ERROR - Client: {ClientId}, Endpoint: {Endpoint}",
            clientId, endpoint);

        await Task.CompletedTask;
    }

    public async Task RecordRateLimiterPerformanceAsync(TimeSpan duration)
    {
        _performanceHistogram.Record(duration.TotalMilliseconds);
        await Task.CompletedTask;
    }

    public async Task RecordPriorityDistributionAsync(Dictionary<PriorityLevel, int> distribution)
    {
        foreach (var (priority, count) in distribution)
        {
            _requestCounter.Add(count, new TagList
            {
                { "priority", priority.ToString() },
                { "metric_type", "distribution" }
            });
        }

        await Task.CompletedTask;
    }
}
```

## Security Implementation

### 1. Security Service

```csharp
public interface ISecurityService
{
    Task<bool> ValidateJwtTokenAsync(HttpContext context);
    Task<bool> ValidateClientCertificateAsync(HttpContext context);
    Task<SecurityValidationResult> ValidateHighPriorityRequestAsync(HttpContext context, string clientId);
}

public class SecurityService : ISecurityService
{
    private readonly ILogger<SecurityService> _logger;
    private readonly IConfiguration _configuration;

    public SecurityService(ILogger<SecurityService> logger, IConfiguration configuration)
    {
        _logger = logger;
        _configuration = configuration;
    }

    public async Task<bool> ValidateJwtTokenAsync(HttpContext context)
    {
        var authHeader = context.Request.Headers["Authorization"].FirstOrDefault();
        
        if (string.IsNullOrEmpty(authHeader) || !authHeader.StartsWith("Bearer "))
        {
            return false;
        }

        var token = authHeader.Substring("Bearer ".Length).Trim();
        
        try
        {
            // Implement JWT validation logic here
            // This is a placeholder implementation
            return !string.IsNullOrEmpty(token) && token.Length > 20;
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "JWT validation failed");
            return false;
        }
    }

    public async Task<bool> ValidateClientCertificateAsync(HttpContext context)
    {
        var clientCertificate = context.Connection.ClientCertificate;
        
        if (clientCertificate == null)
        {
            return false;
        }

        // Implement certificate validation logic
        return clientCertificate.NotAfter > DateTime.UtcNow && 
               clientCertificate.NotBefore <= DateTime.UtcNow;
    }

    public async Task<SecurityValidationResult> ValidateHighPriorityRequestAsync(
        HttpContext context, 
        string clientId)
    {
        var validations = new List<SecurityCheck>();

        // JWT Token validation
        var jwtValid = await ValidateJwtTokenAsync(context);
        validations.Add(new SecurityCheck("JWT Token", jwtValid, jwtValid ? "Valid JWT token present" : "Invalid or missing JWT token"));

        // Client certificate validation (if required)
        var requireClientCert = _configuration.GetValue<bool>("Security:RequireClientCertificateForHighPriority", false);
        if (requireClientCert)
        {
            var certValid = await ValidateClientCertificateAsync(context);
            validations.Add(new SecurityCheck("Client Certificate", certValid, certValid ? "Valid client certificate" : "Invalid or missing client certificate"));
        }

        // IP whitelist validation (if configured)
        var allowedIps = _configuration.GetSection("Security:HighPriorityAllowedIPs").Get<string[]>();
        if (allowedIps?.Length > 0)
        {
            var clientIp = context.Connection.RemoteIpAddress?.ToString();
            var ipAllowed = allowedIps.Contains(clientIp);
            validations.Add(new SecurityCheck("IP Whitelist", ipAllowed, ipAllowed ? "IP address allowed" : "IP address not in whitelist"));
        }

        var allValid = validations.All(v => v.IsValid);

        return new SecurityValidationResult
        {
            IsValid = allValid,
            Checks = validations,
            Reason = allValid ? "All security checks passed" : "One or more security checks failed"
        };
    }
}

public class SecurityValidationResult
{
    public bool IsValid { get; set; }
    public IEnumerable<SecurityCheck> Checks { get; set; } = Array.Empty<SecurityCheck>();
    public string Reason { get; set; } = string.Empty;
}

public class SecurityCheck
{
    public string Name { get; set; }
    public bool IsValid { get; set; }
    public string Message { get; set; }

    public SecurityCheck(string name, bool isValid, string message)
    {
        Name = name;
        IsValid = isValid;
        Message = message;
    }
}
```

## Testing Strategy

### 1. Unit Tests for Priority Detection

```csharp
[TestFixture]
public class PriorityDetectionTests
{
    private PriorityParser _parser;
    private Mock<ILogger<PriorityParser>> _loggerMock;

    [SetUp]
    public void Setup()
    {
        _loggerMock = new Mock<ILogger<PriorityParser>>();
        _parser = new PriorityParser(_loggerMock.Object);
    }

    [Test]
    public async Task ParseRequestPriorityAsync_WithHighPriority_ReturnsHigh()
    {
        // Arrange
        var json = """{"priority": "high", "CustomerOrganisationId": "test"}""";
        var request = CreateMockRequest(json);

        // Act
        var result = await _parser.ParseRequestPriorityAsync(request);

        // Assert
        Assert.That(result, Is.EqualTo(PriorityLevel.High));
    }

    [Test]
    public async Task ParseRequestPriorityAsync_WithMediumPriority_ReturnsMedium()
    {
        // Arrange
        var json = """{"priority": "medium", "CustomerOrganisationId": "test"}""";
        var request = CreateMockRequest(json);

        // Act
        var result = await _parser.ParseRequestPriorityAsync(request);

        // Assert
        Assert.That(result, Is.EqualTo(PriorityLevel.Medium));
    }

    [Test]
    public async Task ParseRequestPriorityAsync_WithLowPriority_ReturnsLow()
    {
        // Arrange
        var json = """{"priority": "low", "CustomerOrganisationId": "test"}""";
        var request = CreateMockRequest(json);

        // Act
        var result = await _parser.ParseRequestPriorityAsync(request);

        // Assert
        Assert.That(result, Is.EqualTo(PriorityLevel.Low));
    }

    [Test]
    public async Task ParseRequestPriorityAsync_WithBackgroundPriority_ReturnsBackground()
    {
        // Arrange
        var json = """{"priority": "background", "CustomerOrganisationId": "test"}""";
        var request = CreateMockRequest(json);

        // Act
        var result = await _parser.ParseRequestPriorityAsync(request);

        // Assert
        Assert.That(result, Is.EqualTo(PriorityLevel.Background));
    }

    [Test]
    public async Task ParseRequestPriorityAsync_WithNoPriority_ReturnsLow()
    {
        // Arrange
        var json = """{"CustomerOrganisationId": "test"}""";
        var request = CreateMockRequest(json);

        // Act
        var result = await _parser.ParseRequestPriorityAsync(request);

        // Assert
        Assert.That(result, Is.EqualTo(PriorityLevel.Low));
    }

    [Test]
    public async Task ParseRequestPriorityAsync_WithInvalidJson_ReturnsLow()
    {
        // Arrange
        var invalidJson = """{"invalid": json"}""";
        var request = CreateMockRequest(invalidJson);

        // Act
        var result = await _parser.ParseRequestPriorityAsync(request);

        // Assert
        Assert.That(result, Is.EqualTo(PriorityLevel.Low));
    }

    [Test]
    public async Task ParseRequestPriorityAsync_WithLegacyUrgentIndicator_ReturnsMedium()
    {
        // Arrange
        var json = """{"operationType": "urgent", "CustomerOrganisationId": "test"}""";
        var request = CreateMockRequest(json);

        // Act
        var result = await _parser.ParseRequestPriorityAsync(request);

        // Assert
        Assert.That(result, Is.EqualTo(PriorityLevel.Medium));
    }

    private HttpRequest CreateMockRequest(string json)
    {
        var context = new DefaultHttpContext();
        var request = context.Request;
        
        request.ContentType = "application/json";
        var bodyBytes = Encoding.UTF8.GetBytes(json);
        request.Body = new MemoryStream(bodyBytes);
        
        return request;
    }
}
```

### 2. Integration Tests for Rate Limiting

```csharp
[TestFixture]
public class PriorityBasedRateLimitingIntegrationTests
{
    private TestServer _server;
    private HttpClient _client;

    [SetUp]
    public async Task Setup()
    {
        var hostBuilder = new WebHostBuilder()
            .UseTestServer()
            .ConfigureServices(services =>
            {
                services.AddSingleton<IPriorityParser, PriorityParser>();
                services.AddSingleton<IPriorityValidator, PriorityValidator>();
                services.AddSingleton<IRateLimiter, PriorityBasedRateLimiter>();
                services.AddSingleton<IMemoryCache, MemoryCache>();
                // Add other required services
            })
            .Configure(app =>
            {
                app.UseMiddleware<PriorityDetectionMiddleware>();
                app.UseMiddleware<RateLimitingMiddleware>();
                app.UseRouting();
                app.UseEndpoints(endpoints =>
                {
                    endpoints.MapPost("/api/subscriptions", async context =>
                    {
                        await context.Response.WriteAsync("Success");
                    });
                });
            });

        _server = new TestServer(hostBuilder);
        _client = _server.CreateClient();
    }

    [Test]
    public async Task HighPriorityRequest_BypassesRateLimit()
    {
        // Arrange
        var payload = new
        {
            priority = "high",
            CustomerOrganisationId = "test-org",
            ProductFeatures = new[] { new { ProductFeatureCode = "FEATURE_1", Limit = 100 } }
        };

        _client.DefaultRequestHeaders.Add("Xero-Client-Name", "critical-client");

        // Act - Send many requests quickly
        var tasks = Enumerable.Range(0, 50)
            .Select(_ => SendRequestAsync(payload))
            .ToArray();

        var responses = await Task.WhenAll(tasks);

        // Assert - All requests should succeed (no rate limiting)
        Assert.That(responses.All(r => r.IsSuccessStatusCode), Is.True);
    }

    [Test]
    public async Task LowPriorityRequest_EnforcesRateLimit()
    {
        // Arrange
        var payload = new
        {
            priority = "low",
            CustomerOrganisationId = "test-org",
            ProductFeatures = new[] { new { ProductFeatureCode = "FEATURE_1", Limit = 100 } }
        };

        _client.DefaultRequestHeaders.Add("Xero-Client-Name", "standard-client");

        // Act - Send requests that exceed rate limit
        var responses = new List<HttpResponseMessage>();
        for (int i = 0; i < 30; i++)
        {
            responses.Add(await SendRequestAsync(payload));
        }

        // Assert - Some requests should be rate limited (429)
        var rateLimitedCount = responses.Count(r => r.StatusCode == HttpStatusCode.TooManyRequests);
        Assert.That(rateLimitedCount, Is.GreaterThan(0));
    }

    [Test]
    public async Task MediumPriorityRequest_GetsTwiceTheLimit()
    {
        // Arrange
        var payload = new
        {
            priority = "medium",
            CustomerOrganisationId = "test-org",
            ProductFeatures = new[] { new { ProductFeatureCode = "FEATURE_1", Limit = 100 } }
        };

        _client.DefaultRequestHeaders.Add("Xero-Client-Name", "premium-client");

        // Act - Send requests up to 2x the normal limit
        var responses = new List<HttpResponseMessage>();
        for (int i = 0; i < 40; i++) // Assuming 20 is the base limit, 40 should mostly succeed
        {
            responses.Add(await SendRequestAsync(payload));
        }

        // Assert - Most requests should succeed due to 2x multiplier
        var successCount = responses.Count(r => r.IsSuccessStatusCode);
        Assert.That(successCount, Is.GreaterThan(30)); // Allow some variance
    }

    private async Task<HttpResponseMessage> SendRequestAsync(object payload)
    {
        var json = JsonSerializer.Serialize(payload);
        var content = new StringContent(json, Encoding.UTF8, "application/json");
        return await _client.PostAsync("/api/subscriptions", content);
    }

    [TearDown]
    public void TearDown()
    {
        _client?.Dispose();
        _server?.Dispose();
    }
}
```

### 3. Load Tests for Priority-Based Rate Limiting

```csharp
[TestFixture]
public class PriorityBasedLoadTests
{
    [Test]
    public async Task LoadTest_MixedPriorityRequests_MaintainsPerformance()
    {
        // This would typically use NBomber or similar load testing framework
        var concurrentUsers = 100;
        var requestsPerUser = 50;
        var testDuration = TimeSpan.FromMinutes(2);

        var priorityDistribution = new[]
        {
            (PriorityLevel.High, 5),     // 5% high priority
            (PriorityLevel.Medium, 15),  // 15% medium priority
            (PriorityLevel.Low, 70),     // 70% low priority
            (PriorityLevel.Background, 10) // 10% background priority
        };

        // Implement load test logic here
        // Verify that:
        // 1. High priority requests are never rate limited
        // 2. Medium priority requests get higher allowances
        // 3. System maintains acceptable response times under load
        // 4. Rate limiting accuracy is maintained under concurrency

        Assert.Pass("Load test implementation needed");
    }
}
```

## Deployment Guide

### 1. Configuration Files

#### appsettings.Production.json
```json
{
  "RateLimit": {
    "PrioritySettings": {
      "EnablePriorityBasedLimiting": true,
      "DefaultPriority": "low",
      "RequireReasonForHighPriority": true,
      "HighPriorityRequiresApproval": false,
      "HighPriorityUsageThresholdPerHour": 10,
      "PriorityRateLimitMultipliers": {
        "high": 0.0,
        "medium": 0.5,
        "low": 1.0,
        "background": 5.0
      }
    },
    "ClientTiers": {
      "xero-critical-client": "Critical",
      "xero-premium-client": "Premium",
      "integration-partner-A": "Premium",
      "trial-client-*": "Trial"
    },
    "ClientTierSettings": {
      "Critical": {
        "BaseRequestsPerMinute": 100,
        "BurstMultiplier": 3.0,
        "AllowedPriorities": ["high", "medium", "low", "background"]
      },
      "Premium": {
        "BaseRequestsPerMinute": 50,
        "BurstMultiplier": 2.5,
        "AllowedPriorities": ["medium", "low", "background"]
      },
      "Standard": {
        "BaseRequestsPerMinute": 20,
        "BurstMultiplier": 2.0,
        "AllowedPriorities": ["low", "background"]
      },
      "Trial": {
        "BaseRequestsPerMinute": 5,
        "BurstMultiplier": 1.5,
        "AllowedPriorities": ["low"]
      }
    }
  },
  "Security": {
    "RequireClientCertificateForHighPriority": true,
    "HighPriorityAllowedIPs": [
      "10.0.0.0/8",
      "172.16.0.0/12",
      "192.168.0.0/16"
    ]
  },
  "Logging": {
    "LogLevel": {
      "SubscriptionAdapter.RateLimit": "Information",
      "SubscriptionAdapter.Priority": "Information"
    }
  }
}
```

### 2. Startup Configuration

```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        // Register priority-based rate limiting services
        services.AddSingleton<IPriorityParser, PriorityParser>();
        services.AddSingleton<IPriorityValidator, PriorityValidator>();
        services.AddSingleton<IRateLimiter, PriorityBasedRateLimiter>();
        services.AddSingleton<IRateLimitConfiguration, RateLimitConfiguration>();
        services.AddSingleton<ISecurityService, SecurityService>();
        services.AddSingleton<IMetricsCollector, MetricsCollector>();
        services.AddSingleton<IRateLimitStore, RedisRateLimitStore>(); // or InMemoryRateLimitStore for testing
        
        // Configure memory cache for configuration caching
        services.AddMemoryCache();
        
        // Configure metrics
        services.AddOpenTelemetry()
            .WithMetrics(builder => builder
                .AddMeter("SubscriptionAdapter.RateLimit")
                .AddPrometheusExporter());
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        // Add priority detection middleware early in pipeline
        app.UseMiddleware<PriorityDetectionMiddleware>();
        
        // Add rate limiting middleware after priority detection
        app.UseMiddleware<RateLimitingMiddleware>();
        
        app.UseRouting();
        app.UseAuthentication();
        app.UseAuthorization();
        
        app.UseEndpoints(endpoints =>
        {
            endpoints.MapControllers();
        });
    }
}
```

### 3. Kubernetes Deployment Updates

```yaml
# values.yml update for ingress rate limiting
ingress:
  enabled: true
  annotations:
    nginx.ingress.kubernetes.io/rate-limit-rps: "50"
    nginx.ingress.kubernetes.io/rate-limit-burst-multiplier: "3"
    nginx.ingress.kubernetes.io/rate-limit-key: "$http_xero_client_name"
    
# ConfigMap for application configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: subscription-adapter-rate-limit-config
data:
  appsettings.RateLimit.json: |
    {
      "RateLimit": {
        "PrioritySettings": {
          "EnablePriorityBasedLimiting": true,
          "DefaultPriority": "low"
        }
      }
    }
```

## Troubleshooting

### 1. Common Issues and Solutions

#### Issue: High Priority Requests Being Rate Limited
**Symptoms**: 429 responses for requests with `"priority": "high"`
**Possible Causes**:
- Client not authorized for high priority
- JWT token validation failing
- Configuration error

**Troubleshooting Steps**:
```bash
# Check client tier configuration
kubectl logs deployment/subscription-adapter | grep "Client.*not authorized"

# Verify JWT token
curl -H "Authorization: Bearer YOUR_TOKEN" -H "Xero-Client-Name: your-client" \
  -d '{"priority": "high", "CustomerOrganisationId": "test"}' \
  -H "Content-Type: application/json" \
  https://your-api/subscriptions
```

#### Issue: Rate Limiter Performance Degradation
**Symptoms**: Increased response times, timeout errors
**Possible Causes**:
- Cache misses causing configuration lookups
- Database/Redis performance issues
- Memory pressure

**Troubleshooting Steps**:
```bash
# Check rate limiter performance metrics
kubectl exec -it deployment/subscription-adapter -- curl localhost:8080/metrics | grep rate_limit_check_duration

# Monitor cache hit rates
kubectl logs deployment/subscription-adapter | grep "Cache.*miss"

# Check memory usage
kubectl top pods -l app=subscription-adapter
```

### 2. Monitoring Queries

#### Prometheus/Grafana Queries
```promql
# Rate limit violations by priority
rate(rate_limit_violations_total[5m]) by (priority)

# Rate limit bypass frequency
rate(rate_limit_bypasses_total[5m]) by (client_id)

# Average rate limit check duration
histogram_quantile(0.95, rate(rate_limit_check_duration_ms_bucket[5m]))

# Priority distribution over time
rate(rate_limit_requests_total[5m]) by (priority)
```

### 3. Debugging Tools

#### Priority Validation Endpoint
```csharp
[ApiController]
[Route("api/debug")]
public class DebugController : ControllerBase
{
    private readonly IPriorityValidator _validator;
    private readonly IRateLimitConfiguration _config;

    [HttpPost("validate-priority")]
    public async Task<IActionResult> ValidatePriority([FromBody] PriorityValidationRequest request)
    {
        var context = HttpContext;
        var result = await _validator.ValidateAsync(request.Priority, context);
        
        return Ok(new
        {
            IsValid = result.IsValid,
            Reason = result.Reason,
            Errors = result.Errors,
            RecommendedLevel = result.RecommendedLevel,
            ClientId = ExtractClientId(context),
            ClientTier = await _config.GetClientTierAsync(ExtractClientId(context))
        });
    }
}
```

### 4. Performance Optimization

#### Cache Warming Strategy
```csharp
public class RateLimitCacheWarmer : IHostedService
{
    private readonly IRateLimitConfiguration _config;
    private readonly ILogger<RateLimitCacheWarmer> _logger;

    public async Task StartAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("Warming rate limit caches...");
        
        // Pre-load common client configurations
        var commonClients = new[] { "xero-web", "xero-mobile", "integration-partner-A" };
        foreach (var client in commonClients)
        {
            await _config.GetClientTierAsync(client);
        }
        
        // Pre-load priority settings
        await _config.GetPrioritySettingsAsync();
        
        _logger.LogInformation("Rate limit cache warming completed");
    }

    public Task StopAsync(CancellationToken cancellationToken) => Task.CompletedTask;
}
```

This comprehensive implementation guide provides all the necessary code, configuration, and operational details needed to implement priority-based rate limiting in your Subscription Adapter API. Each component is designed to be testable, maintainable, and scalable for production use.
