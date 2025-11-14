* Configure VS Code Debug: press `Ctrl+Shift+D`  to create  `.vscode\launch.json`
  ```json
  {
        // Use IntelliSense to learn about possible attributes.
        // Hover to view descriptions of existing attributes.
        // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
        "version": "0.2.0",
        "configurations": [
        {
            "name": "GO-Launch Main",
            "type": "go",
            "request": "launch",
            "mode": "debug",
            "program": "${workspaceFolder}/link/cmd/linkd", 
            "cwd": "${workspaceFolder}",
            "env": {},
            "args": []
        }
        ]
  }
  ```
  * `${workspaceFolder}`: `09-composition-patterns\05-context-value-propagation-pattern`
  * Install Delve (Go Debugger)
    ```bash
    go install github.com/go-delve/delve/cmd/dlv@latest
    ```
* In `VSCode` press `Ctrl+Shift+P` ,and then type `Go: Locate Configured Go Tools`
* Clean old compiling cache
  ```bash
  go clean -cache -modcache -testcache -fuzzcache
  ```
* Clean old pkg
  ```bash
  rmdir /s /q "%GOROOT%\pkg"
  ```   

 
# 9 Composition patterns  
## 9.5 Context value propagation pattern
Besides our logger middleware, the handlers and downstream code can log. But matching log entries to their originating requests is difficult without a shared identifier, making troubleshooting guesswork. We can solve this issue by adding a trace ID to log entries:
```bash
. . .error="unreachable database instance: . . ." trace_id=42
. . .path=/r/go status=500. . .trace_id=42
```
We can simplify troubleshooting by linking critical log entries when issues arise. Without an ID, we could face scattered logs that are frustrating to piece together. As figure 9.5 shows, we’ll add new middleware in the new `traceid` package to relay these IDs with `Context`. This middleware wraps downstream handlers, including other middleware, to relay trace IDs, allowing each to access trace IDs when needed.


> [!NOTE]
> `Context` can carry any type of value across package APIs.

![architecture diagram](pics/CH09_F05_Gumus.drawio.svg)  
Figure 9.5 Injecting and propagating trace IDs through `Request.Context`

In the figure, `traceid.Middleware` receives and clones a `*Request`, attaching a unique trace ID to the cloned `*Request`’s `Context`. Then it relays this `*Request` downstream. As long as we pass the same `Context`, every part of our program can access the same trace ID.

In the following sections, we’ll learn to add values to `Context` and set up middleware for trace IDs. These skills are key for real-world projects and also improve our grasp of Go’s type system. Finally, we’ll implement a custom `slog.Handler` to automatically extract and log trace IDs from a provided `Context`.

### 9.5.1 Generating, storing, and retrieving
Now that we know how to relay trace IDs, let’s see how to store them inside a `Context`. A `Context` can store a single key-value pair. To add a value to a `Context`, we use
```golang
context.WithValue(parent Context, key any, value any) Context
```
To retrieve a value, we use the Value method:
```golang
Context.Value(key any) (value any)
```
Because `Context` is like a tree, adding a value returns a new `Context`, preserving values from its parents. Value searches for the key in the chain, not only in the specified `Context`, allowing us to access values in that one and its parents.

As the next listing shows, the `traceid` package generates trace IDs and attaches and retrieves them from a `Context`. With `traceid`, we consistently handle trace IDs.
- [Listing 9.10: Package `traceid`](../../all-listings/09-composition-patterns/10-package-traceid.md)

In this code, `New` returns a naive trace ID using timestamps for demonstration. To achieve stronger uniqueness, consider using universally unique identifiers (UUIDs). We skip this step to avoid downloading a package now, but you can do so if you like. See https://go.dev/play/p/8yxDxIvU7iG for example code for generating UUIDs using the https://pkg.go.dev/github.com/google/uuid package.

`WithContext` stores the trace ID in the `Context`, and `FromContext` retrieves it using a type assertion (`.(string)`) because `Context.Value` returns an `any` type. This assertion returns the trace ID as a string value and a `true` for success, allowing us to extract safely.

The names of our functions are idiomatic. Because the `traceid` package already has the name, we omit the `TraceID` prefix from our functions. If our package had a different name, we would use the `TraceIDWithContext` and `TraceIDFromContext` function names for clarity.

Both functions use the same unexported key type, `traceIDContextKey`, to store and retrieve values from a Context. We keep the key unexported so other packages can’t access it, eliminating key collisions between packages. Even if another package declares a key with the same `traceIDContextKey` name, these keys never conflict because each key type is scoped to its own package. Ours, for example, is scoped as `traceid.traceIDContextKey`.

> [!NOTE]DEFINITION 
> A key collision occurs when multiple packages use the same key type in a shared `Context` to write/read values, unintentionally overwriting or reading one another’s values.

Both our functions use empty structs (`traceIDContextKey{}`) as keys each time they store or retrieve a trace ID from a Context. Empty structs never allocate memory, and every new empty struct of the same type (i.e., `traceIDContextKey`) is always equal. This equality ensures both functions consistently access the same trace ID values from the `Context`.

### 9.5.2 Middleware
As listing 9.11 shows, now that we can save and retrieve trace IDs from `Context`, we’ll implement the trace ID middleware. This middleware clones the `Request` with a new `Context` with a unique trace ID and passes this cloned `Request` along the downstream handler chain, allowing them to extract the same `Context` when needed.
- [Listing 9.11: `traceid` middleware](../../all-listings/09-composition-patterns/11-traceid-middleware.md)

We call `FromContext` to check whether a trace ID exists in the `Context`. If it does, we skip adding one; other middleware may have already provided it. If not, we use `New` to get a trace ID and store it in a `Context` using `WithContext`. Then we clone the `Request` with this `Context`. Finally, we pass that `Request` to the handler, ensuring that the trace ID is available downstream.

In this section, we learned how to store and propagate request-scoped values using `Context`. Although useful, `Context` stores values as the type-unsafe empty interface (any), hiding the underlying types of these values. Misusing `Context` can lead to subtle bugs that are hard to debug. We use `Context` here for trace IDs because these values are peripheral: they inform but don’t dictate how our program works if they’re missing. Avoid relying on `Context` values to control your program logic and flow for maintainable, reliable programs.

### 9.5.3 Wrapping slog handlers
Our traceid.Middleware can inject trace IDs into `Request.Context` for each incoming request. As figure 9.6 shows, we’ll now implement a `slog.Handler` to add them to the logs automatically on every logging method call, such as `LogAttrs`. Otherwise, we would have to extract these trace IDs from `Context` and pass them every time we call logging methods.

![architecture diagram](pics/CH09_F06_Gumus.drawio.svg)  
Figure 9.6 Automatically adding attributes to log records by wrapping the original log handler with a custom one that extracts trace IDs from the `Context` we pass to logging methods, such as `LogAttrs`

`slog.Logger` provides an API for logging but delegates the actual work, such as formatting log records, to a type such as `slog.TextHandler`, which satisfies the `slog.Handler` interface. This `Logger`, for example, uses a `slog.TextHandler` as a `Handler`:
```golang
lg := slog.New(slog.NewTextHandler(os.Stderr, nil)) #1
```
#1 NewTextHandler returns *slog.TextHandler, a Handler implementation.

This one uses our `LogHandler` that wraps the previous `TextHandler`:
```golang
lg = slog.New(traceid.NewLogHandler(lg.Handler())) #1
lg.LogAttrs(ctx, . . .) #2
```
#1 lg is a *slog.Logger that uses our LogHandler as a Handler.   
#2 Calls the LogHandler.Handle method, which adds a trace_id to the log record before forwarding it to TextHandler.Handle   

Calling `lg.LogAttrs` automatically outputs a trace ID if the `ctx` has a trace ID:
```bash
. . .error="unreachable database instance: . . ." trace_id=42
```
`LogHandler` attaches a trace ID `slog.Attr` to a `slog.Record` and calls the `TextHandler.Handle` method, which renders a log output with the trace ID.


### 9.5.4 Implementing a slog.Handler
Now that we know we can output extra log attributes by wrapping `slog.Handlers`, let’s add our own. As listing 9.12 shows, we’ll add a `Handler` that wraps an existing one. It extracts a trace ID from the provided `Context` and injects the trace ID into the slog.Record. This way, extra log attributes, such as a trace ID, will appear alongside existing ones during logging.

- [Listing 9.12: `LogHandler`](../../all-listings/09-composition-patterns/12-loghandler.md)

Our `LogHandler` type wraps an existing `Handler`. The `Handle` method extracts the trace ID from the provided `Context`. If a trace ID exists, it clones the `Record` and adds the trace ID as a new `slog.Attr` using the configured log key (e.g., “`trace_id`”), as in this example:
```golang
lg.LogAttrs(ctx, . . .) #1
```
#1 Assumes that this Context carries a trace ID value

Output:
```bash
. . .trace_id=2889906900
```
Calling `LogAttrs` eventually calls `LogHandler.Handle`, passing it a `Context` (assuming that the `lg` is a `Logger` that uses `LogHandler` as its `Handler` and `Context` has a trace ID).

### 9.5.5 Implementing WithAttrs and WithGroup
We can add trace IDs to log records automatically, but we have one remaining issue to fix. slog.Logger provides methods to add permanently or group log attributes for convenience, such as WithAttrs and WithGroup. Using these methods return a new Logger with the original Handler we wrapped, not our LogHandler. As a result, calling LogAttrs logs without trace IDs looks like the following:
```golang
lg = lg.With("ver", "1.0.0").WithGroup("request") #1
lg.LogAttrs(. . .)
```
#1 Derives a Logger without LogHandler, although the original Handler was LogHandler

Output:
```bash
. . .ver=1.0.0 request.error="something failed" #1
```
#1 Loses the trace ID log attribute

As the next listing shows, we’ll implement two more `Handler` interface methods: `WithAttrs` and `WithGroup`. They return a new LogHandler, preserving our trace ID log attribute.
- [Listing 9.13: `WithAttrs` and `WithGroup`](../../all-listings/09-composition-patterns/13-withattrs-and-withgroup.md)

We’ve returned a new `LogHandler` from each method, wrapping the original Handler to preserve the `LogHandler` behavior. Each returned Handler will log with trace IDs.

`LogHandler` integrates with existing `slog` handlers without altering their core logic. It’s like middleware for log handlers. Combining it with the trace ID middleware allows us to generate, propagate, and log trace IDs consistently throughout our program.

> [!TIP]DEEP DIVE: PERFORMANCE CONCERNS
> Our L`ogHandler.Handle` clones the `slog.Record` on each `slog.Logger` method call, allocating a new slice underneath. This extra allocation usually doesn’t affect most servers (like ours) because I/O tends to dominate, but it can slow down others. For more details on performance, see the `slog` documentation at https://pkg.go.dev/log/slog and https://mng.bz/X71v.

### 9.5.6 Integration
Now that we have a `traceid` package, middleware to inject trace IDs into a `Request`’s `Context`, and a `LogHandler` to enrich logs with trace IDs, let’s see how everything fits into our server. We’ll create a new logger and wrap our previous logger middleware as follows.
- [Listing 9.14: Integrating trace IDs](../../all-listings/09-composition-patterns/14-integrating-trace-ids.md)

We wrap the existing `Logger`’s `Handler` with our `LogHandler` to add trace IDs to logs. We ensure that the new `Logger` is passed to the rest of our code so that each can automatically log with trace IDs. Then we wrap the existing logger middleware with the trace ID middleware to inject trace IDs into each `Request.Context`. Now the downstream handlers automatically log with trace IDs as they use `LogHandler`.

When the following HTTP requests come, for example
```bash
$ curl -i localhost:8080/health
```
we see
```bash
. . .path=/health method=GET. . .trace_id=1747066003015604000
```

We wrapped the `slog.Handler` interface with our `LogHandler` type to add trace ID log attributes from `Context` values. Our code is reusable and self-contained, and we can test and replace it independently, which improves maintainability. Our approach reduces complexity and is consistent with Go’s principles of composition with simple independent parts.
