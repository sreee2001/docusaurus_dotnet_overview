### Middleware

#### Short Introduction

Middleware in ASP.NET Core forms the request processing pipeline, where each component can process requests and responses or pass them to the next component.

#### Official Definition

Middleware is software that's assembled into an application pipeline to handle requests and responses. Each component chooses whether to pass the request to the next component in the pipeline and can perform work before and after the next component in the pipeline is invoked.

#### Usage

```csharp
// Built-in middleware
app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

// Custom middleware
public class RequestLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestLoggingMiddleware> _logger;

    public RequestLoggingMiddleware(RequestDelegate next, ILogger<RequestLoggingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var stopwatch = Stopwatch.StartNew();

        _logger.LogInformation("Request {Method} {Path} started",
            context.Request.Method, context.Request.Path);

        try
        {
            await _next(context);
        }
        finally
        {
            stopwatch.Stop();
            _logger.LogInformation("Request {Method} {Path} completed in {ElapsedMilliseconds}ms with status {StatusCode}",
                context.Request.Method,
                context.Request.Path,
                stopwatch.ElapsedMilliseconds,
                context.Response.StatusCode);
        }
    }
}

// Register middleware
app.UseMiddleware<RequestLoggingMiddleware>();
```

#### Use Cases

- Request/response logging
- Authentication and authorization
- Error handling
- Rate limiting
- Request/response modification
- Cross-cutting concerns

#### Sample Usage

```csharp
// Error handling middleware
public class GlobalExceptionMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<GlobalExceptionMiddleware> _logger;

    public GlobalExceptionMiddleware(RequestDelegate next, ILogger<GlobalExceptionMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "An unhandled exception occurred");
            await HandleExceptionAsync(context, ex);
        }
    }

    private static async Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        context.Response.ContentType = "application/json";

        var response = exception switch
        {
            NotFoundException => new { message = exception.Message, statusCode = 404 },
            ValidationException => new { message = exception.Message, statusCode = 400 },
            UnauthorizedException => new { message = "Unauthorized", statusCode = 401 },
            _ => new { message = "An error occurred", statusCode = 500 }
        };

        context.Response.StatusCode = response.statusCode;
        await context.Response.WriteAsync(JsonSerializer.Serialize(response));
    }
}
```

---
