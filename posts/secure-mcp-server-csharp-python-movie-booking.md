---
Author: "Mohammed Iliyas Patel"
title: "Secure MCP Server in C# and Python: Movie Booking Example"
description: "Build a secure MCP server in C# and Python with a movie booking example: JWT auth, tool permissions, rate limiting, audit logs and automated tests."
slug: secure-mcp-server-csharp-python-movie-booking
primary_keyword: secure MCP server
secondary_keywords:
  - MCP server C#
  - MCP server Python
  - MCP authentication
  - MCP authorization
long_tail_keywords:
  - MCP server rate limiting
  - MCP tool permissions
  - MCP audit logging
  - JWT authentication MCP server
  - MCP server tutorial
schema: Article, BreadcrumbList, FAQPage
og_title: "Build a Secure MCP Server in C# and Python (Movie Ticket Booking Example)"
og_description: "Identity, permissions, usage limits and tests for MCP tools in .NET and Python."
---

# Build a Secure MCP Server in C# and Python (Movie Ticket Booking Example)

A secure MCP server needs more than a valid access token. Once an AI agent can call tools that search, reserve and sell, you need to know who called what, stop callers from exceeding their permissions, and cap how much work any one caller can trigger.

In this tutorial you build a **movie ticket booking MCP server** in both **C# (ASP.NET Core)** and **Python**, with identical behavior in each:

- **Authentication:** verify who is calling (JWT validation)
- **Authorization:** enforce which tools each caller may use (MCP tool permissions)
- **Rate limiting:** cost-weighted limits per verified caller (MCP server rate limiting)
- **Audit logging:** record every call without logging argument values (MCP audit logging)
- **Automated tests:** prove the controls keep working after every change

**Who this is for:** .NET and Python developers building MCP servers that will face real users or agents.
**Repository:** `[AUTHOR: repository link]`
**Last tested:** `[AUTHOR: date]` with .NET `[AUTHOR: version]`, `ModelContextProtocol.AspNetCore` `[AUTHOR: version]`, Python `[AUTHOR: version]`, `mcp` `[AUTHOR: version]`.

> **Quick answer:** To secure an MCP server, validate tokens on the server, authorize every tool call in deterministic code, rate-limit by verified subject with per-tool costs, log argument names but not values, and test all of it automatically.

> **Honesty note on the code:** SDK APIs for MCP authentication and request filters change between releases. Pin exact versions and run every snippet before relying on it. Snippets marked **Verify** touch SDK-specific APIs.

## Table of contents

1. [How MCP server security works: five controls](#how-mcp-server-security-works-five-controls)
2. [Step 1: Define tool permissions and costs](#step-1-define-tool-permissions-and-costs)
3. [Step 2: Write tools that treat arguments as untrusted](#step-2-write-tools-that-treat-arguments-as-untrusted)
4. [Step 3: Authenticate callers](#step-3-authenticate-callers)
5. [Step 4: Authorize every tool call](#step-4-authorize-every-tool-call)
6. [Step 5: Rate limit per verified subject](#step-5-rate-limit-per-verified-subject)
7. [Step 6: Add audit logging](#step-6-add-audit-logging)
8. [Step 7: Wire it into the MCP server](#step-7-wire-it-into-the-mcp-server)
9. [Step 8: Test the controls](#step-8-test-the-controls)
10. [Step 9: Trace one request end to end](#step-9-trace-one-request-end-to-end)
11. [Step 10: Harden for production](#step-10-harden-for-production)
12. [Common MCP security mistakes](#common-mcp-security-mistakes)
13. [C# vs Python: MCP security compared](#c-vs-python-mcp-security-compared)
14. [Can tests prove an MCP server is safe?](#can-tests-prove-an-mcp-server-is-safe)
15. [When to use a gateway instead](#when-to-use-a-gateway-instead)
16. [Monitoring and alerting for MCP servers](#monitoring-and-alerting-for-mcp-servers)
17. [Production checklist](#production-checklist)
18. [FAQ](#faq)
19. [Sources](#sources)

---

## How MCP server security works: five controls

Every request passes through the same pipeline:

```text
Caller -> Token validation -> Authorization -> Rate limiter -> Tool (validates input) -> Activity log
```

| Control | Question it answers | C# | Python |
|---|---|---|---|
| Authentication | Who is calling? | ASP.NET Core JWT bearer | MCP `TokenVerifier` |
| Authorization | May this caller use this tool? | `[Authorize]` + SDK filters | Explicit scope checks |
| Rate limiting | Is one caller consuming too much? | `PartitionedRateLimiter` | Per-subject token bucket |
| Input validation | Is this argument acceptable? | Regex and length checks | Regex and length checks |
| Audit logging | What happened? | `ILogger` + `LoggerMessage` | Structured JSON via `logging` |

The rule behind all five: **identity comes from a verified token, decisions are made by server code, and tool arguments are untrusted.** The language model never decides who may do what.

---

## Step 1: Define tool permissions and costs

Decide the policy before writing code. Every later step reads from this table.

| Tool | C# roles | Python scopes | Cost |
|---|---|---|---:|
| `search_movies` | Customer, PremiumCustomer, CinemaAdmin | `movies:read`, `tickets:book`, `cinema:admin` | 1 |
| `book_tickets` | Customer, PremiumCustomer, CinemaAdmin | `tickets:book`, `cinema:admin` | 5 |
| `download_daily_sales` | CinemaAdmin | `cinema:admin` | 10 |

Rate-limit budget: a **30-token bucket that refills every minute**. That allows six bookings (5 each) or three sales downloads (10 each) in an immediate burst. These are demonstration values; load-test before choosing production numbers.

Personas used in the tests:

- **Search-only user:** `movies:read` only. Can search, cannot book.
- **Customer:** `tickets:book`. Can search and book.
- **Cinema admin:** `cinema:admin`. Can do everything, including sales downloads.

Unknown tools must **fail closed** (denied) and get a non-trivial default cost.

---

## Step 2: Write tools that treat arguments as untrusted

Validate with an allow-list pattern that must match the **entire** input. Anchor with `\z` in .NET (plain `$` accepts a trailing newline) and use `fullmatch` in Python.

### C#

```csharp
// Tools/CinemaTools.cs
using System.ComponentModel;
using System.Text.RegularExpressions;
using Microsoft.AspNetCore.Authorization;
using ModelContextProtocol;
using ModelContextProtocol.Server;

namespace McpSecureDemo.Tools;

[McpServerToolType]
public sealed partial class CinemaTools
{
    [GeneratedRegex(@"^SHOW-[0-9]{1,8}\z")]
    private static partial Regex ShowIdPattern();

    [McpServerTool(Name = "search_movies")]
    [Description("Search movies by title or genre.")]
    [Authorize(Roles = "Customer,PremiumCustomer,CinemaAdmin")]
    public static string SearchMovies(string query)
    {
        if (string.IsNullOrWhiteSpace(query) || query.Length > 200)
            throw new McpException("query must be 1 to 200 characters.");

        return "[demo] 2 movies matched.";
    }

    [McpServerTool(Name = "book_tickets")]
    [Description("Book tickets for a show, for example SHOW-1001.")]
    [Authorize(Roles = "Customer,PremiumCustomer,CinemaAdmin")]
    public static string BookTickets(string showId)
    {
        if (!ShowIdPattern().IsMatch(showId ?? string.Empty))
            throw new McpException("showId must look like SHOW-1234.");

        return $"[demo] Tickets booked for {showId}.";
    }

    [McpServerTool(Name = "download_daily_sales")]
    [Description("Download the daily cinema sales report. Expensive. CinemaAdmin only.")]
    [Authorize(Roles = "CinemaAdmin")]
    public static string DownloadDailySales() => "[demo] daily sales download started.";
}
```

### Python

```python
# src/mcp_secure_demo/tools.py
import re

SHOW_ID_PATTERN = re.compile(r"SHOW-[0-9]{1,8}")


class ToolValidationError(ValueError):
    """Raised when a tool argument fails validation."""


def search_movies_impl(query: str) -> str:
    query = query.strip()
    if not query or len(query) > 200:
        raise ToolValidationError("query must contain between 1 and 200 characters")
    return "[demo] 2 movies matched."


def book_tickets_impl(show_id: str) -> str:
    if not SHOW_ID_PATTERN.fullmatch(show_id or ""):
        raise ToolValidationError("show_id must look like SHOW-1234")
    return f"[demo] Tickets booked for {show_id}."


def download_daily_sales_impl() -> str:
    return "[demo] daily sales download started."
```

Keep tool logic in plain functions (`*_impl`) so it is testable without an MCP transport.

---

## Step 3: Authenticate callers

Authentication answers "who is calling?" It must happen on the server, on every request, using a validated token. Never use `X-User` headers, tool arguments or anything the caller controls as identity.

### C#: JWT bearer authentication

Configure issuer, audience and lifetime validation in `Program.cs` (full file in [Step 7](#step-7-wire-it-into-the-mcp-server)). The essentials:

```csharp
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = builder.Configuration["Auth:Authority"];
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidAudience = builder.Configuration["Auth:Audience"],
            RoleClaimType = "roles",
        };
    });
```

Setting `Authority` lets the middleware fetch signing keys from your identity provider's metadata, so you never hard-code keys.

### Python: MCP authentication with `TokenVerifier`

The Python SDK asks you to supply a `TokenVerifier`. A real one verifies signature, issuer, expiry, audience and required claims. This example uses PyJWT with a JWKS endpoint.

> **Verify:** `AccessToken` fields differ between SDK versions. Some versions have no `subject` field, so this example subclasses `AccessToken` to carry one. Confirm against your pinned version.

```python
# src/mcp_secure_demo/auth.py
import asyncio

import jwt
from jwt import PyJWKClient
from mcp.server.auth.provider import AccessToken


class VerifiedToken(AccessToken):
    subject: str


class JwtTokenVerifier:
    def __init__(self, jwks_url: str, issuer: str, audience: str) -> None:
        self._jwks = PyJWKClient(jwks_url)
        self._issuer = issuer
        self._audience = audience

    async def verify_token(self, token: str) -> VerifiedToken | None:
        try:
            # PyJWKClient does blocking network I/O; keep it off the event loop.
            signing_key = await asyncio.to_thread(
                self._jwks.get_signing_key_from_jwt, token
            )
            claims = jwt.decode(
                token,
                signing_key.key,
                algorithms=["RS256"],          # never accept "none" or HS* here
                audience=self._audience,
                issuer=self._issuer,
                options={"require": ["exp", "iss", "aud", "sub"]},
            )
        except jwt.PyJWTError:
            return None                        # invalid token -> 401

        raw_scopes = claims.get("scp") or claims.get("scope") or ""
        scopes = raw_scopes.split() if isinstance(raw_scopes, str) else list(raw_scopes)

        return VerifiedToken(
            token=token,
            client_id=str(claims.get("azp") or claims.get("client_id") or "unknown"),
            scopes=scopes,
            expires_at=int(claims["exp"]),
            subject=str(claims["sub"]),
        )
```

Do not publish a verifier that only decodes a token without validating it. That is the most common authentication bug in MCP examples.

---

## Step 4: Authorize every tool call

Authentication says who. **Authorization says what they may do**, and it must run for every call.

### C#

`[Authorize(Roles = ...)]` on each tool, plus `.AddAuthorizationFilters()` on the MCP builder so the SDK enforces the attributes. **Verify** filter behavior and ordering against your SDK version with a test (Step 8).

### Python

```python
# src/mcp_secure_demo/security.py
TOOL_SCOPES: dict[str, frozenset[str]] = {
    "search_movies": frozenset({"movies:read", "tickets:book", "cinema:admin"}),
    "book_tickets": frozenset({"tickets:book", "cinema:admin"}),
    "download_daily_sales": frozenset({"cinema:admin"}),
}

TOOL_COSTS: dict[str, int] = {
    "search_movies": 1,
    "book_tickets": 5,
    "download_daily_sales": 10,
}

DEFAULT_TOOL_COST = 5
TOKEN_LIMIT = 30


class AccessDeniedError(PermissionError):
    pass


class RateLimitExceededError(RuntimeError):
    pass


def authorize_tool(tool_name: str, caller_scopes: set[str]) -> None:
    allowed = TOOL_SCOPES.get(tool_name)
    if allowed is None:
        raise AccessDeniedError("Unknown tool")          # fail closed
    if caller_scopes.isdisjoint(allowed):
        raise AccessDeniedError("Access denied")
```

Never accept roles or scopes from tool arguments. They come from the validated token only.

---

## Step 5: Rate limit per verified subject

A token bucket with **per-tool costs** protects expensive operations better than a flat requests-per-minute cap. Partition by verified subject, never by a caller-supplied value.

### C#: inside `ToolCallGuard`

The full guard is in [Step 6](#step-6-add-audit-logging), since limiting and logging share one code path. The limiter itself:

```csharp
_limiter = PartitionedRateLimiter.Create<string, string>(subject =>
    RateLimitPartition.GetTokenBucketLimiter(subject, _ =>
        new TokenBucketRateLimiterOptions
        {
            TokenLimit = 30,
            TokensPerPeriod = 30,
            ReplenishmentPeriod = TimeSpan.FromMinutes(1),
            QueueLimit = 0,
            AutoReplenishment = true,
        }));
```

### Python: process-local async token bucket

> **This implementation is process-local. It does not provide a cluster-wide limit.** See [Step 10](#step-10-harden-for-production).

```python
# Add to src/mcp_secure_demo/security.py
import asyncio
import time
from dataclasses import dataclass


@dataclass(slots=True)
class Bucket:
    tokens: float
    updated_at: float


class PerSubjectTokenBucket:
    def __init__(self, capacity: int, refill_tokens: int, refill_period_seconds: float) -> None:
        self._capacity = capacity
        self._refill_rate = refill_tokens / refill_period_seconds
        self._buckets: dict[str, Bucket] = {}
        self._lock = asyncio.Lock()

    async def acquire(self, subject: str, cost: int) -> bool:
        if not subject:
            return False

        cost = max(1, min(cost, self._capacity))
        now = time.monotonic()

        async with self._lock:
            bucket = self._buckets.get(subject)
            if bucket is None:
                bucket = Bucket(tokens=float(self._capacity), updated_at=now)
                self._buckets[subject] = bucket

            elapsed = max(0.0, now - bucket.updated_at)
            bucket.tokens = min(float(self._capacity), bucket.tokens + elapsed * self._refill_rate)
            bucket.updated_at = now

            if bucket.tokens < cost:
                return False

            bucket.tokens -= cost
            return True


limiter = PerSubjectTokenBucket(capacity=TOKEN_LIMIT, refill_tokens=TOKEN_LIMIT, refill_period_seconds=60)
```

Design note: the limiter runs **before** the tool executes, so denied and malformed calls also consume budget. That is intentional; it stops an attacker from probing for free.

---

## Step 6: Add audit logging

Each call produces one record: who, which client, which tool, what outcome, how long, what cost. Log **argument names and sizes, not values**. Values can contain secrets, personal data and prompts.

Outcomes: `Success`, `ToolError`, `Rejected`, `RateLimited`, `Cancelled`, `Failed`.

### C#: `ToolCallGuard`

> **Verify:** `CallToolResult`, `TextContentBlock` and the filter context types against your pinned SDK.

```csharp
// Security/ToolCallGuard.cs
using System.Diagnostics;
using System.Security.Claims;
using System.Text.Json;
using System.Threading.RateLimiting;
using Microsoft.Extensions.Logging;
using ModelContextProtocol;
using ModelContextProtocol.Protocol;

namespace McpSecureDemo.Security;

public enum ToolCallOutcome { Success, ToolError, Rejected, RateLimited, Cancelled, Failed }

public sealed partial class ToolCallGuard : IDisposable
{
    private const int TokenLimit = 30;
    private const int DefaultCost = 5;

    private static readonly IReadOnlyDictionary<string, int> Costs =
        new Dictionary<string, int>
        {
            ["search_movies"] = 1,
            ["book_tickets"] = 5,
            ["download_daily_sales"] = 10,
        };

    private readonly ILogger _activityLogger;
    private readonly PartitionedRateLimiter<string> _limiter;

    public ToolCallGuard(ILoggerFactory loggerFactory)
    {
        _activityLogger = loggerFactory.CreateLogger("McpBookingActivity");

        _limiter = PartitionedRateLimiter.Create<string, string>(subject =>
            RateLimitPartition.GetTokenBucketLimiter(subject, _ =>
                new TokenBucketRateLimiterOptions
                {
                    TokenLimit = TokenLimit,
                    TokensPerPeriod = TokenLimit,
                    ReplenishmentPeriod = TimeSpan.FromMinutes(1),
                    QueueLimit = 0,
                    AutoReplenishment = true,
                }));
    }

    public async ValueTask<CallToolResult> RunAsync(
        ClaimsPrincipal? user,
        string? toolName,
        IDictionary<string, JsonElement>? arguments,
        Func<ValueTask<CallToolResult>> next)
    {
        var started = Stopwatch.GetTimestamp();
        var tool = Clean(toolName);
        var cost = Math.Min(Costs.GetValueOrDefault(tool, DefaultCost), TokenLimit);
        var subject = GetSubject(user);
        var outcome = ToolCallOutcome.Failed;

        try
        {
            if (subject is null)
                throw new McpException("Access denied: the token has no subject.");

            using var lease = _limiter.AttemptAcquire(subject, cost);

            if (!lease.IsAcquired)
            {
                outcome = ToolCallOutcome.RateLimited;
                var wait = lease.TryGetMetadata(MetadataName.RetryAfter, out var retry)
                    ? (int)Math.Ceiling(retry.TotalSeconds)
                    : 60;

                return new CallToolResult
                {
                    IsError = true,
                    Content = [new TextContentBlock { Text = $"Rate limit exceeded. Retry in about {wait} seconds." }],
                };
            }

            var result = await next();
            outcome = result.IsError == true ? ToolCallOutcome.ToolError : ToolCallOutcome.Success;
            return result;
        }
        catch (OperationCanceledException) { outcome = ToolCallOutcome.Cancelled; throw; }
        catch (McpException) { outcome = ToolCallOutcome.Rejected; throw; }
        catch { outcome = ToolCallOutcome.Failed; throw; }
        finally
        {
            LogToolActivity(
                subject ?? "unknown",
                GetClientId(user),
                tool,
                outcome,
                (long)Stopwatch.GetElapsedTime(started).TotalMilliseconds,
                cost,
                ArgNames(arguments),
                ArgChars(arguments),
                Activity.Current?.TraceId.ToString());
        }
    }

    [LoggerMessage(
        EventId = 1001,
        Level = LogLevel.Information,
        Message = "mcp_tool_call subject={Subject} client={ClientId} tool={Tool} outcome={Outcome} durationMs={DurationMs} cost={Cost} argNames={ArgNames} argChars={ArgChars} traceId={TraceId}")]
    private partial void LogToolActivity(
        string subject, string? clientId, string tool, ToolCallOutcome outcome,
        long durationMs, int cost, string argNames, int argChars, string? traceId);

    private static string? GetSubject(ClaimsPrincipal? user) =>
        user?.FindFirst("sub")?.Value ?? user?.FindFirst(ClaimTypes.NameIdentifier)?.Value;

    private static string? GetClientId(ClaimsPrincipal? user) =>
        user?.FindFirst("azp")?.Value ?? user?.FindFirst("client_id")?.Value;

    // Strip control characters and bound length so callers cannot forge log lines.
    private static string Clean(string? value)
    {
        if (string.IsNullOrEmpty(value)) return "(none)";
        var text = value.Length > 100 ? value[..100] : value;
        return string.Concat(text.Where(c => !char.IsControl(c)));
    }

    private static string ArgNames(IDictionary<string, JsonElement>? args) =>
        args is null ? "" : Clean(string.Join(",", args.Keys.Take(20)));

    private static int ArgChars(IDictionary<string, JsonElement>? args) =>
        args?.Values.Sum(v => v.GetRawText().Length) ?? 0;

    public void Dispose() => _limiter.Dispose();
}
```

### Python: structured records and guarded execution

```python
# Add to src/mcp_secure_demo/security.py
import inspect
import json
import logging
import re
from collections.abc import Awaitable, Callable
from enum import StrEnum
from time import perf_counter
from typing import Any, TypeVar

activity_logger = logging.getLogger("mcp.booking.activity")
CONTROL_CHARACTERS = re.compile(r"[\x00-\x1f\x7f]")
T = TypeVar("T")


class ToolCallOutcome(StrEnum):
    SUCCESS = "Success"
    TOOL_ERROR = "ToolError"
    REJECTED = "Rejected"
    RATE_LIMITED = "RateLimited"
    CANCELLED = "Cancelled"
    FAILED = "Failed"


def clean_log_text(value: object, limit: int = 100) -> str:
    text = str(value) if value is not None else "(none)"
    return CONTROL_CHARACTERS.sub("", text)[:limit]


def argument_metadata(arguments: dict[str, Any] | None) -> tuple[list[str], int]:
    if not arguments:
        return [], 0
    names = [clean_log_text(n) for n in list(arguments.keys())[:20]]
    chars = sum(len(json.dumps(v, default=str)) for v in arguments.values())
    return names, chars


def write_activity_record(
    *, subject: str, client_id: str | None, tool: str, outcome: ToolCallOutcome,
    duration_ms: int, cost: int, arguments: dict[str, Any] | None, trace_id: str | None,
) -> None:
    arg_names, arg_chars = argument_metadata(arguments)
    record = {
        "event": "mcp_tool_call",
        "subject": clean_log_text(subject),
        "client_id": clean_log_text(client_id),
        "tool": clean_log_text(tool),
        "outcome": outcome.value,
        "duration_ms": duration_ms,
        "cost": cost,
        "arg_names": arg_names,
        "arg_chars": arg_chars,
        "trace_id": clean_log_text(trace_id),
    }
    activity_logger.info("%s", json.dumps(record, separators=(",", ":")))


async def execute_guarded(
    *,
    subject: str | None,
    client_id: str | None,
    scopes: set[str],
    tool_name: str,
    arguments: dict[str, Any] | None,
    operation: Callable[[], T | Awaitable[T]],
    trace_id: str | None = None,
) -> T:
    started = perf_counter()
    outcome = ToolCallOutcome.FAILED
    cost = min(TOOL_COSTS.get(tool_name, DEFAULT_TOOL_COST), TOKEN_LIMIT)

    try:
        if not subject:
            raise AccessDeniedError("Access denied: verified subject is missing")

        authorize_tool(tool_name, scopes)

        if not await limiter.acquire(subject, cost):
            raise RateLimitExceededError("Rate limit exceeded. Retry later.")

        result = operation()
        if inspect.isawaitable(result):
            result = await result

        outcome = ToolCallOutcome.SUCCESS
        return result

    except asyncio.CancelledError:
        outcome = ToolCallOutcome.CANCELLED
        raise
    except RateLimitExceededError:
        outcome = ToolCallOutcome.RATE_LIMITED
        raise
    except (AccessDeniedError, ValueError):
        outcome = ToolCallOutcome.REJECTED
        raise
    except Exception:
        outcome = ToolCallOutcome.FAILED
        raise
    finally:
        write_activity_record(
            subject=subject or "unknown",
            client_id=client_id,
            tool=tool_name,
            outcome=outcome,
            duration_ms=int((perf_counter() - started) * 1000),
            cost=cost,
            arguments=arguments,
            trace_id=trace_id,
        )
```

---

## Step 7: Wire it into the MCP server

### C#: `Program.cs`

> **Verify:** `WithRequestFilters`, `AddCallToolFilter`, `AddAuthorizationFilters` and `context.User` against your pinned SDK. Filter registration order decides which filter wraps which, so confirm the order with a test.

```csharp
// Program.cs
using System.Security.Claims;
using System.Threading.RateLimiting;
using McpSecureDemo.Security;
using McpSecureDemo.Tools;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.AspNetCore.RateLimiting;
using Microsoft.IdentityModel.Tokens;

var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        var authority = builder.Configuration["Auth:Authority"];
        if (!string.IsNullOrEmpty(authority))
            options.Authority = authority;

        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidAudience = builder.Configuration["Auth:Audience"],
            RoleClaimType = "roles",
        };
    });

builder.Services.AddAuthorization();
builder.Services.AddSingleton<ToolCallGuard>();

// Coarse HTTP-level limit: a cheap first line of defense before MCP parsing.
builder.Services.AddRateLimiter(options =>
{
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(ctx =>
    {
        var key = ctx.User.FindFirst("sub")?.Value
            ?? ctx.User.FindFirst(ClaimTypes.NameIdentifier)?.Value
            ?? ctx.Connection.RemoteIpAddress?.ToString()
            ?? "unknown";

        return RateLimitPartition.GetFixedWindowLimiter(key, _ => new FixedWindowRateLimiterOptions
        {
            PermitLimit = 300,
            Window = TimeSpan.FromMinutes(1),
            QueueLimit = 0,
        });
    });
});

builder.Services
    .AddMcpServer()
    .WithHttpTransport()
    .WithTools<CinemaTools>()
    .WithRequestFilters(filters =>
    {
        filters.AddCallToolFilter(next => async (context, cancellationToken) =>
        {
            var guard = context.Services!.GetRequiredService<ToolCallGuard>();
            return await guard.RunAsync(
                context.User,
                context.Params?.Name,
                context.Params?.Arguments,
                () => next(context, cancellationToken));
        });
    })
    .AddAuthorizationFilters();

var app = builder.Build();

app.UseAuthentication();   // must run before the limiter so ctx.User is populated
app.UseRateLimiter();
app.UseAuthorization();

app.MapMcp("/mcp").RequireAuthorization();
app.Run();

public partial class Program { }
```

Add the HTTP-level limiter in addition to the tool-level one. They protect different layers.

### Python: `server.py`

> **Verify:** the official SDK exposes `FastMCP` in `mcp.server.fastmcp` in many releases; newer versions may rename or move it. Check `AuthSettings` fields and `get_access_token` for your pinned version.

```python
# src/mcp_secure_demo/server.py
import os
from typing import Any

from mcp.server.auth.middleware.auth_context import get_access_token
from mcp.server.auth.settings import AuthSettings
from mcp.server.fastmcp import FastMCP

from .auth import JwtTokenVerifier
from .security import execute_guarded
from .tools import book_tickets_impl, download_daily_sales_impl, search_movies_impl

mcp = FastMCP(
    "McpSecureDemo",
    token_verifier=JwtTokenVerifier(
        jwks_url=os.environ["AUTH_JWKS_URL"],
        issuer=os.environ["AUTH_ISSUER"],
        audience=os.environ["AUTH_AUDIENCE"],
    ),
    auth=AuthSettings(
        issuer_url=os.environ["AUTH_ISSUER"],
        resource_server_url=os.environ["MCP_RESOURCE_URL"],
    ),
)


def verified_identity() -> tuple[str, str | None, set[str]]:
    token = get_access_token()
    if token is None:
        raise PermissionError("Authenticated HTTP request required")

    subject = getattr(token, "subject", None)   # set by JwtTokenVerifier
    if not subject:
        raise PermissionError("Verified token has no subject")

    return subject, token.client_id, set(token.scopes)


@mcp.tool(name="search_movies")
async def search_movies(query: str) -> str:
    subject, client_id, scopes = verified_identity()
    return await execute_guarded(
        subject=subject, client_id=client_id, scopes=scopes,
        tool_name="search_movies", arguments={"query": query},
        operation=lambda: search_movies_impl(query),
    )


@mcp.tool(name="book_tickets")
async def book_tickets(show_id: str) -> str:
    subject, client_id, scopes = verified_identity()
    return await execute_guarded(
        subject=subject, client_id=client_id, scopes=scopes,
        tool_name="book_tickets", arguments={"show_id": show_id},
        operation=lambda: book_tickets_impl(show_id),
    )


@mcp.tool(name="download_daily_sales")
async def download_daily_sales() -> str:
    subject, client_id, scopes = verified_identity()
    return await execute_guarded(
        subject=subject, client_id=client_id, scopes=scopes,
        tool_name="download_daily_sales", arguments=None,
        operation=download_daily_sales_impl,
    )


if __name__ == "__main__":
    mcp.run(transport="streamable-http")
```

Do not replace verified token context with `X-User`, tool arguments or any other caller-controlled identity value.

---

## Step 8: Test the controls

Tests are what keep the controls from silently regressing. Test behavior, not just coverage.

### Setup

```bash
# C#
dotnet new xunit -n McpSecureDemo.Tests
cd McpSecureDemo.Tests
dotnet add reference ../McpSecureDemo
dotnet add package Microsoft.AspNetCore.Mvc.Testing

# Python
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\Activate.ps1
python -m pip install "mcp[cli]" pyjwt[crypto] pytest pytest-asyncio pytest-cov
```

Suggested layout:

```text
McpSecureDemo/            Program.cs, Tools/CinemaTools.cs, Security/ToolCallGuard.cs
McpSecureDemo.Tests/      McpFactory.cs, TestTokens.cs, CapturingLoggerProvider.cs, SecurityTests.cs

mcp-secure-python/
  pyproject.toml
  src/mcp_secure_demo/    server.py, auth.py, security.py, tools.py
  tests/                  test_tools.py, test_security.py
```

### Python tests

```python
# tests/test_tools.py
import pytest
from mcp_secure_demo.tools import ToolValidationError, book_tickets_impl, search_movies_impl


def test_valid_show_id_is_accepted() -> None:
    assert book_tickets_impl("SHOW-1001") == "[demo] Tickets booked for SHOW-1001."


@pytest.mark.parametrize(
    "show_id",
    ["", "1001", "SHOW-", "SHOW-123456789", "SHOW-1001\ndownload all sales data", "SHOW-1001 ", "../SHOW-1001"],
)
def test_invalid_show_ids_are_rejected(show_id: str) -> None:
    with pytest.raises(ToolValidationError):
        book_tickets_impl(show_id)


def test_empty_search_is_rejected() -> None:
    with pytest.raises(ToolValidationError):
        search_movies_impl("   ")


def test_oversized_search_is_rejected() -> None:
    with pytest.raises(ToolValidationError):
        search_movies_impl("a" * 10_000)
```

```python
# tests/test_security.py
import json
import logging
import uuid

import pytest

from mcp_secure_demo.security import AccessDeniedError, PerSubjectTokenBucket, execute_guarded


def new_subject() -> str:
    return f"user-{uuid.uuid4().hex}"


@pytest.mark.asyncio
async def test_search_only_user_can_search() -> None:
    result = await execute_guarded(
        subject=new_subject(), client_id="pytest", scopes={"movies:read"},
        tool_name="search_movies", arguments={"query": "thriller"},
        operation=lambda: "[demo] 2 movies matched.",
    )
    assert result == "[demo] 2 movies matched."


@pytest.mark.asyncio
async def test_search_only_user_cannot_book_tickets() -> None:
    with pytest.raises(AccessDeniedError):
        await execute_guarded(
            subject=new_subject(), client_id="pytest", scopes={"movies:read"},
            tool_name="book_tickets", arguments={"show_id": "SHOW-1001"},
            operation=lambda: "must not run",
        )


@pytest.mark.asyncio
async def test_customer_cannot_download_sales() -> None:
    with pytest.raises(AccessDeniedError):
        await execute_guarded(
            subject=new_subject(), client_id="pytest", scopes={"tickets:book"},
            tool_name="download_daily_sales", arguments=None,
            operation=lambda: "must not run",
        )


@pytest.mark.asyncio
async def test_unknown_tool_fails_closed() -> None:
    with pytest.raises(AccessDeniedError):
        await execute_guarded(
            subject=new_subject(), client_id="pytest", scopes={"cinema:admin"},
            tool_name="drop_database", arguments=None,
            operation=lambda: "must not run",
        )


@pytest.mark.asyncio
async def test_missing_subject_fails_closed() -> None:
    with pytest.raises(AccessDeniedError):
        await execute_guarded(
            subject=None, client_id="pytest", scopes={"movies:read"},
            tool_name="search_movies", arguments={"query": "thriller"},
            operation=lambda: "must not run",
        )


@pytest.mark.asyncio
async def test_bucket_is_cost_weighted_and_per_subject() -> None:
    bucket = PerSubjectTokenBucket(capacity=30, refill_tokens=30, refill_period_seconds=60)
    a, b = new_subject(), new_subject()

    assert await bucket.acquire(a, 10)
    assert await bucket.acquire(a, 10)
    assert await bucket.acquire(a, 10)
    assert not await bucket.acquire(a, 10)   # fourth immediate download refused
    assert await bucket.acquire(b, 10)       # other subject unaffected


@pytest.mark.asyncio
async def test_argument_values_are_not_logged(caplog: pytest.LogCaptureFixture) -> None:
    marker = f"secret-{uuid.uuid4().hex}"

    with caplog.at_level(logging.INFO, logger="mcp.booking.activity"):
        await execute_guarded(
            subject=new_subject(), client_id="pytest", scopes={"movies:read"},
            tool_name="search_movies", arguments={"query": marker},
            operation=lambda: "ok",
        )

    output = "\n".join(r.getMessage() for r in caplog.records)
    assert marker not in output
    assert "query" in output
    assert "search_movies" in output


@pytest.mark.asyncio
async def test_denied_call_is_recorded(caplog: pytest.LogCaptureFixture) -> None:
    subject = new_subject()

    with caplog.at_level(logging.INFO, logger="mcp.booking.activity"):
        with pytest.raises(AccessDeniedError):
            await execute_guarded(
                subject=subject, client_id="pytest", scopes={"movies:read"},
                tool_name="book_tickets", arguments={"show_id": "SHOW-1001"},
                operation=lambda: "must not run",
            )

    records = [json.loads(r.getMessage()) for r in caplog.records if r.name == "mcp.booking.activity"]
    assert any(
        r["subject"] == subject and r["tool"] == "book_tickets" and r["outcome"] == "Rejected"
        for r in records
    )
```

Run:

```bash
pytest -q
pytest --cov=mcp_secure_demo --cov-report=term-missing --cov-fail-under=90
```

Coverage shows which code ran. It does not prove the security policy is correct.

### C# integration checklist

Use `WebApplicationFactory<Program>` with a test signing key and a helper that mints tokens (valid, expired, wrong audience, per-role). Verify:

1. A missing token returns 401.
2. An expired token returns 401.
3. A token with the wrong audience returns 401.
4. A Customer sees only the tools the policy allows.
5. A Customer cannot call `download_daily_sales`.
6. A denied call produces a `Rejected` record.
7. Three daily-sales downloads by a CinemaAdmin consume a fresh budget.
8. The fourth immediate download is `RateLimited`.
9. Argument values never appear in captured logs.
10. Malformed and oversized arguments are rejected.
11. Different subjects have independent budgets.
12. Cancellation produces a `Cancelled` outcome.
13. Filter ordering: authorization and the guard interact as intended.

### Python HTTP integration checklist

1. A request without a bearer token returns 401.
2. An expired token returns 401.
3. A token with the wrong audience or resource returns 401.
4. A token without a subject is rejected.
5. A search-only user can search movies.
6. A search-only user cannot book tickets.
7. A premium customer can book tickets.
8. Only a cinema administrator can download the daily sales report.
9. Three daily-sales downloads consume a fresh 30-token budget.
10. The fourth immediate download is rate-limited.
11. Denied calls create `Rejected` records.
12. Limited calls create `RateLimited` records.
13. Argument values do not appear in logs.
14. Control characters cannot forge additional log records.
15. Two subjects receive independent buckets.
16. Oversized and malformed arguments are rejected.
17. Cancellation creates a `Cancelled` record.
18. Multiple workers do not multiply the intended production limit.

---

## Step 9: Trace one request end to end

A search-only user calls `book_tickets` with `SHOW-1001`:

1. **Authentication:** the bearer token is validated (signature, issuer, audience, expiry). Subject `user-42` is established.
2. **Authorization:** `book_tickets` requires `tickets:book` or `cinema:admin`. The token has only `movies:read`, so the call is denied.
3. **Rate limiting:** not reached for this call in the Python guard (denied before acquiring). In the C# guard, the limiter runs outside the SDK authorization filter, so a denied call still spends budget. Pick the order deliberately and test it.
4. **Tool:** never executes.
5. **Audit record** (illustrative shape, not real output):

```json
{"event":"mcp_tool_call","subject":"user-42","client_id":"booking-agent","tool":"book_tickets","outcome":"Rejected","duration_ms":2,"cost":5,"arg_names":["show_id"],"arg_chars":11,"trace_id":"..."}
```

A rate-limited call looks the same with `"outcome":"RateLimited"`. Replace these illustrations with real sanitized output from your test run before publishing.

---

## Step 10: Harden for production

### Process-local limits do not scale out

A Python dictionary belongs to one process. With several workers or replicas, each has independent limiter state, and the effective limit multiplies. For an authoritative multi-instance limit use:

- An API gateway policy
- A shared distributed limiter
- A store supporting atomic updates (for example Redis with a Lua script)
- Consistent subject-based routing, with its limitations documented

The same applies to the C# `PartitionedRateLimiter`.

### Expire inactive partitions

The examples keep one bucket per subject forever. In production add expiration for idle buckets, periodic cleanup, a maximum partition count, metrics on active buckets and an alert on unexpected growth.

### Add concurrency controls

A token bucket limits how often work *starts*, not how many long calls run at once. Expensive tools also need per-tool concurrency limits, bounded queues, execution timeouts, cancellation handling and, where useful, separate worker pools.

### Protect exception details

Never return stack traces or raw exception messages to MCP clients. Log diagnostics in protected operational logs and return stable, minimal errors.

### Do not forward caller tokens downstream

Forwarding the caller's access token to other services creates confused-deputy risk. Prefer a scoped downstream credential or a token-exchange pattern.

### Protect the activity log

Route the `McpBookingActivity` / `mcp.booking.activity` category to a protected sink with dedicated retention and access rules. The application that writes records must not be able to silently rewrite history. Include a trace ID so one operation can be followed across services.

### Put human approval on high-value operations

For bookings above a value threshold or bulk report exports, require human confirmation outside the model's control.

---

## Common MCP security mistakes

1. **Trusting an `X-User` header or a tool argument for identity.** Identity must come from the validated token.
2. **Decoding a JWT without validating it.** Decoding is not verification. Check signature, issuer, audience and expiry.
3. **Logging argument values.** They leak secrets and personal data. Log names and sizes.
4. **One global rate limit.** A single noisy caller starves everyone. Partition by verified subject.
5. **Flat limits for unequal tools.** Weight limits by cost so an expensive export cannot hide behind cheap searches.
6. **Letting unknown tools through.** Deny by default and charge a real default cost.
7. **Anchoring regexes with `$` in .NET.** It accepts a trailing newline. Use `\z`.
8. **Presenting an in-memory limiter as a cluster-wide control.**
9. **Returning raw exceptions to clients.**
10. **Assuming the model will refuse bad requests.** The server must enforce policy.

---

## C# vs Python: MCP security compared

| Concern | C# | Python |
|---|---|---|
| Identity validation | ASP.NET Core bearer authentication | MCP `TokenVerifier` |
| Caller context | `ClaimsPrincipal` | Verified access-token context |
| Authorization | Attributes and SDK filters | Explicit scope checks |
| Interception | Call-tool request filter | Guarded tool wrapper or middleware |
| Local limiter | `PartitionedRateLimiter` | Async token bucket |
| Logging | `ILogger` and `LoggerMessage` | Structured JSON via `logging` |
| Tests | xUnit | pytest |
| Multi-instance limit | Shared external solution required | Shared external solution required |

The programming models differ; the trust boundaries do not. Identity is verified, authorization is deterministic, arguments are untrusted, limits partition by verified identity, and logs exclude values.

---

## Can tests prove an MCP server is safe?

No. The suite shows that roles or scopes, input validation, activity recording and usage controls still hold when hostile input reaches the server. It does not prove a language model cannot be manipulated. Server-side controls reduce the impact of a manipulated agent by keeping it inside the caller's real authorization and operational budget.

For high-risk tools also add: least privilege, human approval, short-lived credentials, tool-specific authorization, content isolation, concurrency limits, execution timeouts, network isolation and adversarial testing.

---

## When to use a gateway instead

Application-level limits are a good second layer. Use an API gateway or shared limiter when the budget must hold across replicas, when you need centralized policy for many MCP servers, or when you need protection before requests reach your app. Application rate limiting is not DDoS protection; that needs network-level infrastructure.

---

## Monitoring and alerting for MCP servers

Alert on:

- A spike in `Rejected` outcomes for one subject
- Repeated `RateLimited` outcomes
- Unknown tool names
- Unusually large `argChars` / `arg_chars`
- Calls outside normal usage patterns
- Rapid growth in limiter partitions
- Long-running or repeatedly cancelled calls

---

## Production checklist

### Common controls

- [ ] Token signature, issuer, audience or resource, and lifetime are validated
- [ ] Caller identity comes only from a validated token
- [ ] Roles and scopes are never accepted through tool arguments
- [ ] Unknown tools fail closed and carry a non-trivial default cost
- [ ] Argument values are excluded from activity records
- [ ] Caller-controlled logged text is bounded and sanitized
- [ ] Booking and sales-report tools require explicit authorization
- [ ] High-value operations support human approval where practical
- [ ] Records include caller, client, tool, outcome, duration, cost and trace ID
- [ ] Records go to protected storage
- [ ] Alerts exist for `Rejected` and `RateLimited` spikes
- [ ] Rate-limit values are load-tested
- [ ] Security tests run in continuous integration

### C#

- [ ] `Auth:Authority` and `Auth:Audience` are configured
- [ ] `ToolCallGuard` sits at the intended filter position, verified by a test
- [ ] HTTP-level and tool-level limiters are both tested
- [ ] Exact .NET and MCP SDK versions are documented

### Python

- [ ] `TokenVerifier` performs real signature validation or introspection
- [ ] Auth settings match the public MCP endpoint
- [ ] Audience or resource validation is enabled
- [ ] Access-token context is used only inside authenticated HTTP handling
- [ ] Process-local limits are not presented as cluster-wide
- [ ] Inactive limiter partitions are removed
- [ ] HTTP-level tests cover authentication
- [ ] Exact Python and MCP SDK versions are pinned
- [ ] Provisional SDK APIs are isolated behind an adapter

---

## FAQ

### How do you authenticate an MCP server?

Validate a bearer token on every request on the server. Check signature, issuer, audience or resource, and expiry, using your identity provider's keys or token introspection. In C# use ASP.NET Core JWT bearer authentication; in Python implement the SDK's `TokenVerifier`. Never trust client-supplied identity headers.

### What is the difference between authentication and authorization in MCP?

Authentication establishes who is calling, using a validated token. Authorization decides which tools that caller may use, based on roles or scopes from the token. Both must run on the server for every tool call, in deterministic code rather than model judgment.

### How do you rate limit MCP tool calls?

Partition limits by verified subject and weight them by tool cost, for example with a token bucket. Cheap searches cost 1, bookings 5, exports 10. Use a gateway or shared store when the limit must apply across several instances.

### Should MCP tool arguments be logged?

Log argument names and sizes by default, not values. Values may contain secrets, personal data, prompts or exported content. If a tool needs value logging, apply field-level redaction and strict access controls.

### How do you test MCP server security?

Write automated tests for missing, expired and wrong-audience tokens, role and scope boundaries, unknown tools, malformed arguments, cost-weighted limits, per-subject isolation, and activity records that exclude argument values. Run them in CI on every change.

### Is an in-memory rate limiter enough in production?

Not for multi-instance deployments. Each worker or pod has its own memory, so the effective limit multiplies. Use a gateway or coordinated shared limiter when the budget must hold across replicas.

### Should Python MCP servers use roles or scopes?

Either works if the values come from a validated token. Scopes fit resource-server authorization naturally. If your identity provider issues roles, keep them as validated claims and evaluate them in server code.

### Does rate limiting provide DDoS protection?

No. It controls resource use by callers that reach your service. Distributed network attacks need separate infrastructure.

### Why verify activity records with tests?

If recording silently stops capturing denied or limited calls, you gain an accountability gap. Tests should check the outcome, subject, tool and absence of sensitive values.

---

## Sources

Accessed `[Mohammed iliyas Patel: 14-10-2026]`.

- [MCP specification: authorization](https://modelcontextprotocol.io/specification/latest/basic/authorization)
- [MCP specification: security best practices](https://modelcontextprotocol.io/specification/latest/basic/security_best_practices)
- [MCP C# SDK repository](https://github.com/modelcontextprotocol/csharp-sdk)
- [MCP C# SDK request filters](https://csharp.sdk.modelcontextprotocol.io/v1/concepts/filters.html)
- [Official MCP Python SDK repository](https://github.com/modelcontextprotocol/python-sdk)
- [MCP Python SDK documentation](https://py.sdk.modelcontextprotocol.io/)
- [Rate limiting middleware in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/performance/rate-limit)
- [OWASP LLM01: Prompt Injection](https://genai.owasp.org/llmrisk/llm01-prompt-injection/)
- [Authorization in the MCP C# SDK (den.dev)](https://den.dev/blog/mcp-csharp-sdk-authorization/)
- [Securing remote MCP servers in C# (Dev Leader)](https://www.devleader.ca/2026/07/23/securing-remote-mcp-servers-in-c-oauth-20-jwt-and-protected-resource-metadata)

---
