# .NET 10 Design Patterns Reference

## Table of Contents

1. [Creational Patterns](#creational-patterns)
2. [Structural Patterns](#structural-patterns)
3. [Behavioral Patterns](#behavioral-patterns)
4. [Cloud-Native Patterns](#cloud-native-patterns)

---

## Creational Patterns

### Singleton with DI

Prefer DI registration over manual singleton:

```csharp
// ✅ Preferred: Let DI handle lifetime
builder.Services.AddSingleton<ICacheService, MemoryCacheService>();

// ⚠️ Only when DI not available
public sealed class LegacySingleton
{
    private static readonly Lazy<LegacySingleton> _instance = new(() => new());
    public static LegacySingleton Instance => _instance.Value;
    private LegacySingleton() { }
}
```

### Builder Pattern

```csharp
public record HttpRequestConfig
{
    public required string Url { get; init; }
    public string Method { get; init; } = "GET";
    public Dictionary<string, string> Headers { get; init; } = [];
    public TimeSpan Timeout { get; init; } = TimeSpan.FromSeconds(30);
}

public class HttpRequestBuilder
{
    private string _url = "";
    private string _method = "GET";
    private readonly Dictionary<string, string> _headers = [];
    private TimeSpan _timeout = TimeSpan.FromSeconds(30);

    public HttpRequestBuilder WithUrl(string url) { _url = url; return this; }
    public HttpRequestBuilder WithMethod(string method) { _method = method; return this; }
    public HttpRequestBuilder WithHeader(string key, string value) { _headers[key] = value; return this; }
    public HttpRequestBuilder WithTimeout(TimeSpan timeout) { _timeout = timeout; return this; }
    
    public HttpRequestConfig Build() => new()
    {
        Url = _url,
        Method = _method,
        Headers = _headers,
        Timeout = _timeout
    };
}

// Usage
var config = new HttpRequestBuilder()
    .WithUrl("https://api.example.com")
    .WithMethod("POST")
    .WithHeader("Authorization", "Bearer token")
    .Build();
```

### Abstract Factory

```csharp
public interface IUIFactory
{
    IButton CreateButton();
    ITextBox CreateTextBox();
}

public class WindowsUIFactory : IUIFactory
{
    public IButton CreateButton() => new WindowsButton();
    public ITextBox CreateTextBox() => new WindowsTextBox();
}

public class MacUIFactory : IUIFactory
{
    public IButton CreateButton() => new MacButton();
    public ITextBox CreateTextBox() => new MacTextBox();
}

// Registration
builder.Services.AddSingleton<IUIFactory>(sp =>
    OperatingSystem.IsWindows() ? new WindowsUIFactory() : new MacUIFactory());
```

---

## Structural Patterns

### Decorator Pattern

```csharp
public interface INotificationService
{
    Task SendAsync(string message, CancellationToken ct = default);
}

public class EmailNotificationService : INotificationService
{
    public Task SendAsync(string message, CancellationToken ct = default)
        => Task.CompletedTask; // Send email
}

public class LoggingNotificationDecorator(
    INotificationService inner,
    ILogger<LoggingNotificationDecorator> logger) : INotificationService
{
    public async Task SendAsync(string message, CancellationToken ct = default)
    {
        logger.LogInformation("Sending notification: {Message}", message);
        await inner.SendAsync(message, ct);
        logger.LogInformation("Notification sent");
    }
}

// Registration with Scrutor
builder.Services.AddScoped<INotificationService, EmailNotificationService>();
builder.Services.Decorate<INotificationService, LoggingNotificationDecorator>();
```

### Adapter Pattern

```csharp
// Legacy interface
public interface ILegacyPayment
{
    void ProcessPayment(double amount, string currency);
}

// Modern interface
public interface IPaymentGateway
{
    Task<PaymentResult> ProcessAsync(PaymentRequest request, CancellationToken ct);
}

// Adapter
public class LegacyPaymentAdapter(ILegacyPayment legacy) : IPaymentGateway
{
    public Task<PaymentResult> ProcessAsync(PaymentRequest request, CancellationToken ct)
    {
        legacy.ProcessPayment((double)request.Amount, request.Currency);
        return Task.FromResult(new PaymentResult(true, "Success"));
    }
}
```

---

## Behavioral Patterns

### Strategy Pattern

```csharp
public interface IPricingStrategy
{
    decimal CalculatePrice(decimal basePrice, int quantity);
}

public class RegularPricing : IPricingStrategy
{
    public decimal CalculatePrice(decimal basePrice, int quantity) 
        => basePrice * quantity;
}

public class BulkPricing : IPricingStrategy
{
    public decimal CalculatePrice(decimal basePrice, int quantity) 
        => quantity >= 10 ? basePrice * quantity * 0.9m : basePrice * quantity;
}

public class PremiumPricing : IPricingStrategy
{
    public decimal CalculatePrice(decimal basePrice, int quantity) 
        => basePrice * quantity * 0.85m;
}

// Usage with pattern matching
public class PricingService
{
    public IPricingStrategy GetStrategy(CustomerType type) => type switch
    {
        CustomerType.Regular => new RegularPricing(),
        CustomerType.Bulk => new BulkPricing(),
        CustomerType.Premium => new PremiumPricing(),
        _ => throw new ArgumentOutOfRangeException(nameof(type))
    };
}
```

### Observer Pattern (Modern)

```csharp
public interface IEventPublisher
{
    Task PublishAsync<TEvent>(TEvent @event, CancellationToken ct = default) 
        where TEvent : class;
}

public interface IEventHandler<TEvent> where TEvent : class
{
    Task HandleAsync(TEvent @event, CancellationToken ct = default);
}

public class InMemoryEventPublisher(IServiceProvider sp) : IEventPublisher
{
    public async Task PublishAsync<TEvent>(TEvent @event, CancellationToken ct = default) 
        where TEvent : class
    {
        var handlers = sp.GetServices<IEventHandler<TEvent>>();
        await Parallel.ForEachAsync(handlers, ct, async (handler, token) =>
            await handler.HandleAsync(@event, token));
    }
}

// Registration
builder.Services.AddScoped<IEventPublisher, InMemoryEventPublisher>();
builder.Services.AddScoped<IEventHandler<OrderCreated>, SendEmailHandler>();
builder.Services.AddScoped<IEventHandler<OrderCreated>, UpdateInventoryHandler>();
```

### Mediator Pattern (MediatR Alternative)

```csharp
public interface IMediator
{
    Task<TResponse> SendAsync<TResponse>(IRequest<TResponse> request, CancellationToken ct = default);
}

public interface IRequest<TResponse> { }
public interface IRequestHandler<TRequest, TResponse> where TRequest : IRequest<TResponse>
{
    Task<TResponse> HandleAsync(TRequest request, CancellationToken ct = default);
}

public class Mediator(IServiceProvider sp) : IMediator
{
    public async Task<TResponse> SendAsync<TResponse>(
        IRequest<TResponse> request, 
        CancellationToken ct = default)
    {
        var handlerType = typeof(IRequestHandler<,>).MakeGenericType(request.GetType(), typeof(TResponse));
        var handler = sp.GetRequiredService(handlerType);
        var method = handlerType.GetMethod(nameof(IRequestHandler<IRequest<TResponse>, TResponse>.HandleAsync))!;
        return await (Task<TResponse>)method.Invoke(handler, [request, ct])!;
    }
}
```

---

## Cloud-Native Patterns

### Retry with Polly

```csharp
builder.Services.AddHttpClient<IApiClient, ApiClient>()
    .AddStandardResilienceHandler();

// Or custom policy
builder.Services.AddResiliencePipeline("default", builder =>
{
    builder
        .AddRetry(new RetryStrategyOptions
        {
            MaxRetryAttempts = 3,
            Delay = TimeSpan.FromMilliseconds(500),
            BackoffType = DelayBackoffType.Exponential
        })
        .AddTimeout(TimeSpan.FromSeconds(30));
});
```

### Circuit Breaker

```csharp
builder.Services.AddResiliencePipeline("circuit-breaker", builder =>
{
    builder.AddCircuitBreaker(new CircuitBreakerStrategyOptions
    {
        FailureRatio = 0.5,
        SamplingDuration = TimeSpan.FromSeconds(10),
        MinimumThroughput = 8,
        BreakDuration = TimeSpan.FromSeconds(30)
    });
});
```

### Health Checks

```csharp
builder.Services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>()
    .AddRedis(builder.Configuration.GetConnectionString("Redis")!)
    .AddCheck<CustomHealthCheck>("custom");

app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});
```

### Outbox Pattern (Eventual Consistency)

```csharp
public record OutboxMessage
{
    public Guid Id { get; init; } = Guid.NewGuid();
    public required string Type { get; init; }
    public required string Payload { get; init; }
    public DateTime CreatedAt { get; init; } = DateTime.UtcNow;
    public DateTime? ProcessedAt { get; set; }
}

public class OutboxProcessor(AppDbContext db, IEventPublisher publisher) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            var messages = await db.OutboxMessages
                .Where(m => m.ProcessedAt == null)
                .OrderBy(m => m.CreatedAt)
                .Take(10)
                .ToListAsync(ct);

            foreach (var message in messages)
            {
                // Deserialize and publish
                message.ProcessedAt = DateTime.UtcNow;
            }

            await db.SaveChangesAsync(ct);
            await Task.Delay(TimeSpan.FromSeconds(5), ct);
        }
    }
}
```
