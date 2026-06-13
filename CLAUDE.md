# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

`DotNetRateLimiter` is a NuGet library (`PackageId: DotNetRateLimiter`) that rate-limits ASP.NET Core requests at the **filter** level rather than the middleware level. The motivating idea: middleware affects every request, but MVC action filters and minimal-API endpoint filters let you limit only specific critical endpoints. Storage is in-memory by default, or Redis when a connection string is configured (recommended for distributed apps).

## Commands

```bash
# Build the whole solution (uses the .slnx solution file)
dotnet build DotNet.RateLimit.slnx -c Release

# Run all tests
dotnet test test/DotNet.RateLimiter.Test/DotNet.RateLimiter.Test.csproj

# Run a single test by name
dotnet test test/DotNet.RateLimiter.Test/DotNet.RateLimiter.Test.csproj --filter "FullyQualifiedName~InMemoryRateLimitTest"

# Pack the NuGet package
dotnet pack src/DotNet.RateLimiter/DotNet.RateLimiter.csproj -c Release
```

The Redis tests (`RedisRateLimitTest`) spin up a real Redis 7.4 container via **Testcontainers**, so Docker must be running for the full suite to pass.

## Architecture

Request flow is identical for MVC and minimal API; only the entry point differs. Both funnel into a single `IRateLimitCoordinator` that builds a key and delegates the counting decision to an `IRateLimitService`.

```
MVC:          [RateLimit] attribute (RateLimitFilterFactory.cs, class "RateLimit")
              → resolves RateLimitAttribute (IAsyncActionFilter) from DI
              → IRateLimitCoordinator.CheckRateLimitAsync(ActionExecutingContext, ...)

Minimal API:  .WithRateLimiter(opts) (EndPointExtensions.cs)
              → RateLimitEndPointFilter (IEndpointFilter)
              → IRateLimitCoordinator.CheckRateLimitAsync(EndpointFilterInvocationContext, ...)

Both:         RateLimitCoordinator → IRateLimitService.HasAccessAsync(key, periodInSec, limit)
                                      → InMemoryRateLimitService  OR  RedisRateLimitService
```

Key concepts:

- **Two attribute types, easy to confuse.** `RateLimit` (in `RateLimitFilterFactory.cs`) is the public `IFilterFactory` users apply to controllers/actions. It does NOT do the work — its `CreateInstance` resolves `RateLimitAttribute` (in `RateLimitAttribute.cs`) from DI, which is the actual `IAsyncActionFilter`. The DI-resolved instance is what gets `IRateLimitCoordinator` injected.

- **`RateLimitCoordinator`** (`Implementations/RateLimitCoordinator.cs`) is the brain. It runs `InitialChecking` (disabled-check, IP/client-id whitelist short-circuits, validates `Limit > 0` and `PeriodInSec > 0`), then composes the rate-limit key from: prefix (client-id header if `ClientIdentifier` is set, else IP) + HTTP method + controller + (action, only when `Scope == Action`) + route params + query params + body params. **All exceptions are swallowed and the request is allowed through** (`return true`) — rate limiting must never take down the app.

- **Key scope.** `RateLimitScope.Action` (default) counts each action separately; `RateLimitScope.Controller` omits the action name so the limit is shared across the whole controller.

- **Body params (MVC only).** Extracted in `SetBodyParamsRateLimitKey` by JSON-serializing `ActionArguments` and reading top-level properties of the *first* argument. Nested properties are not supported.

- **Two `IRateLimitService` implementations:**
  - `InMemoryRateLimitService` — `IMemoryCache` + `AsyncKeyedLocker<string>` (registered as singleton) to serialize per-key access. Note the off-by-one: it rejects when `++entry.Total >= limit`.
  - `RedisRateLimitService` — atomic `INCR` + conditional `EXPIRE` via an inlined Lua script (set TTL only on first hit). Rejects when `count > limit`.

- **DI registration** is in `ServiceCollectionExtension.cs`. `AddRateLimitService(config)` auto-selects Redis vs in-memory based on whether `RateLimitOption:RedisConnection` is set (`RateLimitOptions.HasRedis`). Two overloads accept an existing `IConnectionMultiplexer` or `IDatabase` and use `TryAdd*` so they reuse the app's existing Redis connection instead of creating a new one (see `USAGE_EXISTING_REDIS.md`).

- **Response shaping.** `RateLimitResponseBuilder.BuildResponse` produces the 429 body. Default is `{"message":..., "code":...}`; if `RateLimitOptions.ResponseStructure` is set it does placeholder substitution for `$(ErrorMessage)` and `$(HttpStatusCode)`.

## Multi-targeting

The library targets `netstandard2.0;netstandard2.1;net8.0;net9.0;net10.0` with framework-conditional package references in the `.csproj`. Feature gates matter when editing:

- Minimal-API support (`EndPointExtensions.WithRateLimiter`) is `#if NET7_0_OR_GREATER`.
- `RateLimitEndPointFilter` is `#if NET8_0_OR_GREATER`.
- The `EndpointFilterInvocationContext` overload of `CheckRateLimitAsync` is `#if NET7_0_OR_GREATER`.

When changing public API, verify it compiles for the netstandard targets, which lack `Microsoft.AspNetCore.App` and use the `Microsoft.AspNetCore.*.Abstractions` packages instead.

## Configuration (`RateLimitOption` section)

Bound to `RateLimitOptions.cs`. All optional: `EnableRateLimit` (default true), `HttpStatusCode` (429), `ErrorMessage`, `ResponseStructure`, `RedisConnection` (presence switches to Redis), `IpHeaderName` (`X-Forwarded-For`), `IpWhiteList`, `ClientIdentifier` (header name; if present and found, used instead of IP), `ClientIdentifierWhiteList`.

## Testing notes

Tests use **xUnit + Xunit.DependencyInjection + Shouldly**. `Startup`/`StartupRedis` classes (in `Startup.cs`) configure the DI container per the `Xunit.DependencyInjection` convention; `StartupRedis` injects the Testcontainers Redis connection string into config before calling `AddRateLimitService`. `TestInitializer.cs` builds the `ActionContext`/`EndpointFilterInvocationContext` fixtures and exposes `GetRandomIpAddress()` so tests get isolated keys. The library exposes internals to the test project via `InternalsVisibleTo`.

## CI / Release

- `azure-pipelines.yml` — PR/CI build + test on .NET 8/9/10 SDKs (this is the badge build).
- `.github/workflows/package.yml` — manual (`workflow_dispatch`) release that uses GitVersion, packs, and pushes to GitHub Packages.
