# .NET 10 / C# Anti-Pattern Checklist

Reference for the `dotnet-code-review` skill. Each entry includes: what to look for, why it matters, the idiomatic fix, and a MS Docs link where available.

---

## 1. Async / Await

### `async void` outside event handlers — CRITICAL
`async void` methods cannot be awaited, so exceptions silently escape and crash ASP.NET Core.

```csharp
// Avoid: exceptions are unobserved and crash the process
public async void ProcessAsync() { await DoWorkAsync(); }

// Prefer: return Task so callers can await and exceptions propagate
public async Task ProcessAsync() { await DoWorkAsync(); }
```

ASP.NET Core controllers and minimal API handlers must return `Task` or `Task<IResult>`, never `void`.
> Ref: https://learn.microsoft.com/aspnet/core/fundamentals/best-practices

---

### `.Result` or `.Wait()` on a Task — CRITICAL
Blocks the thread and deadlocks when a synchronization context is present (ASP.NET, UI apps). Exceptions are also wrapped in `AggregateException`.

```csharp
// Avoid: deadlocks under sync context; wraps exceptions
var data = GetDataAsync().Result;
someTask.Wait();

// Prefer: await properly
var data = await GetDataAsync();
await someTask;
```

| Blocking (avoid)     | Async equivalent       |
|----------------------|------------------------|
| `Task.Wait()`        | `await task`           |
| `Task.Result`        | `await task`           |
| `Task.WaitAll()`     | `await Task.WhenAll()` |
| `Task.WaitAny()`     | `await Task.WhenAny()` |
| `Thread.Sleep(n)`    | `await Task.Delay(n)`  |

> Ref: https://learn.microsoft.com/dotnet/csharp/asynchronous-programming/async-scenarios#review-considerations-for-asynchronous-programming

---

### `async` method with no `await` — WARNING
Compiles but generates a useless state machine. Return the Task directly, or remove `async`.

```csharp
// Avoid: unnecessary state machine overhead
public async Task<string> GetNameAsync() => "hello";

// Prefer: return directly without async keyword
public Task<string> GetNameAsync() => Task.FromResult("hello");
```

---

### Missing `Async` suffix — SUGGESTION
Convention: all Task-returning methods end in `Async`. Exceptions: interface implementations where the base name is fixed, or event handlers.

```csharp
// Avoid: ambiguous — is this sync or async?
public async Task Process() { ... }

// Prefer: clear intent
public async Task ProcessAsync() { ... }
```

---

### Async lambdas in LINQ — WARNING
LINQ uses deferred execution. Async lambdas inside `Where`/`Select` are evaluated at iteration time, making ordering and error behaviour unpredictable.

```csharp
// Avoid: async predicate in LINQ — exceptions are lost, ordering is undefined
var results = items.Where(async x => await IsValidAsync(x));

// Prefer: materialise first, then process async
var validItems = await Task.WhenAll(
    items.Select(async x => new { x, valid = await IsValidAsync(x) }));
var results = validItems.Where(r => r.valid).Select(r => r.x);
```

> Ref: https://learn.microsoft.com/dotnet/csharp/asynchronous-programming/async-scenarios#use-caution-with-asynchronous-lambdas-in-linq

---

## 2. CancellationToken

### Token not forwarded to downstream calls — WARNING
Accepting `CancellationToken` but passing `CancellationToken.None` (or nothing) to async calls defeats graceful shutdown.

```csharp
// Avoid: stoppingToken is accepted but never forwarded
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    var result = await _http.GetStringAsync(url); // ignores cancellation
}

// Prefer: forward the token
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    var result = await _http.GetStringAsync(url, stoppingToken);
}
```

---

### No cancellation check inside long-running loop body — WARNING
Checking only at the loop condition means shutdown waits for an entire iteration to complete.

```csharp
// Avoid: shutdown blocked until all items in current iteration are processed
while (!stoppingToken.IsCancellationRequested)
{
    foreach (var item in largeList)
    {
        ProcessItem(item);
    }
}

// Prefer: check inside the loop too
while (!stoppingToken.IsCancellationRequested)
{
    foreach (var item in largeList)
    {
        stoppingToken.ThrowIfCancellationRequested();
        ProcessItem(item);
    }
}
```

---

### Long synchronous work before first `await` in `ExecuteAsync` — WARNING
`BackgroundService.ExecuteAsync` blocks the host startup sequence until its first `await`. Synchronous work before the first `await` delays the entire application from starting.

```csharp
// Avoid: host startup is blocked until this completes
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    Thread.Sleep(5000);
    await DoRealWorkAsync(stoppingToken);
}

// Prefer: yield immediately, then do setup work
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    await Task.Yield(); // releases startup thread immediately
    InitialiseExpensiveThing();
    await DoRealWorkAsync(stoppingToken);
}
```

> Ref: https://learn.microsoft.com/aspnet/core/fundamentals/host/hosted-services#ihostedservice-interface

---

## 3. Dependency Injection

### Service locator pattern — WARNING
Calling `GetService<T>()` or `GetRequiredService<T>()` at runtime hides dependencies and makes testing harder.

```csharp
// Avoid: hidden dependency, hard to test
public class MyService(IServiceProvider provider)
{
    public void Do() => provider.GetRequiredService<IFoo>().Work();
}

// Prefer: inject IFoo directly
public class MyService(IFoo foo)
{
    public void Do() => foo.Work();
}
```

The only valid use of `IServiceScopeFactory`/`IServiceProvider` at runtime is when a singleton needs to resolve scoped services per-operation.
> Ref: https://learn.microsoft.com/aspnet/core/fundamentals/dependency-injection#recommendations

---

### Captive dependency (scoped inside singleton) — CRITICAL
A singleton that captures a scoped service holds it alive for the application lifetime, causing stale state and concurrency bugs.

```csharp
// Avoid: IDbContext is scoped but captured by a singleton
builder.Services.AddSingleton<IMyService, MyService>(); // MyService takes IDbContext
builder.Services.AddScoped<IDbContext, AppDbContext>();

// Prefer: create a scope per operation using IServiceScopeFactory
public class MyService(IServiceScopeFactory scopeFactory) : IMyService
{
    public async Task DoWorkAsync()
    {
        await using var scope = scopeFactory.CreateAsyncScope();
        var db = scope.ServiceProvider.GetRequiredService<IDbContext>();
    }
}
```

> Ref: https://learn.microsoft.com/dotnet/core/extensions/dependency-injection/guidelines#example-anti-patterns

---

### Transient `IDisposable` resolved from root container — CRITICAL
Transient `IDisposable` services resolved from the root container are retained until app shutdown — a memory leak.

```csharp
// Avoid: resolved from root, never disposed until app exits
var svc = app.Services.GetRequiredService<ITransientDisposable>();

// Prefer: resolve within a scope so disposal is tied to scope lifetime
using var scope = app.Services.CreateScope();
var svc = scope.ServiceProvider.GetRequiredService<ITransientDisposable>();
```

---

## 4. Null Safety

### `First()` / `Single()` on potentially empty sequences — WARNING
Throws `InvalidOperationException` when the sequence is empty. Use the `OrDefault` variants and null-check the result.

```csharp
// Avoid: throws if sequence is empty
var story = stories.First(s => s.Id == id);

// Prefer: graceful null handling
var story = stories.FirstOrDefault(s => s.Id == id);
if (story is null) return Results.NotFound();
```

---

### Missing null-conditional access — WARNING
Dereferencing a nullable reference without `?.` or a prior null check risks `NullReferenceException`.

```csharp
// Avoid: NullReferenceException if Url is null
var host = new Uri(story.Url).Host;

// Prefer: guard first
var host = story.Url is not null ? new Uri(story.Url).Host : null;
```

---

## 5. Exception Handling

### Swallowed exceptions — CRITICAL
An empty `catch` or catch-and-ignore silently loses errors. In background services this causes the service to appear running while doing nothing.

```csharp
// Avoid: error is silently lost
try { await DoWorkAsync(token); }
catch (Exception) { }

// Prefer: at minimum log it; typically re-throw
try { await DoWorkAsync(token); }
catch (Exception ex) when (ex is not OperationCanceledException)
{
    _logger.LogError(ex, "Work failed");
    throw;
}
```

Note: In .NET 6+, an unhandled exception in `BackgroundService.ExecuteAsync` logs the error and stops the host by default. Filter `OperationCanceledException` to avoid treating graceful shutdown as an error.
> Ref: https://learn.microsoft.com/dotnet/core/compatibility/core-libraries/6.0/hosting-exception-handling

---

### Catching `Exception` too broadly — WARNING
Overly broad catches mask unexpected errors. Be specific, or filter with `when`.

```csharp
// Avoid: catches OutOfMemoryException, StackOverflowException, etc.
catch (Exception ex) { _logger.LogWarning(ex, "Retrying"); Retry(); }

// Prefer: catch only what you can actually handle
catch (HttpRequestException ex) { _logger.LogWarning(ex, "HTTP failed, retrying"); Retry(); }
```

---

## 6. LINQ

### `.Count() > 0` instead of `.Any()` — SUGGESTION
`Any()` short-circuits on the first element. `Count()` enumerates the whole sequence.

```csharp
// Avoid: enumerates everything
if (stories.Count() > 0) { ... }

// Prefer: short-circuits on first match
if (stories.Any()) { ... }
```

---

### Unnecessary `ToList()` / `ToArray()` mid-chain — SUGGESTION
Materialising mid-chain forces eager evaluation before further filtering, wasting memory.

```csharp
// Avoid: materialises all items, then filters
var result = store.GetAll().ToList().Where(s => s.Score > 10);

// Prefer: filter first, materialise once at the end if needed
var result = store.GetAll().Where(s => s.Score > 10).ToList();
```

---

### Side effects inside LINQ predicates — WARNING
LINQ uses deferred execution. Predicates with side effects run at iteration time and may run multiple times if the query is re-enumerated.

```csharp
// Avoid: counter incremented on every enumeration, not once
var filtered = stories.Where(s => { _counter++; return s.Score > 10; });

// Prefer: separate the side effect from the query
var filtered = stories.Where(s => s.Score > 10).ToList();
_counter += filtered.Count;
```

---

## 7. Options Pattern

### `IConfiguration` string indexer in new code — SUGGESTION
Raw string keys are fragile, typo-prone, and don't benefit from validation. Prefer the Options pattern.

```csharp
// Avoid: magic string, no compile-time safety
var url = configuration["SqsOptions:QueueUrl"];

// Prefer: strongly typed, validated, testable
public class MyService(IOptions<SqsOptions> opts)
{
    private readonly SqsOptions _opts = opts.Value;
}
```

---

### `IOptions<T>` where live reload is needed — SUGGESTION
`IOptions<T>` is a snapshot taken at startup. For settings that may change at runtime, prefer `IOptionsMonitor<T>` (singleton-safe) or `IOptionsSnapshot<T>` (scoped).

| Interface              | Lifetime  | Reflects config changes? |
|------------------------|-----------|--------------------------|
| `IOptions<T>`          | Singleton | No                       |
| `IOptionsSnapshot<T>`  | Scoped    | Yes (per request)        |
| `IOptionsMonitor<T>`   | Singleton | Yes (live)               |

---

## 8. C# Idioms (.NET 10)

### Verbose null checks — SUGGESTION
Prefer pattern matching over `== null` / `!= null`.

```csharp
// Avoid: older style
if (story == null) return;
if (story != null) Process(story);

// Prefer: idiomatic C# 9+
if (story is null) return;
if (story is not null) Process(story);
```

---

### Not using primary constructors — SUGGESTION
C# 12 primary constructors eliminate boilerplate for classes that simply store injected dependencies.

```csharp
// Avoid: boilerplate constructor
public class StoryService : IStoryService
{
    private readonly ILogger<StoryService> _logger;
    private readonly StoryStore _store;

    public StoryService(ILogger<StoryService> logger, StoryStore store)
    {
        _logger = logger;
        _store = store;
    }
}

// Prefer: primary constructor (matches existing project style)
public class StoryService(ILogger<StoryService> logger, StoryStore store) : IStoryService
{
    // use logger and store directly, or assign to private fields if needed
}
```

---

### `record` for data-only types — SUGGESTION
DTOs and message shapes compared by value should be `record` or `record struct`.

```csharp
// Avoid: class with manual equality
public class HackerNewsStory
{
    public int Id { get; init; }
    public string Title { get; init; }
}

// Prefer: record — value equality, ToString, and deconstruct for free
public record HackerNewsStory(int Id, string Title, string? Url, int Score, long Time);
```

---

## 9. Thread Safety

### Choosing the right synchronisation primitive — CRITICAL

The correct primitive depends on whether your critical section contains `await` calls.

**`lock` — for synchronous critical sections only**

`lock` is efficient and idiomatic, but the C# compiler prevents `await` inside a `lock` block. Use it when the guarded code is purely synchronous.

```csharp
// Correct use of lock: no awaits inside the critical section
private readonly Lock _lock = new();

public void Add(int id)
{
    lock (_lock) { _ids.Add(id); }
}
```

**`SemaphoreSlim(1, 1)` — for async critical sections**

When the critical section itself needs to `await`, use `SemaphoreSlim` with `WaitAsync()`. Always release in a `finally` block to guarantee the semaphore is returned even on exception.

```csharp
// Avoid: cannot await inside lock (CS1996 compiler error)
lock (_lock) { await SaveAsync(); }

// Prefer: SemaphoreSlim for async mutual exclusion
private readonly SemaphoreSlim _semaphore = new(1, 1);

public async Task SaveAsync(CancellationToken ct = default)
{
    await _semaphore.WaitAsync(ct);
    try
    {
        await _repository.SaveAsync(ct);
    }
    finally
    {
        _semaphore.Release();
    }
}
```

> Ref: https://learn.microsoft.com/dotnet/standard/threading/semaphoreslim

---

### Mutable shared state in a singleton without synchronisation — CRITICAL
If a singleton modifies a shared field from multiple threads (background + request threads), it needs `lock`, `SemaphoreSlim`, `Interlocked`, or a concurrent collection.

```csharp
// Avoid: race condition — HashSet is not thread-safe
public class StoryStore
{
    private readonly HashSet<int> _ids = [];
    public void Add(int id) => _ids.Add(id);
}

// Prefer: lock for sync access, SemaphoreSlim if Add ever needs to await
private readonly Lock _lock = new();
public void Add(int id) { lock (_lock) { _ids.Add(id); } }
```

> Ref: https://learn.microsoft.com/dotnet/core/extensions/dependency-injection/guidelines#thread-safety

---

### `List<T>` or `Dictionary<K,V>` shared across threads — CRITICAL
These collections have no thread-safety guarantees. Use `ConcurrentDictionary<K,V>`, `ConcurrentQueue<T>`, or protect access with `lock` / `SemaphoreSlim`.

---

## 10. Minimal API (ASP.NET Core)

### `HttpContext` captured outside the request — CRITICAL
`HttpContext` is only valid during the active request. Storing it in a field or passing it to a background thread causes crashes.

```csharp
// Avoid: context accessed after request ends
app.MapGet("/", async (HttpContext ctx) =>
{
    _ = Task.Run(() => ctx.Response.WriteAsync("late")); // crashes
});

// Prefer: extract values before background work begins
app.MapGet("/", (HttpRequest req) =>
{
    var value = req.Query["x"].ToString();
    _ = Task.Run(() => DoWork(value));
    return Results.Accepted();
});
```

> Ref: https://learn.microsoft.com/aspnet/core/fundamentals/best-practices

---

### Missing error responses in minimal API endpoints — SUGGESTION
When an operation can fail, return an appropriate `Results.*` response rather than throwing or returning null.

```csharp
// Avoid: returns null body if story not found (200 with null)
app.MapGet("/api/stories/{id:int}", (int id, StoryStore store) =>
    Results.Ok(store.GetById(id)));

// Prefer: explicit 404
app.MapGet("/api/stories/{id:int}", (int id, StoryStore store) =>
    store.TryGet(id, out var story) ? Results.Ok(story) : Results.NotFound());
```

---

## 11. MVC Controllers (ControllerBase / Controller)

### Missing `[ApiController]` on API controllers — WARNING
Without `[ApiController]`, invalid model state silently reaches your action body. The attribute adds automatic 400 responses on validation failure, enforces attribute routing, and improves binding source inference.

```csharp
// Avoid: missing attribute means manual ModelState checks in every action
public class StoriesController : ControllerBase
{
    [HttpPost]
    public IActionResult Create(CreateStoryRequest request)
    {
        if (!ModelState.IsValid) return BadRequest(ModelState);
        // ...
    }
}

// Prefer: [ApiController] validates automatically before the action runs
[ApiController]
[Route("api/[controller]")]
public class StoriesController : ControllerBase
{
    [HttpPost]
    public IActionResult Create(CreateStoryRequest request)
    {
        // invalid input never reaches here — framework returns 400 automatically
    }
}
```

> Ref: https://learn.microsoft.com/dotnet/architecture/modern-web-apps-azure/develop-asp-net-core-mvc-apps#mapping-requests-to-responses

---

### Business logic inside a controller action — WARNING
Controllers are a UI-layer abstraction responsible for validating requests and choosing responses. Business rules, data access, and domain logic belong in services, not action bodies.

```csharp
// Avoid: controller doing too much — hard to test, violates SRP
[HttpGet("{id}")]
public async Task<IActionResult> GetStory(int id)
{
    var story = await _dbContext.Stories.FindAsync(id);
    if (story is null) return NotFound();
    story.ViewCount++;
    await _dbContext.SaveChangesAsync();
    return Ok(new StoryDto { Id = story.Id, Title = story.Title });
}

// Prefer: delegate to a service; controller only coordinates
[HttpGet("{id}")]
public async Task<IActionResult> GetStory(int id)
{
    var dto = await _storyService.GetStoryAsync(id);
    return dto is null ? NotFound() : Ok(dto);
}
```

> Ref: https://learn.microsoft.com/aspnet/core/mvc/controllers/actions#defining-actions

---

### Using `Controller` base class for API-only controllers — SUGGESTION
`Controller` adds view-related helpers (`View()`, `ViewBag`, etc.) that are unnecessary for a pure API. Inherit from `ControllerBase` directly.

```csharp
// Avoid: pulls in view-specific helpers an API will never use
public class StoriesController : Controller { ... }

// Prefer: lean base class for APIs
public class StoriesController : ControllerBase { ... }
```

---

### Not checking `ModelState.IsValid` without `[ApiController]` — CRITICAL
On controllers without `[ApiController]`, skipping the `ModelState.IsValid` check allows invalid input to silently reach business logic.

```csharp
// Avoid: invalid model reaches the service layer
[HttpPost]
public IActionResult Create(CreateRequest request)
{
    _service.Create(request); // request.Name could be null despite [Required]
    return Ok();
}

// Prefer: guard explicitly when [ApiController] is absent
[HttpPost]
public IActionResult Create(CreateRequest request)
{
    if (!ModelState.IsValid) return BadRequest(ModelState);
    _service.Create(request);
    return Ok();
}
```

> Ref: https://learn.microsoft.com/aspnet/core/mvc/models/validation#model-state

---

### Returning raw objects instead of typed `ActionResult<T>` — SUGGESTION
Returning a plain object always produces a 200. Prefer `ActionResult<T>` so the action can return different status codes from the same method.

```csharp
// Avoid: always 200, even when the result is logically absent
[HttpGet("{id}")]
public async Task<StoryDto?> GetStory(int id) =>
    await _storyService.GetStoryAsync(id);

// Prefer: explicit status codes
[HttpGet("{id}")]
public async Task<ActionResult<StoryDto>> GetStory(int id)
{
    var dto = await _storyService.GetStoryAsync(id);
    return dto is null ? NotFound() : Ok(dto);
}
```

---

### Using conventional routing for REST APIs — SUGGESTION
Conventional routing is designed for MVC apps with views. REST APIs should use attribute routing so each endpoint's URL contract is explicit and co-located with the action.

```csharp
// Avoid: URL shape is defined globally, disconnected from the action
app.MapControllerRoute("default", "{controller=Home}/{action=Index}/{id?}");

// Prefer: attribute routing keeps URL and action together
[ApiController]
[Route("api/[controller]")]
public class StoriesController : ControllerBase
{
    [HttpGet("{id:int}")]
    public IActionResult Get(int id) { ... }
}
```

> Ref: https://learn.microsoft.com/aspnet/core/mvc/controllers/routing#conventional-routing
