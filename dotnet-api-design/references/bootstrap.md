# Bootstrap: DI dan HTTP pipeline

Baca untuk aplikasi baru atau perubahan pipeline. Contoh ini **baseline tanpa autentikasi**,
bukan konfigurasi production yang otomatis aman. Sebelum deployment, tentukan endpoint
public/protected dan konfigurasi auth sesuai kebutuhan pengguna. Jangan menimpa startup existing.

Prasyarat: SDK Web .NET 10, nullable/implicit usings enabled, file Foundation dan Module
scaffolding. Paket contoh: `Microsoft.EntityFrameworkCore.SqlServer` dan
`Microsoft.AspNetCore.OpenApi` yang kompatibel dengan .NET 10. Ikuti versi terpusat proyek;
jangan menambahkan Scalar, health-check database, atau library validasi jika tidak diperlukan.

## Program.cs

Invalid model/JSON dipetakan melalui `InvalidModelStateResponseFactory`, bukan exception handler.
`SuppressMapClientErrors` mencegah MVC mengubah status error kosong menjadi ProblemDetails.
Status pages mengisi error kosong hanya pada prefix API; body yang sudah ada tidak ditimpa.

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
    ?? throw new InvalidOperationException("ConnectionStrings:DefaultConnection wajib diisi.");

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
            .ToDictionary(entry => entry.Key, _ => new[] { "Nilai tidak valid." });
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
        ApiResponse.FailureResult($"HTTP_{status}", "Permintaan tidak dapat diproses.", clock),
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

## Integrasi yang bergantung pada proyek

| Kebutuhan | Integrasi wajib bila dipakai |
| :--- | :--- |
| Authentication/authorization | Daftarkan `AddAuthentication(...).Add...(...)` dan `AddAuthorization(...)` dengan scheme/policy nyata. Pasang `UseAuthentication()` lalu `UseAuthorization()` setelah routing dan sebelum endpoint. Terapkan policy/`[Authorize]`/fallback policy yang disepakati; registrasi saja tidak melindungi endpoint. |
| 401/403 dari auth handler | Pertahankan header seperti `WWW-Authenticate`. Status pages hanya mengisi body kosong; gunakan events/result handler yang sesuai scheme bila body ditulis sendiri. Jangan mengganti respons menjadi 200 atau redirect login untuk kontrak API. |
| FluentValidation | Register validator dan panggil validasi di boundary yang disepakati. Validator async harus dijalankan dengan `ValidateAsync` dan token request, bukan MVC validation pipeline sinkron. Jangan mendaftarkan validator tanpa pernah memanggilnya. |
| Readiness database | Tambahkan provider health check dan paketnya; baseline `/health` hanya liveness, bukan bukti database siap. Batasi exposure detail operasional. |
| Reverse proxy | Ikuti konfigurasi forwarded headers/trusted proxies proyek sebelum middleware yang bergantung pada scheme/IP; jangan percaya header dari semua sumber. |

## Kontrak yang harus diperiksa

- Success, domain failure, invalid model/JSON memakai envelope dan status yang sesuai.
- Error kosong 404/405/415 pada `/api` memakai `HTTP_{status}`; error domain memakai `ERR_...`.
  Jika proyek memakai kode lain, pertahankan kode existing dan sesuaikan pemetaan.
- HEAD/204/304 tidak mendapat body; health/OpenAPI/file/stream bukan envelope aplikasi.
- Response yang sudah dimulai tidak ditulis ulang. Client disconnect tidak menjanjikan body.
- Uji host startup dan HTTP pipeline dengan `WebApplicationFactory`/integration test proyek,
  bukan hanya unit test Controller yang tidak menjalankan filter maupun middleware.
