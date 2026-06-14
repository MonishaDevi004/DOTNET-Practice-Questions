# .NET Core / C# Interview Questions & Answers (6–15 Years Experience)

Covers: Collections & Generics, Delegates, Events, LINQ, Web API, Performance & Optimization.

---

## Collections & Generics

### 1. Walk through how `Dictionary<TKey,TValue>` works internally.

`Dictionary<TKey,TValue>` is backed by two arrays: a `bucket` array (storing indices into the entries array) and an `entries` array (storing the actual key/value pairs plus a hash code and a "next" link for collision chaining).

- On `Add`/lookup, the key's `GetHashCode()` is computed, then reduced modulo the bucket count to find the bucket index.
- If the bucket is empty, the entry is placed there. If occupied (collision), the new entry is added to the front of a linked chain via the `next` field — this is **separate chaining**, not open addressing.
- When the entries array fills up (load factor reached), the dictionary **resizes** (typically doubling, then picking the next prime in older implementations / power-of-two in newer ones) and rehashes all entries into new buckets.
- A good `GetHashCode()` (well-distributed, avoids clustering) is critical — a poor hash function degrades lookups from O(1) to O(n) due to long collision chains.

**Follow-up to probe:** ask about `EqualityComparer<T>.Default`, and why overriding `Equals` without `GetHashCode` breaks dictionaries.

---

### 2. When would you choose `IEnumerable<T>`, `ICollection<T>`, `IList<T>`, or `IQueryable<T>` as a method's parameter/return type?

- **`IEnumerable<T>`** — use when the caller only needs to iterate (forward-only, possibly lazy/deferred). Signals "read-only, streaming" semantics. Best default for return types from LINQ-producing methods.
- **`ICollection<T>`** — adds `Count`, `Add`, `Remove`, `Contains`. Use when the caller needs to know size or mutate membership but doesn't need indexed access or ordering guarantees.
- **`IList<T>`** — adds indexer access (`this[int]`), insertion at position. Use when order and positional access matter (e.g., binding to a UI grid).
- **`IQueryable<T>`** — represents an **expression tree** that hasn't been executed yet, typically against a remote data source (EF Core, etc.). Use when you want the underlying provider (e.g., SQL) to do filtering/projection rather than pulling everything into memory.

**Key trap:** returning `IQueryable<T>` from a repository/service layer leaks persistence concerns and can cause "leaky abstractions" where callers accidentally compose queries the data layer never intended to support. Returning `IEnumerable<T>` from a method that's actually `IQueryable<T>` underneath can also cause **silent client-side evaluation** — a major performance pitfall.

---

### 3. Explain covariance and contravariance (`in`/`out`) in generic interfaces.

- **Covariance (`out`)**: allows a generic interface typed for a derived type to be used where the base type is expected. `IEnumerable<out T>` means `IEnumerable<Derived>` can be assigned to `IEnumerable<Base>`. Safe because `T` only appears in **output** positions (return values).
- **Contravariance (`in`)**: the reverse — `Action<in T>` means `Action<Base>` can be assigned to `Action<Derived>`, because the delegate only **consumes** `T` as input.

```csharp
IEnumerable<string> strings = new List<string>();
IEnumerable<object> objects = strings; // OK — covariant

Action<object> objAction = o => Console.WriteLine(o);
Action<string> stringAction = objAction; // OK — contravariant
```

Trying to make `IList<T>` covariant would fail to compile because `T` appears in both input (`Add(T item)`) and output (`T this[int index]`) positions — covariance there would be unsafe (could let you `Add` an `object` into a `List<string>`).

---

### 4. `List<T>` vs `ConcurrentBag<T>` vs `ConcurrentDictionary<TKey,TValue>` — when is each the wrong choice?

- **`List<T>`**: not thread-safe at all. Concurrent reads are fine if nothing mutates, but any concurrent write (even just `Add`) while another thread reads/writes can corrupt internal state or throw `InvalidOperationException` (collection modified during enumeration).
- **`ConcurrentBag<T>`**: optimized for scenarios where the **same thread** that adds an item is likely to remove it (thread-local storage with work-stealing). It's a poor choice for FIFO/ordered processing or for producer/consumer where producers and consumers are different thread pools — overhead from work-stealing can be worse than a simple `ConcurrentQueue<T>` with a lock.
- **`ConcurrentDictionary<TKey,TValue>`**: great for high-read/moderate-write key-value caches, but `GetOrAdd`/`AddOrUpdate` with a value-factory delegate can execute the factory **multiple times** under contention (only one result is kept) — a common bug if the factory has side effects (e.g., increments a counter or opens a connection).

**Better defaults**: for most queue-like workloads, `ConcurrentQueue<T>` or `Channel<T>` (from `System.Threading.Channels`) is usually the right modern choice over `ConcurrentBag<T>`.

---

### 5. How do generic constraints affect what the compiler/runtime generates?

```csharp
public T Create<T>() where T : class, IComparable<T>, new()
{
    var instance = new T();
    return instance;
}
```

- `where T : class` restricts to reference types, enabling `null` checks and reference equality.
- `where T : new()` requires a public parameterless constructor — the compiler emits a call to `Activator.CreateInstance<T>()` under the hood (or a direct `newobj` if it can be resolved at JIT time for value types).
- Interface constraints (`IComparable<T>`) let you call interface members directly on `T` without casting, and the JIT can often devirtualize these calls for value types via generic specialization.

A repository pattern often uses:

```csharp
public interface IEntity { int Id { get; } }

public class Repository<T> where T : class, IEntity, new()
{
    public T GetById(int id) { /* ... */ }
}
```

This constrains `T` so the repository can rely on `Id` existing and can construct new instances generically.

---

### 6. Why is generic code over value types (e.g., `List<int>`) more efficient than over reference types?

For **value types**, the JIT generates a **separate specialized implementation** per concrete value type at runtime (e.g., a distinct compiled method body for `List<int>` vs `List<double>`), avoiding boxing — the `int` is stored inline in the array, not as a boxed `object` on the heap.

For **reference types**, the JIT shares **one** compiled implementation across all reference type instantiations (e.g., `List<string>` and `List<MyClass>` share code), because all reference types are just pointers of the same size — this reduces code bloat ("code explosion") at the cost of an extra level of indirection.

This is why `List<int>` is dramatically faster and more memory-efficient than `ArrayList` (which boxes every `int` into an `object`) or `List<object>`.

---

## Delegates

### 7. `Func<T>`, `Action<T>`, `Predicate<T>` vs custom delegates — when to define your own?

- `Func<T1,...,TResult>` — returns a value.
- `Action<T1,...>` — returns `void`.
- `Predicate<T>` — `Func<T, bool>`, used mainly by collection methods (`List<T>.Find`, `RemoveAll`).

Define a **custom named delegate** when:
1. The signature is reused widely and a descriptive name improves readability (`public delegate void OrderProcessedHandler(Order order, DateTime processedAt);`).
2. You need `ref`/`out`/`in` parameters — `Func`/`Action` don't support these.
3. You're designing a public API and want compile-time clarity / IntelliSense over generic `Func<int,int,bool>` noise.

---

### 8. How does a multicast delegate behave when invoked — especially return values and exceptions?

A multicast delegate maintains an **invocation list**. When invoked:

- Each subscriber is called **sequentially, in registration order**.
- For delegates with a **return value**, only the **return value of the last invoked delegate** is returned via the normal `Invoke()` call — intermediate return values are discarded (you'd need `GetInvocationList()` and call each manually to capture all of them).
- If **any subscriber throws**, the exception propagates immediately and **remaining subscribers are not invoked** — this is a common production bug (e.g., one bad event handler prevents others from running).

```csharp
Func<int> a = () => 1;
Func<int> b = () => 2;
Func<int> combined = a;
combined += b;
int result = combined(); // result == 2 (only last one's return value)

foreach (Func<int> f in combined.GetInvocationList())
{
    Console.WriteLine(f()); // 1, then 2
}
```

---

### 9. What's the performance cost of delegate invocation vs a direct call, and when does it matter?

A delegate call involves an extra **indirect call** (through the delegate's `Invoke` method pointer) compared to a direct/static call, plus the delegate object itself is a heap allocation (unless cached). The JIT generally **cannot inline** through a delegate the way it can a direct method call.

In practice:
- For most application code (handling HTTP requests, business logic), the overhead is **negligible** — microseconds at most, dwarfed by I/O.
- It matters in **hot loops** processing millions of items (e.g., custom LINQ-like pipelines, serializers, game loops) where delegate dispatch in a tight loop can show up in profiling. In those cases, consider caching delegate instances (avoid recreating lambdas that capture variables inside loops — each closure allocation adds GC pressure) or using generic structs/static methods instead.

---

### 10. How do delegates relate to closures, and what's the classic loop-variable capture bug?

A **closure** is created when a lambda/anonymous method references a variable from its enclosing scope; the compiler generates a hidden class to hold that captured variable by **reference**, not by value.

Classic pre-C#5 bug:

```csharp
var actions = new List<Action>();
for (int i = 0; i < 3; i++)
{
    actions.Add(() => Console.WriteLine(i));
}
foreach (var a in actions) a(); // prints 3, 3, 3 (pre-C# 5 'for' captured by reference)
```

Since **C# 5**, `foreach` loop variables get a fresh copy per iteration, so `foreach` doesn't have this issue — but a `for` loop with a shared `i` still captures the **same variable**, and all closures see its final value. Fix: copy into a local inside the loop body (`int local = i;`) before capturing.

---

## Events

### 11. Why does C# favor `event` over a public delegate field?

A public delegate field (`public Action OnSomething;`) allows **any external code** to:
- Assign (`=`) it, wiping out all other subscribers — accidental or malicious.
- Invoke it directly from outside the declaring class.

`event` restricts external code to only `+=`/`-=` (subscribe/unsubscribe) via compiler-generated `add`/`remove` accessors; only the declaring class can `Invoke()` it. This enforces the **publisher-subscriber encapsulation** — the publisher controls when/if the event fires, and no subscriber can clobber others' subscriptions.

---

### 12. How do events cause memory leaks, and how do you fix it?

If object `A` subscribes to an event on long-lived object `B` (`B.SomeEvent += A.Handler`), `B` now holds a reference to `A` through its invocation list. If `A` is supposed to be garbage-collected (e.g., a short-lived view/page) but never unsubscribes, **`B` keeps `A` alive indefinitely** — a classic "lapsed listener" leak, especially common with static/singleton publishers and UI elements (WPF, WinForms) or long-lived services subscribing to short-lived ones.

Fixes:
1. **Explicitly unsubscribe** (`B.SomeEvent -= A.Handler`) in `Dispose`/`OnUnloaded`/finalizer-equivalent lifecycle hooks.
2. Use the **Weak Event Pattern** (e.g., `WeakEventManager` in WPF, or implement `IWeakEventListener`), where the publisher holds a `WeakReference` to subscribers so they can still be collected.
3. Prefer `IObservable<T>`/`IDisposable` subscriptions (Rx.NET) where unsubscription is enforced via `Dispose()`.

---

### 13. Walk through the `EventHandler<TEventArgs>` pattern and when to create a custom `EventArgs`.

Standard pattern:

```csharp
public class OrderShippedEventArgs : EventArgs
{
    public int OrderId { get; }
    public DateTime ShippedAt { get; }
    public OrderShippedEventArgs(int orderId, DateTime shippedAt)
    {
        OrderId = orderId;
        ShippedAt = shippedAt;
    }
}

public class OrderService
{
    public event EventHandler<OrderShippedEventArgs>? OrderShipped;

    protected virtual void OnOrderShipped(OrderShippedEventArgs e) =>
        OrderShipped?.Invoke(this, e);
}
```

- `sender` is always `object` (or `object?`) — the convention lets subscribers know "who raised this" without coupling the signature to a specific type.
- Create a custom `EventArgs` subclass whenever the event needs to carry data beyond what `EventArgs.Empty` provides — naming it `<EventName>EventArgs` is the convention.
- The `?.Invoke(...)` null-conditional pattern avoids a race where the delegate becomes `null` between the null-check and invocation (though it's not a complete fix under heavy concurrency — see next question).

---

### 14. How do you make event subscription/invocation thread-safe?

The `event` add/remove accessors generated by the compiler are **not inherently thread-safe** for the underlying delegate field unless you explicitly synchronize (older compilers generated a `lock`-based implementation for non-`virtual` events in some cases, but this isn't guaranteed and was removed for performance in later versions).

The bigger issue is the **invoke** side: `MyEvent?.Invoke(this, args)` reads the field once into a local copy as part of `?.`, so it's safe from a "null after the check" race in that specific expression — but if a subscriber unsubscribes **during** invocation of a multicast delegate, the in-flight invocation list (captured at the time `Invoke` was called) is unaffected because delegates are **immutable** — `+=`/`-=` create a *new* delegate instance rather than mutating the existing one. So the actual invocation list snapshot is safe.

What you still need to guard:
- If multiple threads can call `+=`/`-=` concurrently, wrap with a `lock` around the field, or use `Interlocked.CompareExchange` in a loop for lock-free add/remove.
- Handler code itself must be thread-safe if multiple threads can trigger the event concurrently.

---

## LINQ

### 15. Explain deferred execution vs immediate execution, with an example of how it can bite you.

**Deferred** operators (`Where`, `Select`, `OrderBy`, `Take`, etc.) build an **iterator/expression** but don't run until enumerated (via `foreach`, `.ToList()`, `.Count()`, etc.). **Immediate** operators (`ToList()`, `ToArray()`, `Count()`, `Sum()`, `First()`, etc.) execute right away.

```csharp
var query = orders.Where(o => o.Total > threshold); // not executed yet
threshold = 1000; // changes the result!
var result = query.ToList(); // executes NOW, using threshold = 1000
```

Common bugs:
- Capturing a variable that changes before enumeration, producing unexpected results.
- Enumerating a deferred query **multiple times** (e.g., once for `.Any()` and again for `foreach`) — each enumeration **re-executes** the whole pipeline, which for `IQueryable` means **multiple round trips to the database**.

---

### 16. `IEnumerable<T>` vs `IQueryable<T>` — where does the "filtering" actually happen?

- `IEnumerable<T>` LINQ extension methods compile lambda arguments into **delegates** (compiled IL) — execution happens **in-process**, item by item, via `MoveNext()`.
- `IQueryable<T>` LINQ extension methods compile lambda arguments into **`Expression<TDelegate>` trees** — these are **data**, not code. A provider (e.g., EF Core) **translates** the expression tree into another language (SQL) and executes it remotely.

The dangerous pitfall: if you call a method on `IQueryable<T>` that the provider **cannot translate** (e.g., a custom C# method, complex string formatting), EF Core may either throw, or in older/permissive configurations, silently **pull the entire table into memory** and filter client-side — a severe, often invisible performance regression. Always check generated SQL (e.g., via logging) when in doubt.

---

### 17. What's the "multiple enumeration" and "N+1 query" problem, and how do you avoid them?

**Multiple enumeration**: enumerating the same `IEnumerable<T>` (especially one backed by a deferred LINQ query or a non-cached source like a file/db reader) multiple times re-runs the whole pipeline each time.

```csharp
var query = dbContext.Orders.Where(o => o.CustomerId == id);
if (query.Any())               // 1st DB round trip
{
    var list = query.ToList(); // 2nd DB round trip — same filter re-run
}
```
Fix: materialize once with `.ToList()`/`.ToArray()` before branching/reusing.

**N+1 query problem**: loading a parent collection, then lazily loading a related collection **per item** in a loop:

```csharp
var customers = dbContext.Customers.ToList();      // 1 query
foreach (var c in customers)
    var orders = c.Orders.ToList();                // N queries (lazy loading)
```
Fix: eager-load with `.Include(c => c.Orders)`, or project with `.Select()` to shape exactly what's needed in a single query.

---

### 18. How does EF Core translate a LINQ query into SQL — what's the role of `Expression<Func<T,bool>>`?

When you write `dbSet.Where(x => x.Age > 18)`, the C# compiler — because `dbSet` is `IQueryable<T>` — compiles the lambda into an **expression tree** (`Expression<Func<T,bool>>`) rather than a delegate. This expression tree is essentially an AST: a `BinaryExpression` node (`>`) with a `MemberExpression` (`x.Age`) and a `ConstantExpression` (`18`).

EF Core's query provider walks this tree, maps `MemberExpression` nodes to column names via its metadata model, and generates a `WHERE Age > 18` SQL clause. Because it's a tree (data), the provider can **inspect and rewrite** it — something impossible with a compiled delegate, which is opaque IL.

This is also why methods like `DateTime.Now`, custom extension methods, or complex string operations sometimes can't be translated — there's no SQL equivalent the provider knows how to generate from that expression node.

---

### 19. Explain `yield return` and how iterator blocks relate to LINQ's laziness.

`yield return` causes the compiler to generate a **state machine** (a hidden class implementing `IEnumerator<T>`/`IEnumerable<T>`) that suspends execution at each `yield` and resumes on the next `MoveNext()` call. This is the mechanism underlying LINQ's deferred execution — `Where`, `Select`, etc. are themselves implemented as iterator blocks.

```csharp
public static IEnumerable<int> Range(int start, int count)
{
    for (int i = 0; i < count; i++)
    {
        Console.WriteLine($"Yielding {start + i}");
        yield return start + i;
    }
}
```

Nothing inside the method body runs until the first `MoveNext()` — calling `Range(1, 5)` returns instantly without printing anything. Each element is produced **on demand**, enabling things like infinite sequences, streaming large files without loading them fully into memory, and combining multiple LINQ operators into a **single pass** over the data (operator fusion) rather than materializing intermediate collections.

---

## Web API (ASP.NET Core)

### 20. Describe the ASP.NET Core middleware pipeline and how request/response flow through it.

Middleware components are registered in `Program.cs` (`app.Use...()`) and form a chain. Each middleware receives a `HttpContext` and a reference to the **next** delegate in the chain (`RequestDelegate next`). It can:
- Do work **before** calling `await next(context)` (on the way "in").
- Do work **after** `await next(context)` returns (on the way "out").
- **Short-circuit** by not calling `next` at all (e.g., authentication failure returning 401 immediately).

```csharp
app.Use(async (context, next) =>
{
    // before
    await next(context);
    // after — response is being built
});

app.UseRouting();
app.UseAuthentication();
app.UseAuthorization();
app.UseEndpoints(endpoints => endpoints.MapControllers());
```

**Order matters critically** — e.g., `UseAuthentication`/`UseAuthorization` must come before `UseEndpoints`/`MapControllers`, and `UseExceptionHandler`/`UseCors` typically come early so they wrap everything downstream.

---

### 21. Explain DI service lifetimes (`Singleton`, `Scoped`, `Transient`) and the "captive dependency" problem.

- **Transient**: a new instance every time it's requested.
- **Scoped**: one instance **per request** (in ASP.NET Core, the scope = the HTTP request).
- **Singleton**: one instance for the **entire application lifetime**.

**Captive dependency**: occurs when a `Singleton` service depends on a `Scoped` (or `Transient` holding scoped state, e.g., `DbContext`) service. Because the singleton is constructed once and holds a reference to that scoped instance forever, the "scoped" service is effectively **captured and reused across requests** — leading to stale data, threading issues (e.g., `DbContext` is not thread-safe and shared `DbContext` across concurrent requests throws), and memory growth.

The DI container (with validation enabled, e.g., `ValidateScopes = true` in development) throws an `InvalidOperationException` at startup if it detects this. The fix: inject `IServiceScopeFactory` into the singleton and create a new scope per use:

```csharp
public class MySingletonService
{
    private readonly IServiceScopeFactory _scopeFactory;
    public MySingletonService(IServiceScopeFactory scopeFactory) => _scopeFactory = scopeFactory;

    public async Task DoWorkAsync()
    {
        using var scope = _scopeFactory.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        // ...
    }
}
```

---

### 22. How do you write async controller actions correctly, and how do deadlocks happen with async code?

Controller actions should be `async Task<IActionResult>` and `await` all I/O-bound calls (DB, HTTP clients) — this frees the thread-pool thread to handle other requests while waiting.

**Deadlock scenario** (classic, mostly in non-ASP.NET-Core contexts with a `SynchronizationContext`, but the pattern matters conceptually): blocking on async code with `.Result` or `.Wait()`:

```csharp
public IActionResult Get()
{
    var data = GetDataAsync().Result; // BLOCKS the calling thread
    return Ok(data);
}
```

If `GetDataAsync()` internally does `await` without `ConfigureAwait(false)` and there's a captured `SynchronizationContext` (classic ASP.NET, WPF, WinForms — **not** ASP.NET Core, which has no `SynchronizationContext`), the continuation tries to resume on the same thread that's now blocked by `.Result` — deadlock.

In **ASP.NET Core** there's no `SynchronizationContext`, so this exact deadlock is less likely, but **sync-over-async** (`.Result`/`.Wait()`) is still harmful: it **ties up a thread-pool thread** while another thread-pool thread does the actual async work, reducing throughput under load and risking thread-pool starvation. Rule of thumb: **`async` all the way down**, never mix `.Result`/`.Wait()` with `async` methods.

---

### 23. What API versioning strategies exist, and what are the trade-offs?

1. **URL path versioning** (`/api/v1/orders`) — most explicit, cache-friendly, easy to route, but "pollutes" the URL and makes resource identity ambiguous across versions.
2. **Query string versioning** (`/api/orders?api-version=1.0`) — keeps URLs stable but easy to overlook/forget; caching proxies may not vary on query strings consistently.
3. **Header versioning** (custom header like `X-Api-Version` or `Accept` media-type versioning, e.g., `Accept: application/vnd.myapi.v2+json`) — keeps URLs clean (good for REST purists/HATEOAS) but harder to test manually (can't just click a link in a browser) and less visible in logs.

In practice, the **`Asp.Versioning`** package (successor to `Microsoft.AspNetCore.Mvc.Versioning`) supports all three via `ApiVersionReader` combinations. The pragmatic choice for most internal/B2B APIs is **URL path versioning** for discoverability; header-based versioning suits public APIs aiming for stricter REST semantics. Whatever the mechanism, a clear **deprecation policy** (sunset headers, documentation) matters more long-term than the mechanism itself.

---

### 24. Compare response caching, output caching, and distributed caching (Redis) in ASP.NET Core — when would you use each?

- **Response Caching** (`Microsoft.AspNetCore.ResponseCaching` / `Cache-Control` headers): instructs **clients/proxies** to cache responses; the server may still process the request unless an intermediary cache (CDN, reverse proxy) intercepts it. Good for public, cacheable GET endpoints (e.g., product catalogs) where slightly-stale data across users is acceptable.
- **Output Caching** (newer middleware, `.NET 7+`): caches the **entire response on the server**, so subsequent matching requests are served without re-executing the endpoint at all — configurable per-route policies (vary by query string, headers, expiration). More powerful and flexible than response caching for server-side cost reduction.
- **Distributed Cache (Redis via `IDistributedCache`)**: used for caching **application data** (not whole HTTP responses) across multiple server instances — e.g., caching a computed result, session state, or a lookup table that's expensive to query repeatedly. Essential in horizontally-scaled deployments where in-memory (`IMemoryCache`) caches would be inconsistent per instance.

A common layered approach: `IMemoryCache` for ultra-hot, small, per-instance data (with short TTLs to limit staleness across instances) + Redis as the shared layer + output caching for whole-response GET endpoints behind a reverse proxy/CDN for static or rarely-changing content.

---

## Performance & Optimization

### 25. Explain .NET's GC generations and Server GC vs Workstation GC.

The GC divides the heap into **generations**:
- **Gen 0**: newly allocated, short-lived objects. Collected frequently and cheaply.
- **Gen 1**: a buffer between short- and long-lived objects.
- **Gen 2**: long-lived objects (e.g., caches, static data). Collections here are more expensive (full heap scan in non-background mode).
- **Large Object Heap (LOH)**: objects ≥ 85,000 bytes go here, collected as part of Gen 2; LOH fragmentation is a known issue (mitigated since .NET Core via `GCSettings.LargeObjectHeapCompactionMode`).

**Workstation GC**: optimized for low-latency client apps; runs GC on the same thread that triggered it (or a dedicated GC thread with background GC for concurrent collection of Gen 2).

**Server GC**: creates a **heap and GC thread per core**, optimized for **throughput** on multi-core server workloads (ASP.NET Core defaults to Server GC). It uses more memory but parallelizes collections. For containerized workloads with tight memory limits, Server GC's higher memory footprint can cause issues — `GCHeapHardLimit`/`GCHeapHardLimitPercent` (or `<ServerGarbageCollection>false</ServerGarbageCollection>` for low-traffic services) are common tuning knobs.

---

### 26. How do `Span<T>` and `Memory<T>` reduce allocations, and give a practical example.

`Span<T>` is a **stack-only**, ref-struct type representing a contiguous, type-safe view over memory — array, stack-allocated memory, or unmanaged memory — **without copying**. `Memory<T>` is the heap-allocatable counterpart (usable in async methods, fields, etc., where `Span<T>` cannot be used due to its ref-struct restriction).

Classic example — parsing a CSV line without allocating substrings:

```csharp
ReadOnlySpan<char> line = "123,456,789";
int start = 0;
for (int i = 0; i <= line.Length; i++)
{
    if (i == line.Length || line[i] == ',')
    {
        ReadOnlySpan<char> token = line.Slice(start, i - start);
        int value = int.Parse(token); // no substring allocation
        start = i + 1;
    }
}
```

Without `Span<T>`, `string.Split(',')` allocates an array **plus** a new `string` per token. `Span<T>`-based parsing avoids all of that — significant in high-throughput parsers, serializers, or hot paths processing large text/binary payloads.

---

### 27. `Task` vs `ValueTask` — when does `ValueTask` actually help, and what's the risk?

`Task<T>` is a reference type — every async method that returns `Task<T>` allocates an object on the heap (unless cached, e.g., `Task.CompletedTask` / cached `Task.FromResult` for common values).

`ValueTask<T>` is a struct that can represent **either** a synchronously-available result (no allocation) **or** wrap an underlying `Task<T>` if it actually needs to go async. It helps when a method **frequently completes synchronously** (e.g., a cache-hit path) — avoiding millions of small `Task` allocations in hot paths (this is why `Stream.ReadAsync` returns `ValueTask<int>` in modern APIs).

**Risks / rules**:
- A `ValueTask` **must not be awaited multiple times**, and you generally **shouldn't store it** or call `.Result`/`.AsTask()` more than once — its underlying representation can be a pooled object that gets reused/invalidated.
- For most application-level async code (controllers, services calling DB/HTTP), `Task`/`Task<T>` is simpler and safer; `ValueTask` is a micro-optimization for library/hot-path code where allocation profiling has shown it matters.

---

### 28. What's the performance impact of boxing/unboxing, and where does it sneak in unexpectedly?

Boxing wraps a value type in a heap-allocated object (so it can be treated as `object`/an interface); unboxing copies it back to the stack. Each boxing operation is a **heap allocation** (GC pressure) plus a copy.

Sneaky places it happens:
- Passing a `struct` to a method expecting `object` — `Console.WriteLine(someInt)` doesn't box (has an `int` overload), but `string.Format("{0}", someInt)` **does** box `someInt` into `object[] args`.
- Storing value types in **non-generic** collections (`ArrayList`, `Hashtable`) — every `int` added is boxed.
- Using a `struct` through a **non-generic interface** reference, e.g., `IComparable c = myStruct;`.
- Enum values used as dictionary keys or compared via `object.Equals` can box.

Fix: use generic collections (`List<int>`), generic constraints/methods instead of `object`-based APIs, and prefer typed `string.Create`/interpolated string handlers (C# 10+ avoids boxing in many interpolation scenarios via `DefaultInterpolatedStringHandler`).

---

### 29. What EF Core/database-level changes give the biggest performance wins, and what do you check first?

1. **`AsNoTracking()`** for read-only queries — skips EF's change-tracking overhead (snapshotting, identity map), often a 2x+ speedup for large read result sets.
2. **Projection (`.Select(x => new Dto {...})`)** instead of loading full entities — reduces columns fetched and avoids materializing navigation properties you don't need.
3. **Avoid N+1** via `.Include()`/`.ThenInclude()` or split queries (`AsSplitQuery()`) when a single JOIN would cause a cartesian explosion across multiple collections.
4. **Indexes** matching your actual `WHERE`/`ORDER BY`/`JOIN` columns — check the execution plan; a missing index on a foreign key used in filters is one of the most common production slowdowns.
5. **Pagination** (`Skip`/`Take`) for large result sets — but be aware `OFFSET`-based paging degrades on large offsets; keyset/cursor pagination (`WHERE Id > lastId ORDER BY Id`) scales better.
6. **Compiled queries** (`EF.CompileAsyncQuery`) for very hot, repeatedly-executed queries — avoids re-building the expression-tree-to-SQL translation each call.
7. Check for **chatty round trips** — batching multiple operations into a single `SaveChanges()` rather than calling it per entity in a loop.

The first thing to check in any "slow API" investigation is usually: **what SQL is actually being generated**, and **how many times is it being called** (enable EF Core logging or use a tool like MiniProfiler).

---

### 30. What tools would you use to diagnose a performance regression in a running .NET Core service?

- **`dotnet-counters`**: live view of GC stats, ThreadPool queue length, exception rate, request rate — great first check for "is this GC pressure or thread starvation?"
- **`dotnet-trace`**: captures detailed ETW/EventPipe traces, viewable in PerfView or Visual Studio's profiler — useful for CPU hot-path analysis and async/await flow.
- **`dotnet-gcdump`**: captures a GC heap snapshot to find memory leaks / large object retention (open in VS or `dotnet-gcdump` analysis tools).
- **BenchmarkDotNet**: for micro-benchmarking specific methods/algorithms in isolation — essential before/after any "optimization" claim, since intuition about allocations and JIT behavior is often wrong.
- **Application Insights / OpenTelemetry traces**: for distributed tracing across services — identifies whether the slowdown is in your code, a downstream dependency (DB, external API), or network latency.
- **`dotnet-stack`**: quick thread dump to spot deadlocks or thread-pool starvation (many threads stuck in the same `await`/lock).

A senior candidate should describe a **methodology** (reproduce → measure with the right tool for the suspected bottleneck class → form a hypothesis → change one variable → re-measure) rather than jumping straight to guesses like "let's add caching" or "let's make it async."

---
