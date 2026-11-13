# Bootstrap: DI and HTTP pipeline

Read for new applications or pipeline changes. This example is an **unauthenticated baseline**,
not a production configuration that is secure by default. Before deployment, decide which endpoints
are public/protected and configure auth according to the user's needs. Do not overwrite existing startup code.

Prerequisites: .NET 10 Web SDK, nullable/implicit usings enabled, and the Foundation and Module
scaffolding files. Example packages: `Microsoft.EntityFrameworkCore.SqlServer` and
`Microsoft.AspNetCore.OpenApi` compatible with .NET 10. Follow the project's central package versions;
do not add Scalar, database health checks, or validation libraries unless needed.

## Program.cs

Invalid model/JSON is mapped through `InvalidModelStateResponseFactory`, not the exception handler.
`SuppressMapClientErrors` stops MVC from turning empty error statuses into ProblemDetails.
Status pages fill empty errors only under the API prefix; existing bodies are not overwritten.

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using MyService.Api.Common.Errors;
using MyService.Api.Common.Handlers;
using MyService.Api.Common.Models;
using MyService.Api.Data;
using MyService.Api.Modules.Employees;

var builder = WebApplication.CreateBuilder(args);
var connectionString = builder.Configuration.GetConnectionString("DefaultConnection")
    ?? throw new InvalidOperationException("ConnectionStrings:DefaultConnection is required.");

builder.Services.AddDbContext<AppDbContext>(options => options.UseSqlServer(connectionString));
builder.Services.AddSingleton(TimeProvider.System);
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();
builder.Services.AddControllers().ConfigureApiBehaviorOptions(options =>
{
    options.SuppressMapClientErrors = true;
    options.InvalidModelStateResponseFactory = context =>
    {
        var clock = context.HttpContext.RequestServices.GetRequiredService<TimeProvider>();
        var errors = context.ModelState
            .Where(entry => entry.Value is { Errors.Count: > 0 })
            .ToDictionary(entry => entry.Key, _ => new[] { "Invalid value." });
        var code = ErrorCode.ValidationError;
        return new BadRequestObjectResult(
            ApiResponse.FailureResult(code.ToErrorCodeString(), code.ToDefaultMessage(), clock, errors));
    };
});
builder.Services.AddEmployeesModule();
builder.Services.AddOpenApi("v1");
builder.Services.AddHealthChecks();

var app = builder.Build();
app.UseExceptionHandler();
app.UseStatusCodePages(async statusContext =>
{
    var context = statusContext.HttpContext;
    if (!context.Request.Path.StartsWithSegments("/api") || HttpMethods.IsHead(context.Request.Method)
        || context.RequestAborted.IsCancellationRequested)
        return;

    var status = context.Response.StatusCode;
    var clock = context.RequestServices.GetRequiredService<TimeProvider>();
    await context.Response.WriteAsJsonAsync(
        ApiResponse.FailureResult($"HTTP_{status}", "The request could not be processed.", clock),
        context.RequestAborted);
});
app.UseHttpsRedirection();
app.UseRouting();

if (app.Environment.IsDevelopment())
    app.MapOpenApi();

app.MapControllers();
app.MapHealthChecks("/health");
app.Run();
```

## Project-dependent integrations

| Need | Required integration when used |
| :--- | :--- |
| Authentication/authorization | Register `AddAuthentication(...).Add...(...)` and `AddAuthorization(...)` with real schemes/policies. Add `UseAuthentication()` then `UseAuthorization()` after routing and before endpoints. Apply the agreed policy/`[Authorize]`/fallback policy; registration alone does not protect endpoints. |
| 401/403 from the auth handler | Keep headers such as `WWW-Authenticate`. Status pages only fill empty bodies; use scheme-specific events/result handlers if you write the body yourself. Do not turn responses into 200 or a login redirect for API contracts. |
| FluentValidation | Register validators and call validation at the agreed boundary. Async validators must run with `ValidateAsync` and the request token, not the synchronous MVC validation pipeline. Do not register validators that are never called. |
| Database readiness | Add a provider health check and its package; the baseline `/health` is liveness only, not proof the database is ready. Limit exposure of operational details. |
| Reverse proxy | Follow the project's forwarded headers/trusted proxies configuration before middleware that depends on scheme/IP; do not trust headers from every source. |

## Contract checks

- Success, domain failure, and invalid model/JSON use the envelope and the appropriate status.
- Empty 404/405/415 errors under `/api` use `HTTP_{status}`; domain errors use `ERR_...`.
  If the project uses other codes, keep the existing codes and adjust the mapping.
- HEAD/204/304 get no body; health/OpenAPI/file/stream are not application envelopes.
- Responses that have already started are not rewritten. A client disconnect does not promise a body.
- Test host startup and the HTTP pipeline with `WebApplicationFactory`/the project's integration tests,
  not only Controller unit tests that run neither filters nor middleware.
