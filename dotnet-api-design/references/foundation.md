# Foundation: envelope dan error handling

Baca hanya ketika membuat atau mengubah fondasi bersama. Pada proyek existing, adaptasi
helper yang sudah ada; jangan mengganti kontrak response hanya agar identik dengan contoh.
Setiap blok C# di bawah adalah file terpisah; menggunakan implicit usings dan nullable enabled.

## Common/Models/ApiResponse.cs

Clock adalah dependency wajib factory, bukan fallback statis. Bentuk JSON tetap
`success`, `code`, `message`, `data`, `errors`, dan `timestamp` dengan default web JSON options.

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
        T data, TimeProvider timeProvider, string message = "Permintaan berhasil diproses.") =>
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
        TimeProvider timeProvider, string message = "Permintaan berhasil diproses.") =>
        new() { Success = true, Code = "SUCCESS", Message = message,
            Timestamp = timeProvider.GetUtcNow() };

    public static new ApiResponse FailureResult(
        string code, string message, TimeProvider timeProvider, object? errors = null) =>
        new() { Success = false, Code = code, Message = message,
            Errors = errors, Timestamp = timeProvider.GetUtcNow() };
}
```

## Common/Models/PagedResult.cs

`TotalCount` berarti total item yang cocok dengan filter, bukan jumlah halaman.
Untuk data sangat besar, pertimbangkan cursor pagination dengan kontrak terpisah.

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
            [ErrorCode.InternalServerError] = (HttpStatusCode.InternalServerError, "Terjadi kesalahan internal pada server."),
            [ErrorCode.ValidationError] = (HttpStatusCode.BadRequest, "Data masukan tidak valid."),
            [ErrorCode.UnauthorizedAccess] = (HttpStatusCode.Unauthorized, "Autentikasi diperlukan."),
            [ErrorCode.ForbiddenAccess] = (HttpStatusCode.Forbidden, "Hak akses ditolak."),
            [ErrorCode.BadRequest] = (HttpStatusCode.BadRequest, "Format permintaan tidak valid."),
            [ErrorCode.OperationCancelled] = ((HttpStatusCode)499, "Permintaan dibatalkan oleh klien."),
            [ErrorCode.EntityNotFound] = (HttpStatusCode.NotFound, "Data tidak ditemukan."),
            [ErrorCode.EntityAlreadyExists] = (HttpStatusCode.Conflict, "Data sudah terdaftar."),
            [ErrorCode.InvalidBusinessRule] = (HttpStatusCode.BadRequest, "Aturan bisnis tidak terpenuhi.")
        }.ToFrozenDictionary();

    public static HttpStatusCode ToHttpStatusCode(this ErrorCode code) =>
        Map.TryGetValue(code, out var value) ? value.Status : HttpStatusCode.InternalServerError;

    public static string ToDefaultMessage(this ErrorCode code) =>
        Map.TryGetValue(code, out var value) ? value.Message : "Terjadi kesalahan internal pada server.";

    public static string ToErrorCodeString(this ErrorCode code) =>
        code == ErrorCode.None ? "SUCCESS" : $"ERR_{code.ToString().ToUpperInvariant()}";
}
```

## Common/Exceptions/AppException.cs

Hanya pesan domain yang aman untuk klien boleh dimasukkan ke exception ini.
Jangan membungkus semua kegagalan database/network menjadi `AppException`.

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

public class ValidationException(object errors, string message = "Validasi data gagal.")
    : AppException(ErrorCode.ValidationError, message, errors, HttpStatusCode.BadRequest);
```

## Common/Handlers/GlobalExceptionHandler.cs

`IExceptionHandler` didaftarkan sebagai singleton; jangan inject scoped DbContext/repository.
ASP.NET Core dapat menangani request abort sebelum memanggil handler; jangan mengandalkan
handler ini untuk menghitung semua disconnect atau mengirim envelope pada koneksi mati.

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

Cancellation tanpa `RequestAborted` tidak otomatis berarti 499. Jika aplikasi memiliki
timeout policy, petakan timeout yang teridentifikasi pada batas pemiliknya ke status yang
disepakati (misalnya 503/504 sesuai semantik). Error tak dikenal tetap 500 dan dicatat.
Validasi otomatis, authentication challenge, dan status non-exception membutuhkan
integrasi pipeline tersendiri; lihat pemilihan referensi di SKILL.md.
