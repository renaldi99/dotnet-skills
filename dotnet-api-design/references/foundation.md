# Foundation: envelope and error handling

Read only when creating or changing the shared foundation. In existing projects, adapt
the helpers already present; do not change the response contract just to match this example.
Each C# block below is a separate file; implicit usings and nullable are enabled.

## Common/Models/ApiResponse.cs

The clock is a required factory dependency, not a static fallback. The JSON shape stays
`success`, `code`, `message`, `data`, `errors`, and `timestamp` with the default web JSON options.

```csharp
namespace MyService.Api.Common.Models;

public class ApiResponse<T>
{
    public bool Success { get; init; }
    public string Code { get; init; } = string.Empty;
    public string Message { get; init; } = string.Empty;
    public T? Data { get; init; }
    public object? Errors { get; init; }
    public required DateTimeOffset Timestamp { get; init; }

    public static ApiResponse<T> SuccessResult(
        T data, TimeProvider timeProvider, string message = "Request processed successfully.") =>
        new() { Success = true, Code = "SUCCESS", Message = message,
            Data = data, Timestamp = timeProvider.GetUtcNow() };

    public static ApiResponse<T> FailureResult(
        string code, string message, TimeProvider timeProvider, object? errors = null) =>
        new() { Success = false, Code = code, Message = message,
            Errors = errors, Timestamp = timeProvider.GetUtcNow() };
}

public class ApiResponse : ApiResponse<object>
{
    public static ApiResponse SuccessResult(
        TimeProvider timeProvider, string message = "Request processed successfully.") =>
        new() { Success = true, Code = "SUCCESS", Message = message,
            Timestamp = timeProvider.GetUtcNow() };

    public static new ApiResponse FailureResult(
        string code, string message, TimeProvider timeProvider, object? errors = null) =>
        new() { Success = false, Code = code, Message = message,
            Errors = errors, Timestamp = timeProvider.GetUtcNow() };
}
```

## Common/Models/PagedResult.cs

`TotalCount` is the total number of items matching the filter, not the number of pages.
For very large data sets, consider cursor pagination with a separate contract.

```csharp
namespace MyService.Api.Common.Models;

public sealed record PagedResult<T>(
    IReadOnlyList<T> Items, int Page, int PageSize, int TotalCount);
```

## Common/Errors/ErrorCode.cs

```csharp
namespace MyService.Api.Common.Errors;

public enum ErrorCode
{
    None = 0,
    InternalServerError = 1000,
    ValidationError = 1001,
    UnauthorizedAccess = 1002,
    ForbiddenAccess = 1003,
    BadRequest = 1004,
    OperationCancelled = 1005,
    EntityNotFound = 2000,
    EntityAlreadyExists = 2001,
    InvalidBusinessRule = 2002
}
```

## Common/Errors/ErrorCodeExtensions.cs

```csharp
using System.Collections.Frozen;
using System.Net;

namespace MyService.Api.Common.Errors;

public static class ErrorCodeExtensions
{
    private static readonly FrozenDictionary<ErrorCode, (HttpStatusCode Status, string Message)> Map =
        new Dictionary<ErrorCode, (HttpStatusCode, string)>
        {
            [ErrorCode.None] = (HttpStatusCode.OK, "Success."),
            [ErrorCode.InternalServerError] = (HttpStatusCode.InternalServerError, "An internal server error occurred."),
            [ErrorCode.ValidationError] = (HttpStatusCode.BadRequest, "The input data is invalid."),
            [ErrorCode.UnauthorizedAccess] = (HttpStatusCode.Unauthorized, "Authentication is required."),
            [ErrorCode.ForbiddenAccess] = (HttpStatusCode.Forbidden, "Access denied."),
            [ErrorCode.BadRequest] = (HttpStatusCode.BadRequest, "The request format is invalid."),
            [ErrorCode.OperationCancelled] = ((HttpStatusCode)499, "The request was cancelled by the client."),
            [ErrorCode.EntityNotFound] = (HttpStatusCode.NotFound, "Data not found."),
            [ErrorCode.EntityAlreadyExists] = (HttpStatusCode.Conflict, "Data already exists."),
            [ErrorCode.InvalidBusinessRule] = (HttpStatusCode.BadRequest, "Business rule not satisfied.")
        }.ToFrozenDictionary();

    public static HttpStatusCode ToHttpStatusCode(this ErrorCode code) =>
        Map.TryGetValue(code, out var value) ? value.Status : HttpStatusCode.InternalServerError;

    public static string ToDefaultMessage(this ErrorCode code) =>
        Map.TryGetValue(code, out var value) ? value.Message : "An internal server error occurred.";

    public static string ToErrorCodeString(this ErrorCode code) =>
        code == ErrorCode.None ? "SUCCESS" : $"ERR_{code.ToString().ToUpperInvariant()}";
}
```

## Common/Exceptions/AppException.cs

Only domain messages that are safe for clients may be put into this exception.
Do not wrap every database/network failure in an `AppException`.

```csharp
using System.Net;
using MyService.Api.Common.Errors;

namespace MyService.Api.Common.Exceptions;

public class AppException(ErrorCode errorCode, string? message = null, object? errors = null,
    HttpStatusCode? statusCode = null) : Exception(message ?? errorCode.ToDefaultMessage())
{
    public ErrorCode ErrorCode { get; } = errorCode;
    public HttpStatusCode StatusCode { get; } = statusCode ?? errorCode.ToHttpStatusCode();
    public object? Errors { get; } = errors;
}

public class NotFoundException(ErrorCode errorCode, string? message = null)
    : AppException(errorCode, message, statusCode: HttpStatusCode.NotFound);

public class ConflictException(ErrorCode errorCode, string? message = null)
    : AppException(errorCode, message, statusCode: HttpStatusCode.Conflict);

public class ValidationException(object errors, string message = "Data validation failed.")
    : AppException(ErrorCode.ValidationError, message, errors, HttpStatusCode.BadRequest);
```

## Common/Handlers/GlobalExceptionHandler.cs

`IExceptionHandler` is registered as a singleton; do not inject a scoped DbContext/repository.
ASP.NET Core may handle a request abort before calling the handler; do not rely on
this handler to count every disconnect or to send an envelope over a dead connection.

```csharp
using Microsoft.AspNetCore.Diagnostics;
using MyService.Api.Common.Errors;
using MyService.Api.Common.Exceptions;
using MyService.Api.Common.Models;

namespace MyService.Api.Common.Handlers;

public sealed class GlobalExceptionHandler(
    ILogger<GlobalExceptionHandler> logger, TimeProvider timeProvider) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext context, Exception exception, CancellationToken cancellationToken)
    {
        if (exception is OperationCanceledException && context.RequestAborted.IsCancellationRequested)
        {
            logger.LogDebug("Client disconnected: {Path}", context.Request.Path);
            if (!context.Response.HasStarted)
                context.Response.StatusCode = 499;
            return true;
        }

        if (context.Response.HasStarted)
            return false;

        var error = exception as AppException;
        var status = error is null ? 500 : (int)error.StatusCode;
        var code = error?.ErrorCode ?? ErrorCode.InternalServerError;
        var message = error?.Message ?? code.ToDefaultMessage();

        if (status >= 500)
            logger.LogError(exception, "Request failed: {Path}", context.Request.Path);
        else
            logger.LogWarning("Domain error {Code}: {Path}", code, context.Request.Path);

        context.Response.StatusCode = status;
        await context.Response.WriteAsJsonAsync(
            ApiResponse.FailureResult(code.ToErrorCodeString(), message, timeProvider, error?.Errors),
            cancellationToken);
        return true;
    }
}
```

Cancellation without `RequestAborted` does not automatically mean 499. If the application has
a timeout policy, map identified timeouts at their owning boundary to the agreed status
(for example 503/504 depending on semantics). Unknown errors stay 500 and are logged.
Automatic validation, authentication challenges, and non-exception statuses need their
own pipeline integration; see the reference selection in SKILL.md.
