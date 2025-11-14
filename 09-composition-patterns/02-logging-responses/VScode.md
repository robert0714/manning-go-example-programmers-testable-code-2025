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
  * `${workspaceFolder}`: `09-composition-patterns\02-logging-responses`
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
## 9.2 Logging responses
Suppose that we want to log handler-response durations to see whether the handlers respond slowly. Unlike request details, such as URL paths and HTTP methods, response details, such as durations, aren’t readily available, so we must calculate them, as shown in figure 9.3.

![architecture diagram](pics/CH09_F03_Gumus.drawio.svg)  
Figure 9.3 RecordResponse uses the Duration middleware to calculate handler durations.

Our `Middleware` calls `RecordResponse` , which wraps the handler using the `Duration` middleware. `Duration` measures how long the handler takes to respond. After the handler finishes serving the request, `Duration` saves the response duration in a `Response` value. Then `RecordResponse` returns this `Response` to `Middleware`, which logs the duration.

We separate concerns by giving each piece one duty. `Middleware` handles logging, `RecordResponse` orchestrates middleware that measures response metrics, `Duration` measures durations, and `Response` stores response metrics. Keeping each piece independent allows us to add or remove the pieces easily, composing new functionality without complicating the setup. In section 9.3.2, we’ll add new middleware to capture HTTP status codes.

### 9.2.1 Measuring durations
As listing 9.3 shows, we start with the Duration middleware, which measures how long it takes a handler to serve a request. The middleware wraps a handler, records the start time, and updates the provided duration variable when the handler finishes serving the request.

- [Listing 9.3: `Duration` middleware](../../all-listings/09-composition-patterns/03-duration-middleware.md)

The `Duration` middleware is a building block. It measures durations without assuming where we store its result. Instead of saving the measured duration directly in a `Response`, for example, `Duration` saves the duration via a pointer. This approach makes `Duration` reusable. The returned closure captures this pointer, however, so it’s critical that the provided pointer points to a new variable for each request. Otherwise, we may introduce race conditions. Section 9.2.2 shows how to pass new variables to `Duration` safely using `RecordResponse`.

> [!TIP]
>  Avoid unnecessary coupling between types to improve composability.

Using `defer` here is idiomatic because it ties the end measurement directly to the handler’s completion. The deferred function updates the duration right before returning, ensuring that the measured result is always accurate. Using a `defer` statement is a typical, concise Go pattern for cleanup and finalization tasks.

### 9.2.2 Response recording
We added the `Duration` middleware to measure how long handlers take to process requests, but that’s not enough. Next, we plan to capture HTTP status codes, so we want a structured way to store multiple response metrics together and extend later without many code changes.

Listing 9.4 introduces the `Response` struct for holding response details and a helper, `RecordResponse`. This helper chains multiple pieces of middleware, including `Duration` and `StatusCode` (section 9.3.2), runs the wrapped handler, and returns the collected metrics.
- [Listing 9.4: Response recording](../../all-listings/09-composition-patterns/04-response-recording.md)

We implemented `Response` and `RecordResponse` to capture and store response metrics. We also demonstrated the middleware chaining pattern:
* Each middleware takes a handler, wraps it, and returns a new handler.
* Each middleware runs sequentially around the wrapped handler, allowing each middleware to store its metrics in a Response before and after the handler.
  
Here’s how the process works. `RecordResponse` creates a shared `Response` and passes its fields (by pointer) to response-handling middleware, like `Duration`, which updates the fields via the fields’ pointers. (We currently have only one piece of middleware but will add more.)

These pieces of middleware must wrap around the provided handler, h, so they execute before the wrapped handler to measure response details accurately. So we wrap the handler starting from the last middleware in the slice. This way, the first middleware serves the incoming request first, followed by subsequent middleware and finally the wrapped handler itself. When processing finishes, `RecordResponse` returns the populated `Response`.

> [!TIP]
> `slices.Backward` returns an iterator that iterates over a slice in reverse.

`RecordResponse` is a helper that encapsulates middleware chaining, providing a Response with a single function call. Meanwhile, middleware like `Duration` is a building block that callers can compose individually, allowing fine-grained control of metrics collection.

`Duration` is decoupled and chainable with other middleware that handles response metrics. There’s no perfect approach; however—only a tradeoff. The tradeoff is that `Duration` updates a variable through a pointer, which may lead to race conditions if multiple concurrent requests share a variable. Users must ensure that they pass a separate variable to `Duration` for each request, exactly as our `RecordResponse` does. That’s what we documented in listing 9.3, reminding users of this important consideration.

### 9.2.3 Integration
Our logger middleware logs only request details, like paths and methods. As the next listing shows, we’re improving our middleware using `RecordResponse` to log response details.
- [Listing 9.5: Integrating response recording](../../all-listings/09-composition-patterns/05-integrating-response-recording.md)

We’ve integrated `RecordResponse` to capture, store, and log durations. We’ve improved our middleware without complicating the logging, keeping each piece composable and simple.

`RecordResponse` wraps the next handler with our `Duration` middleware, measures how long it takes the handler to process each request, and returns this metric within a `Response`. The logger middleware uses the `slog.Duration` function to log the captured response duration. When the following HTTP request comes
```bash
$ curl -i localhost:8080/health
```
we see the following server logs:
```bash
level=INFO msg=request app=linkd path=/health method=GET duration=6.208µs
```
The flow of this incoming requests looks like this:
```bash
Request -> . . . -> Middleware -> Duration -> ServeMux -> Health Handler
```
In this section, we’ve progressively improved our middleware to capture and log response metrics. We started by creating small functions, like `Duration`, that handle a single task independently. Then we added composable building blocks, such as `Response` and `RecordResponse`, providing easy extendibility without tightly coupling the parts. Finally, our integration unified these separate pieces. This approach embodies the philosophy of composition: preferring simple, independent parts over complex, coupled ones, enabling users to extend or adapt functionality as needed. Next, we’ll apply another compositional pattern to capture HTTP status codes by embedding `ResponseWriter` into another type.