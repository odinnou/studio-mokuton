---
title: "Building Caresse #3: Integration Testing a .NET API Without Touching Production"
subtitle: "Testcontainers for PostgreSQL, Azurite for Azure Blob Storage, a custom auth handler to replace Firebase, and WireMock.Net for RevenueCat, LLM providers and TTS — all wired into one xUnit suite that runs in CI with zero external dependencies."
tags: [dotnet, testing, testcontainers, xunit, aspnetcore, building-caresse]
series: "Building Caresse"
series_index: 3
reading_time: "12 min"
language: en
medium:
  member_only: false  # PUBLIC — high shareability, strong SEO, funnel for #4+
  publication_target: "Better Programming"
canonical: https://caresse.app
---

# Building Caresse #3: Integration Testing a .NET API Without Touching Production

> *Third article in **Building Caresse** — inside the stack of a shipped, bootstrapped AI product. Previous: [#1 hexagonal architecture](https://caresse.app), [#2 multi-phase LLM pipeline](https://caresse.app).*

The [Caresse](https://caresse.app) API depends on several external systems: PostgreSQL, Azure Blob Storage, Firebase (auth), RevenueCat (subscriptions), multiple LLM providers, and multiple TTS providers. In our integration test suite, none of them are the real thing.

- PostgreSQL runs in a Docker container spun up by Testcontainers
- Azure Blob Storage is replaced by Azurite, Microsoft's local emulator
- Firebase JWT validation is bypassed by a custom `TestAuthHandler`
- RevenueCat, LLM providers, and TTS providers are all stubbed by WireMock.Net

The suite runs in CI, takes under 90 seconds, and has never required a production credential. This article shows the full setup — every piece, wired together.

<!--more-->

---

## Why integration tests, not unit tests

This is worth addressing upfront, because the instinct on a project like this is to mock everything and keep tests fast. We did that at first. It bit us twice.

The first time: a PostgreSQL migration that passed all unit tests and crashed on apply against the real schema. The migration had a column rename; the NSubstitute mock didn't care about column names. The real database did.

The second time: a change to how we serialized the RevenueCat webhook payload. The mock returned whatever we told it to — so the tests passed. The production handler silently dropped webhooks for two days before we caught it in the logs.

The failure mode with mocks isn't that they're wrong — it's that they're *too agreeable*. They return exactly what you tell them to, which means your tests only verify what you already assumed. Real HTTP contracts, real schema migrations, real serialization: these are the things that actually break in production, and mocks can't surface them.

The setup in this article is more complex than a pure unit test suite. It's also the reason we haven't had a production incident from a broken integration in over six months.

---

## Project structure

```
tests/
└── Caresse.Api.IntegrationTests/
    ├── Fixtures/
    │   ├── ApiFactory.cs          # WebApplicationFactory + container wiring
    │   ├── PostgresFixture.cs     # Testcontainers PostgreSQL
    │   └── WireMockFixture.cs     # WireMock server lifecycle
    ├── Helpers/
    │   └── AuthHelper.cs          # JWT generation for TestAuthHandler
    ├── Stubs/
    │   ├── OpenAiStubs.cs
    │   ├── ElevenLabsStubs.cs
    │   └── RevenueCatStubs.cs
    └── Tests/
        ├── GenerationTests.cs
        ├── SubscriptionTests.cs
        └── StorageTests.cs
```

Everything flows through `ApiFactory` — one entry point, all dependencies wired in one place. Adding a new external dependency means touching exactly one file.

---

## Dependencies

```xml
<PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" Version="9.*" />
<PackageReference Include="Testcontainers.PostgreSql"        Version="3.*" />
<PackageReference Include="WireMock.Net"                     Version="1.*" />
<PackageReference Include="xunit"                            Version="2.*" />
<PackageReference Include="xunit.runner.visualstudio"        Version="2.*" />
<PackageReference Include="FluentAssertions"                 Version="6.*" />
```

Azurite ships as a Docker image — no NuGet package needed beyond `Azure.Storage.Blobs` which is already in the main project.

---

## 1. Testcontainers — real PostgreSQL in tests

The standard approach for PostgreSQL in unit tests is to mock `IRepository` or inject an in-memory provider. Both approaches hide the thing most likely to break: the database schema, the migrations, and the actual SQL queries. Testcontainers fixes this by spinning up a real PostgreSQL 16 container on test startup, running migrations against it, and tearing it down after.

```csharp
// Fixtures/PostgresFixture.cs
public sealed class PostgresFixture : IAsyncLifetime
{
    private readonly PostgreSqlContainer _container = new PostgreSqlBuilder()
        .WithImage("postgres:16-alpine")
        .WithDatabase("caresse_test")
        .WithUsername("test")
        .WithPassword("test")
        .WithCleanUp(true)
        .Build();

    public string ConnectionString => _container.GetConnectionString();

    public async Task InitializeAsync() => await _container.StartAsync();
    public async Task DisposeAsync()    => await _container.StopAsync();
}
```

The container starts once per test collection, not once per test. This matters for speed — a fresh PostgreSQL container takes ~3 seconds to start; starting one per test would make the suite unusable. The xUnit mechanism that enables this is `ICollectionFixture`:

```csharp
[CollectionDefinition("Integration")]
public class IntegrationCollection : ICollectionFixture<PostgresFixture> { }

[Collection("Integration")]
public class GenerationTests(ApiFactory factory) : IClassFixture<ApiFactory> { ... }
```

xUnit shares a single `PostgresFixture` instance across all tests in the `"Integration"` collection. Each test still runs in isolation — we reset the database state between tests using transactions or explicit cleanup, depending on the test scenario.

Migrations run automatically inside `ApiFactory` on first startup. The test database is always in the correct schema state before the first test assertion runs.

---

## 2. Azurite — Azure Blob Storage without Azure

[Caresse](https://caresse.app) stores generated audio files in Azure Blob Storage. Testing storage operations against the real Azure service in CI would require a real connection string, a real storage account, and a real bill. Azurite — Microsoft's official local emulator — eliminates all three.

Azurite runs as a Docker container alongside PostgreSQL:

```csharp
private readonly IContainer _azurite = new ContainerBuilder()
    .WithImage("mcr.microsoft.com/azure-storage/azurite")
    .WithPortBinding(10000, 10000)
    .WithWaitStrategy(Wait.ForUnixContainer().UntilPortIsAvailable(10000))
    .Build();
```

The connection string uses Azurite's hardcoded default credentials — these are published in the official documentation and are not secrets:

```csharp
public string AzuriteConnectionString =>
    "DefaultEndpointsProtocol=http;" +
    "AccountName=devstoreaccount1;" +
    "AccountKey=Eby8vdM02xNOcqFlqUwJPLlmEtlCDXJ1OUzFT50uSRZ6IFsuFq2UVErCz4I6tiqIBwJI/UDZCyDXlbWy0QYfA==;" +
    "BlobEndpoint=http://127.0.0.1:10000/devstoreaccount1;";
```

In `ApiFactory`, the real `BlobServiceClient` registration is overridden with one pointing at Azurite:

```csharp
services.AddSingleton(_ => new BlobServiceClient(azuriteConnectionString));
```

Blob containers are created during fixture initialization so they exist before any test runs:

```csharp
var blobClient = new BlobServiceClient(AzuriteConnectionString);
await blobClient.CreateBlobContainerAsync("audio-output");
await blobClient.CreateBlobContainerAsync("assets");
```

From that point on, every storage operation in the application code — upload, download, delete, existence check — runs against the local emulator with no code change in the adapter. The `IBlobStoragePort` implementation doesn't know it's talking to Azurite.

---

## 3. TestAuthHandler — bypassing Firebase JWT

Firebase authentication works by validating a JWT against Google's public key endpoint. In tests, we have two problems: we don't want a network call to Google on every test run, and we can't realistically issue valid Firebase tokens from CI without a service account.

The solution is to replace the entire authentication handler at test startup — not mock it, *replace* it. The production application code that reads `user_id` from claims doesn't change at all; only the mechanism that populates those claims is swapped.

The test token is deliberately simple — just a base64-encoded user ID:

```csharp
// Helpers/AuthHelper.cs
public static class AuthHelper
{
    public const string TestUserId = "test-user-uid-001";
    public const string Scheme     = "TestAuth";

    public static string GenerateToken(string userId = TestUserId) =>
        Convert.ToBase64String(Encoding.UTF8.GetBytes(userId));
}
```

The handler decodes it and builds a `ClaimsPrincipal` with the same claim structure that Firebase would have produced:

```csharp
// TestAuthHandler.cs
public sealed class TestAuthHandler(
    IOptionsMonitor<AuthenticationSchemeOptions> options,
    ILoggerFactory logger,
    UrlEncoder encoder)
    : AuthenticationHandler<AuthenticationSchemeOptions>(options, logger, encoder)
{
    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        var userId = Context.Request.Headers.Authorization
            .ToString()
            .Replace("Bearer ", "");

        if (string.IsNullOrEmpty(userId))
            return Task.FromResult(AuthenticateResult.Fail("No token"));

        var decoded = Encoding.UTF8.GetString(Convert.FromBase64String(userId));

        var claims = new[]
        {
            new Claim(ClaimTypes.NameIdentifier, decoded),
            new Claim("user_id", decoded),
        };

        var identity  = new ClaimsIdentity(claims, Scheme);
        var principal = new ClaimsPrincipal(identity);
        var ticket    = new AuthenticationTicket(principal, Scheme);

        return Task.FromResult(AuthenticateResult.Success(ticket));
    }
}
```

In `ApiFactory`, Firebase is removed and `TestAuthHandler` is registered in its place:

```csharp
builder.ConfigureServices(services =>
{
    var firebaseDescriptor = services
        .SingleOrDefault(d => d.ServiceType == typeof(IAuthenticationSchemeProvider));
    if (firebaseDescriptor != null)
        services.Remove(firebaseDescriptor);

    services
        .AddAuthentication(AuthHelper.Scheme)
        .AddScheme<AuthenticationSchemeOptions, TestAuthHandler>(
            AuthHelper.Scheme, _ => { });
});
```

In tests, authenticating is one line — and it works for any user ID you need to simulate:

```csharp
client.DefaultRequestHeaders.Authorization =
    new AuthenticationHeaderValue("Bearer", AuthHelper.GenerateToken());

// or, to test a specific user scenario:
client.DefaultRequestHeaders.Authorization =
    new AuthenticationHeaderValue("Bearer", AuthHelper.GenerateToken("premium-user-uid-002"));
```

No Firebase project, no service account, no network. Every endpoint that requires authentication behaves identically to production.

---

## 4. WireMock.Net — stubbing RevenueCat, LLM providers, and TTS

WireMock.Net runs as an in-process HTTP server that intercepts outbound HTTP calls. Every external HTTP API that we can't or shouldn't call in CI goes through it: RevenueCat, all LLM providers (OpenAI, Grok, DeepSeek…), and all TTS providers (ElevenLabs, Cartesia…).

The server starts once and stays up for the duration of the test run:

```csharp
// Fixtures/WireMockFixture.cs
public sealed class WireMockFixture : IAsyncLifetime
{
    public WireMockServer Server { get; private set; } = null!;

    public Task InitializeAsync()
    {
        Server = WireMockServer.Start(new WireMockServerSettings
        {
            Port = 9090,
            StartAdminInterface = false,
        });
        return Task.CompletedTask;
    }

    public Task DisposeAsync()
    {
        Server.Stop();
        return Task.CompletedTask;
    }
}
```

**LLM providers.** For OpenAI, Grok, DeepSeek and others, WireMock returns a fixed completion response. The goal isn't to test model output quality — it's to verify that the pipeline correctly routes to the right endpoint, sends the right headers, handles the response shape, and passes the output to the next stage. A static placeholder is exactly the right level of fidelity for that:

```csharp
// Stubs/OpenAiStubs.cs
public static void RegisterCompletion(WireMockServer server, string apiKey) =>
    server
        .Given(Request.Create()
            .WithPath("/v1/chat/completions")
            .WithHeader("Authorization", $"Bearer {apiKey}")
            .UsingPost())
        .RespondWith(Response.Create()
            .WithStatusCode(200)
            .WithHeader("Content-Type", "application/json")
            .WithBodyAsJson(new
            {
                id      = "chatcmpl-test",
                choices = new[]
                {
                    new
                    {
                        message      = new { role = "assistant", content = "Stage output placeholder." },
                        finish_reason = "stop"
                    }
                },
                usage = new { prompt_tokens = 10, completion_tokens = 20, total_tokens = 30 }
            }));
```

**TTS providers.** TTS stubs return a minimal but valid audio payload — a real MP3 header stored as a byte array in the test project. This is important: if the stub returns an empty body, the audio assembly stage will fail downstream. The stub needs to be *valid enough* for the pipeline to proceed end-to-end:

```csharp
// Stubs/ElevenLabsStubs.cs
public static void RegisterTts(WireMockServer server, string apiKey) =>
    server
        .Given(Request.Create()
            .WithPath("/v1/text-to-speech/*")
            .WithHeader("xi-api-key", apiKey)
            .UsingPost())
        .RespondWith(Response.Create()
            .WithStatusCode(200)
            .WithHeader("Content-Type", "audio/mpeg")
            .WithBody(TestAudioFixtures.MinimalMp3));
```

**RevenueCat.** Subscription stubs are registered per-scenario, not globally. This lets each test declare exactly what RevenueCat returns for its specific case — active subscription, expired, never subscribed, webhook event — without stubs bleeding across tests:

```csharp
// inside a test
RevenueCatStubs.RegisterActiveSubscription(_wireMock.Server, AuthHelper.TestUserId);

var response = await _client.GetAsync("/api/subscription/status");
```

The static stub builders (`OpenAiStubs`, `ElevenLabsStubs`, `RevenueCatStubs`) keep test bodies readable. The JSON shapes live in one place and are reused across the entire suite.

In `ApiFactory`, all external base URLs are overridden to point at WireMock:

```csharp
config.AddInMemoryCollection(new Dictionary<string, string?>
{
    ["RevenueCat:BaseUrl"] = "http://localhost:9090",
    ["OpenAi:BaseUrl"]     = "http://localhost:9090",
    ["ElevenLabs:BaseUrl"] = "http://localhost:9090",
});
```

The production `HttpClient` instances for every provider hit `localhost:9090` in tests. No change in the adapters.

---

## 5. ApiFactory — wiring it all together

`ApiFactory` is the single assembly point. It owns the container lifecycle, overrides configuration, and swaps authentication:

```csharp
public sealed class ApiFactory : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly PostgresFixture _postgres = new();
    private readonly WireMockFixture _wireMock = new();

    public WireMockServer WireMock => _wireMock.Server;

    public async Task InitializeAsync()
    {
        await _postgres.InitializeAsync();
        await _wireMock.InitializeAsync();
    }

    public new async Task DisposeAsync()
    {
        await _postgres.DisposeAsync();
        await _wireMock.DisposeAsync();
    }

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureAppConfiguration((_, config) =>
        {
            config.AddInMemoryCollection(new Dictionary<string, string?>
            {
                ["ConnectionStrings:Default"]    = _postgres.ConnectionString,
                ["Azure:BlobStorage:Connection"] = _postgres.AzuriteConnectionString,
                ["RevenueCat:BaseUrl"]           = "http://localhost:9090",
                ["OpenAi:BaseUrl"]               = "http://localhost:9090",
                ["ElevenLabs:BaseUrl"]           = "http://localhost:9090",
            });
        });

        builder.ConfigureServices(services =>
        {
            RemoveFirebaseAuth(services);
            services
                .AddAuthentication(AuthHelper.Scheme)
                .AddScheme<AuthenticationSchemeOptions, TestAuthHandler>(
                    AuthHelper.Scheme, _ => { });

            var sp = services.BuildServiceProvider();
            using var scope = sp.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
            db.Database.Migrate();
        });
    }

    private static void RemoveFirebaseAuth(IServiceCollection services)
    {
        var descriptor = services.SingleOrDefault(
            d => d.ServiceType == typeof(IAuthenticationSchemeProvider));
        if (descriptor != null) services.Remove(descriptor);
    }
}
```

Tests receive `ApiFactory` via `IClassFixture<ApiFactory>` and call `factory.CreateClient()`. That client hits the real ASP.NET Core pipeline, with the real middleware, the real DI container, and real adapters — just with local infrastructure underneath.

---

## 6. Writing the tests

Convention throughout the suite: `Method_Condition_ShouldBehavior`. It's mechanical, but it pays back when a test fails at 2am and you need to know at a glance what broke and under what condition.

```csharp
[Collection("Integration")]
public class SubscriptionTests(ApiFactory factory) : IClassFixture<ApiFactory>
{
    private readonly HttpClient _client = factory.CreateClient();

    [Fact]
    public async Task GetStatus_WithActiveSubscription_ShouldReturnPremium()
    {
        _client.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", AuthHelper.GenerateToken());

        RevenueCatStubs.RegisterActiveSubscription(factory.WireMock, AuthHelper.TestUserId);

        var response = await _client.GetAsync("/api/subscription/status");
        var body     = await response.Content.ReadFromJsonAsync<SubscriptionStatusDto>();

        response.StatusCode.Should().Be(HttpStatusCode.OK);
        body!.Tier.Should().Be("premium");
        body.ExpiresAt.Should().BeAfter(DateTime.UtcNow);
    }

    [Fact]
    public async Task GetStatus_WithExpiredSubscription_ShouldReturnFree()
    {
        _client.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", AuthHelper.GenerateToken());

        RevenueCatStubs.RegisterExpiredSubscription(factory.WireMock, AuthHelper.TestUserId);

        var response = await _client.GetAsync("/api/subscription/status");
        var body     = await response.Content.ReadFromJsonAsync<SubscriptionStatusDto>();

        response.StatusCode.Should().Be(HttpStatusCode.OK);
        body!.Tier.Should().Be("free");
    }
}
```

Each test is self-contained: it sets up its own WireMock stubs, makes one HTTP call, and asserts on the response. No shared mutable state, no test ordering dependencies.

---

## 7. CI configuration

One of the practical wins of this setup is how little the CI configuration needs to do:

```yaml
# .github/workflows/tests.yml
jobs:
  integration-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '10.x'

      - name: Run integration tests
        run: dotnet test tests/Caresse.Api.IntegrationTests --configuration Release
```

No `services:` block, no `docker-compose`, no environment variables for external APIs. Testcontainers pulls the PostgreSQL and Azurite images at runtime — Docker is available on `ubuntu-latest` out of the box. WireMock is in-process. The runner needs nothing pre-installed beyond .NET and Docker.

Total runtime: ~85 seconds on a cold runner (image pull included), ~40 seconds on a warm runner with cached images. For a suite that covers the full request lifecycle across every external dependency, that's acceptable.

---

## What this replaces

Before this setup, the test strategy was unit tests with NSubstitute mocks for everything external, plus manual QA against a shared staging environment before each release.

The mocks gave false confidence: they verified that our code called the right methods, not that the right things happened. The staging environment was a bottleneck: only one person could run a meaningful end-to-end test at a time, and "it works on staging" was often wrong by the time it mattered.

The integration suite replaced both. Every PR now runs against a real database with the real schema and real migrations, real blob storage operations, and real HTTP contracts for every provider. The only things that aren't real are the vendors we can't run locally — and for those, WireMock enforces the actual HTTP contract, not a hand-rolled assumption. The pipeline runs end-to-end: prompt in, audio bytes out, every adapter exercised, no real API key required.

---

## Coming next in *Building Caresse*

**Building Caresse #4** covers multi-provider LLM integration: the `IPromptExecutorPort` interface, how five different vendors implement it, how we handle streaming vs. non-streaming APIs in a unified way, and the NSubstitute tests that make provider swaps safe.

Follow on Medium to catch it when it drops.

---

## See it running

- 🌐 **[caresse.app](https://caresse.app)** — overview, supported languages (FR / EN / ES / PT)
- 📱 **[Caresse on Google Play](https://play.google.com/store/apps/details?id=com.flareai.caresse)**

*Found a better approach to any of these? Reply on Medium — I read every message.*

---

**Technical level:** 🟩🟩🟦 Confirmed — assumes familiarity with ASP.NET Core, xUnit, and Docker basics.

**Series:** Building Caresse — inside the stack of a bootstrapped AI product.
**Previous — #2:** the multi-phase LLM pipeline.
**Next up — #4:** multi-provider LLM integration with ports & adapters.
