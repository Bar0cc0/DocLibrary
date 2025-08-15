# C# Logging Best Practices Cheatsheet

## Logging Frameworks

| Framework | Features | Best For |
|-----------|----------|----------|
| **Serilog** | Structured logging, sinks, enrichers | Modern apps, cloud-native |
| **NLog** | Flexible routing, high performance | Complex logging requirements |
| **log4net** | Mature, extensible | Legacy applications |
| **Microsoft.Extensions.Logging** | Built-in ASP.NET Core integration | ASP.NET Core applications |

## Log Levels

- **Trace**: Detailed debugging information (disabled in production)
- **Debug**: Development-time information (disabled in production)
- **Information**: Notable but normal application events
- **Warning**: Non-critical issues, degraded functionality
- **Error**: Failures requiring attention, application can continue
- **Critical**: Critical failures requiring immediate attention

## Structured Logging

```csharp
// ❌ Avoid string concatenation
logger.LogInformation("User " + userName + " logged in at " + loginTime);

// ✅ Use templates with named parameters
logger.LogInformation("User {UserName} logged in at {LoginTime}", userName, loginTime);
```

## Configuration Best Practices

```csharp
// Program.cs in ASP.NET Core
builder.Host.UseSerilog((context, configuration) => 
    configuration
        .ReadFrom.Configuration(context.Configuration)
        .Enrich.FromLogContext()
        .WriteTo.Console()
        .WriteTo.File("logs/app.log", rollingInterval: RollingInterval.Day));
```

## appsettings.json Example

```json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "System": "Warning"
      }
    }
  }
}
```

## Dependency Injection

```csharp
public class UserService
{
    private readonly ILogger<UserService> _logger;

    public UserService(ILogger<UserService> logger)
    {
        _logger = logger;
    }
    
    public async Task<User> GetUserAsync(int id)
    {
        _logger.LogDebug("Retrieving user with ID {UserId}", id);
        // Implementation
    }
}
```

## Context Enrichment

```csharp
// Adding context properties
using (LogContext.PushProperty("TransactionId", transactionId))
using (LogContext.PushProperty("UserId", userId))
{
    _logger.LogInformation("Processing payment");
    // Operation code
}
```

## Exception Handling

```csharp
try
{
    // Operation that might fail
}
catch (Exception ex)
{
    // ❌ Avoid: loses stack trace and context
    _logger.LogError("Error occurred: " + ex.Message);
    
    // ✅ Better: includes full exception details
    _logger.LogError(ex, "Error processing payment for {OrderId}", orderId);
}
```

## Performance Considerations

```csharp
// ❌ Avoid: Always evaluates expensive operation
_logger.LogDebug("User details: " + GetUserDetails(user));

// ✅ Better: Only evaluates if debug is enabled
_logger.LogDebug("User details: {UserDetails}", () => GetUserDetails(user));

// ✅ Or check level first
if (_logger.IsEnabled(LogLevel.Debug))
{
    _logger.LogDebug("User details: {UserDetails}", GetUserDetails(user));
}
```

## Security Best Practices

### Data Protection
- **PII Classification**: Create a classification system for your data (public, internal, sensitive, highly-sensitive)
- **Data Masking Example**:
  ```csharp
  // ❌ Avoid
  _logger.LogInformation("Credit card {CardNumber} processed", cardNumber);
  
  // ✅ Better
  _logger.LogInformation("Credit card {CardNumber} processed", MaskCardNumber(cardNumber));
  
  private string MaskCardNumber(string cardNumber) => 
      cardNumber.Length > 4 ? $"****-****-****-{cardNumber.Substring(cardNumber.Length - 4)}" : "****";
  ```

### Compliance & Governance
- **Implement Log Sanitization Middleware**:
  ```csharp
  public class LogSanitizationMiddleware
  {
      private readonly RequestDelegate _next;
      private static readonly Regex _sensitiveDataRegex = new Regex(@"(password|ssn|creditcard)[:=]\s*[^&\s]+", RegexOptions.IgnoreCase);
      
      public LogSanitizationMiddleware(RequestDelegate next) => _next = next;
      
      public async Task InvokeAsync(HttpContext context)
      {
          // Create a copy of the request body that can be read multiple times
          context.Request.EnableBuffering();
          
          // Read and sanitize the request for logging
          using var reader = new StreamReader(context.Request.Body, leaveOpen: true);
          var body = await reader.ReadToEndAsync();
          var sanitizedBody = _sensitiveDataRegex.Replace(body, "$1: [REDACTED]");
          
          // Log the sanitized version
          context.Items["SanitizedRequestBody"] = sanitizedBody;
          
          // Reset position for the next middleware
          context.Request.Body.Position = 0;
          await _next(context);
      }
  }
  ```

### Secure Storage
- **Encryption at Rest**: Enable encryption for log files
  ```csharp
  // Example using Serilog with encrypted file sink
  Log.Logger = new LoggerConfiguration()
      .WriteTo.File(new EncryptingJsonFormatter(encryptionKey), "logs/secure.json")
      .CreateLogger();
  ```
- **Set proper file permissions**: Ensure log files have minimal necessary permissions
- **Use centralized logging**: Ship logs to secure SIEM systems instead of storing locally

## Log Correlation
Log correlation is the practice of connecting related log entries across different components, services, and time periods using unique identifiers. This creates a complete picture of transactions as they flow through a distributed system.

### Generating Correlation IDs
```csharp
// ASP.NET Core middleware to add correlation ID
app.Use(async (context, next) =>
{
    var correlationId = context.Request.Headers["X-Correlation-ID"].FirstOrDefault()
        ?? Guid.NewGuid().ToString();
        
    using (LogContext.PushProperty("CorrelationId", correlationId))
    {
        context.Response.Headers["X-Correlation-ID"] = correlationId;
        await next();
    }
});

```

### Middleware Approach

```csharp
public class CorrelationMiddleware
{
    private readonly RequestDelegate _next;
    private const string CorrelationHeaderName = "X-Correlation-ID";
    
    public CorrelationMiddleware(RequestDelegate next)
    {
        _next = next;
    }
    
    public async Task InvokeAsync(HttpContext context, ILogger<CorrelationMiddleware> logger)
    {
        // Extract or generate correlation ID
        string correlationId = context.Request.Headers[CorrelationHeaderName].FirstOrDefault() 
            ?? Guid.NewGuid().ToString();
            
        // Add to response headers
        context.Response.Headers[CorrelationHeaderName] = correlationId;
        
        // Add to LogContext for all logs in this request
        using (LogContext.PushProperty("CorrelationId", correlationId))
        {
            logger.LogInformation("Request {Method} {Path} started with {CorrelationId}", 
                context.Request.Method, context.Request.Path, correlationId);
                
            await _next(context);
            
            logger.LogInformation("Request {Method} {Path} completed", 
                context.Request.Method, context.Request.Path);
        }
    }
}

// Register in Program.cs
app.UseMiddleware<CorrelationMiddleware>();
```

### Propagating Correlation ID to Downstream Services

```csharp
public class CorrelatedHttpClient
{
    private readonly HttpClient _httpClient;
    private readonly IHttpContextAccessor _httpContextAccessor;
    private const string CorrelationHeaderName = "X-Correlation-ID";
    
    public CorrelatedHttpClient(HttpClient httpClient, IHttpContextAccessor httpContextAccessor)
    {
        _httpClient = httpClient;
        _httpContextAccessor = httpContextAccessor;
    }
    
    public async Task<HttpResponseMessage> SendAsync(HttpRequestMessage request)
    {
        // Get correlation ID from current context
        string correlationId = _httpContextAccessor.HttpContext?.Request.Headers[CorrelationHeaderName]
            ?? LogContext.PushProperty("CorrelationId")?.ToString()
            ?? Guid.NewGuid().ToString();
            
        // Add to outgoing request
        request.Headers.Add(CorrelationHeaderName, correlationId);
        
        return await _httpClient.SendAsync(request);
    }
}
```

### Background Job Correlation

```csharp
public class CorrelatedBackgroundJob
{
    private readonly ILogger<CorrelatedBackgroundJob> _logger;
    
    public CorrelatedBackgroundJob(ILogger<CorrelatedBackgroundJob> logger)
    {
        _logger = logger;
    }
    
    public async Task ExecuteAsync(JobParameters parameters)
    {
        // Extract correlation ID from job parameters
        string correlationId = parameters.CorrelationId ?? Guid.NewGuid().ToString();
        
        using (LogContext.PushProperty("CorrelationId", correlationId))
        using (LogContext.PushProperty("JobId", parameters.JobId))
        {
            _logger.LogInformation("Starting background job {JobId}", parameters.JobId);
            // Job execution
            _logger.LogInformation("Completed background job {JobId}", parameters.JobId);
        }
    }
}
```

### Integration with OpenTelemetry

```csharp
// Program.cs
builder.Services.AddOpenTelemetry()
    .WithTracing(builder => builder
        .AddSource("MyApplication")
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddOtlpExporter());
        
// Connect logs to traces
builder.Host.UseSerilog((context, services, configuration) => configuration
    .ReadFrom.Configuration(context.Configuration)
    .ReadFrom.Services(services)
    .Enrich.FromLogContext()
    // This adds trace context (trace ID, span ID) to all logs
    .Enrich.WithSpan());
```


### Correlation in Non-HTTP Contexts

```csharp
// MassTransit (message bus) example
public class OrderProcessor : IConsumer<SubmitOrder>
{
    private readonly ILogger<OrderProcessor> _logger;
    
    public OrderProcessor(ILogger<OrderProcessor> logger)
    {
        _logger = logger;
    }
    
    public async Task Consume(ConsumeContext<SubmitOrder> context)
    {
        // Extract correlation ID from message headers
        var correlationId = context.Headers.Get<string>("X-Correlation-ID");
        
        using (LogContext.PushProperty("CorrelationId", correlationId))
        using (LogContext.PushProperty("OrderId", context.Message.OrderId))
        {
            _logger.LogInformation("Processing order {OrderId}", context.Message.OrderId);
            // Processing logic
        }
    }
}
```

## Common Anti-Patterns
### Inconsistent Logging Levels
- **Anti-pattern**: Using incorrect severity levels
  ```csharp
  // ❌ Inappropriate level - this is not an error
  _logger.LogError("User preferences updated successfully");
  
  // ✅ Correct level
  _logger.LogInformation("User preferences updated successfully");
  ```

### Log Pollution
- **Anti-pattern**: Excessive logging in high-throughput code paths
  ```csharp
  // ❌ Logging in a tight loop
  foreach (var item in millionsOfItems)
  {
      _logger.LogDebug("Processing item {ItemId}", item.Id); // Will generate millions of logs!
  }
  
  // ✅ Better approach
  _logger.LogDebug("Starting batch processing of {Count} items", millionsOfItems.Count);
  var processedCount = 0;
  foreach (var item in millionsOfItems)
  {
      // Process without logging each item
      processedCount++;
      if (processedCount % 10000 == 0)
      {
          _logger.LogDebug("Processed {Count}/{Total} items", processedCount, millionsOfItems.Count);
      }
  }
  _logger.LogDebug("Completed processing {Count} items", processedCount);
  ```

### Exception Swallowing
- **Anti-pattern**: Catching exceptions without proper logging or re-throwing
  ```csharp
  // ❌ Exception swallowing
  try 
  {
      DoSomethingRisky();
  }
  catch (Exception) 
  {
      // Silent failure!
  }
  
  // ✅ Better approach
  try 
  {
      DoSomethingRisky();
  }
  catch (Exception ex) when (LogError(ex))
  {
      throw; // Re-throw or handle appropriately
  }
  
  private bool LogError(Exception ex)
  {
      _logger.LogError(ex, "Error in risky operation");
      return false; // Continue to the throw statement
  }
  ```
### The `catch (Exception ex) when (LogError(ex))` Pattern

Advantage = Guaranteed Logging and Clean Exception Propagation

```csharp
// ❌ Potential issues with this approach
try 
{
    DoSomethingRisky();
}
catch (Exception ex) 
{
    _logger.LogError(ex, "Error occurred"); // If logging fails, original exception is lost
    throw; // This might never execute if logging throws
}
```
```csharp
// ✅ More robust approach
try 
{
    DoSomethingRisky();
}
catch (Exception ex) when (LogError(ex)) // LogError returns false = filter condition fails so catch block is executed
{
    throw; // Original exception is preserved
}
```



## Monitoring Integration
### OpenTelemetry Integration
```csharp
// Program.cs
builder.Services.AddOpenTelemetry()
    .WithTracing(tracerProviderBuilder =>
        tracerProviderBuilder
            .AddSource("MyApplication")
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddOtlpExporter())
    .WithMetrics(metricsProviderBuilder =>
        metricsProviderBuilder
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddOtlpExporter());

// Correlation between logs and traces
builder.Host.UseSerilog((context, services, configuration) => configuration
    .ReadFrom.Configuration(context.Configuration)
    .ReadFrom.Services(services)
    .Enrich.FromLogContext()
    .WriteTo.Console(
        outputTemplate: "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}")
    .WriteTo.OpenTelemetry());
```

### Alert Configuration
- **Set up smart alerting rules**:
  - Alert on error rate increases, not individual errors
  - Use dynamic thresholds based on historical patterns
  - Implement alert grouping to prevent alert storms

```csharp
// Example: Custom health check that monitors error logs
public class LogErrorRateHealthCheck : IHealthCheck
{
    private readonly ILogAnalyzer _logAnalyzer;
    
    public LogErrorRateHealthCheck(ILogAnalyzer logAnalyzer)
    {
        _logAnalyzer = logAnalyzer;
    }
    
    public async Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, CancellationToken cancellationToken = default)
    {
        var errorRate = await _logAnalyzer.GetErrorRateAsync(TimeSpan.FromMinutes(5));
        
        return errorRate < 0.05 
            ? HealthCheckResult.Healthy($"Error rate: {errorRate:P2}")
            : HealthCheckResult.Unhealthy($"Error rate too high: {errorRate:P2}");
    }
}
```

### Log Aggregation Best Practices
- **Centralized dashboard setup**:
  - Create dashboards for each microservice
  - Set up service maps to visualize dependencies
  - Define custom metrics derived from logs
- **Custom log processing**:
  ```csharp
  // Extracting business metrics from logs
  services.AddSingleton<ILogProcessor, LogProcessor>();
  services.AddHostedService<LogProcessingService>();
  
  public class LogProcessingService : BackgroundService
  {
      private readonly ILogProcessor _processor;
      
      public LogProcessingService(ILogProcessor processor)
      {
          _processor = processor;
      }
      
      protected override async Task ExecuteAsync(CancellationToken stoppingToken)
      {
          await _processor.ProcessLogStreamAsync(stoppingToken);
      }
  }
  ```


## Testing Logging

```csharp
[Fact]
public void LogsCorrectInformation_WhenUserCreated()
{
    // Arrange
    var loggerMock = new Mock<ILogger<UserService>>();
    var service = new UserService(loggerMock.Object);
    
    // Act
    service.CreateUser(new User { Name = "Test" });
    
    // Assert
    loggerMock.Verify(
        x => x.Log(
            LogLevel.Information,
            It.IsAny<EventId>(),
            It.Is<It.IsAnyType>((v, t) => v.ToString().Contains("Created user Test")),
            null,
            It.IsAny<Func<It.IsAnyType, Exception, string>>()),
        Times.Once);
}
```

