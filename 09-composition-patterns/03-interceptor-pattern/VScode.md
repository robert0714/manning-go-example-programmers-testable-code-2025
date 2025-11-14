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
  * `${workspaceFolder}`: `09-composition-patterns\03-interceptor-pattern`
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
## 9.3 Interceptor pattern
We logged response durations by wrapping handlers using the `Duration` middleware. Now suppose that we want to track HTTP status codes to spot errors or monitor successful responses. Unlike response durations, status codes aren’t easily accessible to middleware because handlers send them directly to clients by calling `ResponseWriter.WriteHeader`. We must find a way to intercept the `WriteHeader` method calls to capture status codes.

> [!NOTE] 
> Section 8.3 explains that `ResponseWriter` is an interface with a `WriteHeader` method that takes an integer and writes an HTTP status code to clients, such as `200` (OK).

As figure 9.4 shows, we’ll introduce an `Interceptor` type that wraps the handler’s original `ResponseWriter`. It intercepts the handler’s `WriteHeader` calls to capture status codes.


![architecture diagram](pics/CH09_F04_Gumus.drawio.svg)  
Figure 9.4 `Interceptor` captures HTTP status codes by intercepting `WriteHeader`. Then `Interceptor` forwards the call to the underlying `ResponseWriter`.

We have three requirements for our `Interceptor` type:
* It must be a `ResponseWriter` so we can pass it to handlers.
* It must intercept `WriteHeader` calls so we can capture status codes.
* After capturing the status code, it must forward the `WriteHeader` calls to the original `ResponseWriter` so handlers can respond status codes to clients normally.

We’ll meet these requirements by embedding `ResponseWriter` in our `Interceptor` and wrapping it around a handler’s original `ResponseWriter`. Embedding automatically forwards all `ResponseWriter` method calls to the embedded `ResponseWriter`, allowing handlers to respond as before. Because our `Interceptor` declares its own `WriteHeader` method, it can capture HTTP status codes transparently, without handlers being aware or needing modification.

First, we’ll declare the `Interceptor` and embed the `ResponseWriter` using an anonymous field. Then we’ll add StatusCode middleware to inject the `Interceptor` into handlers. Finally, we’ll update our `RecordResponse` with `StatusCode` and logger middleware to log status codes.

> [!TIP]
> Embedding is useful syntactic sugar. It doesn’t override methods but prioritizes the embedding type’s methods with the same name. In our case, the `Interceptor`’s `WriteHeader` takes precedence, letting us capture status codes.

### 9.3.1 Field embedding
Let’s start with the `Interceptor` type to capture HTTP status codes. We’ve learned that embedding forwards all method calls to the embedded type except those we add.

As listing 9.6 shows, we’re embedding `ResponseWriter` (anonymously, without a field name) in the `Interceptor` and adding a `WriteHeader` method to capture status codes and notify consumers by calling `OnWriteHeader`. Our `Interceptor` type is flexible and composable. Its only duty is to capture HTTP status codes and notify consumers.

- [Listing 9.6: Intercepting method calls](../../all-listings/09-composition-patterns/06-intercepting-method-calls.md)

`Interceptor` satisfies the ResponseWriter interface by embedding it because the compiler automatically promotes all of that interface’s methods to the `Interceptor` type. Satisfying `ResponseWriter` allows us to pass an `Interceptor` to handlers as a `ResponseWriter`.

When we inject an `Interceptor` into a handler, whenever the handler calls `WriteHeader`, that `Interceptor` captures the HTTP status code and calls `OnWriteHeader` before calling the original `ResponseWriter`. Go automatically forwards the rest of the `ResponseWriter` method calls from the handler to the handler’s original `ResponseWriter`.

> [!TIP]
> Although useful, field embedding can also be dangerous. Embedding a type that implements the `json.Marshaler` interface, for example, can change how our type gets serialized to JSON. Visit https://go.dev/play/p/RvISoxRjFBz for an example.

### 9.3.2 Capturing and saving
Now that we can intercept `WriteHeader` calls, we’ll declare the `StatusCode` middleware to capture HTTP status codes into the provided integer variable, as shown in the following listing.

- [Listing 9.7: `StatusCode` middleware](../../all-listings/09-composition-patterns/07-statuscode-middleware.md)

We’ve added middleware similar to the previous `Duration` middleware. When a request comes in, `StatusCode` wraps the original `ResponseWriter` with an `Interceptor` to track the handler’s WriteHeader call and capture the status code without the handler’s knowledge.

Because `Server` responds with `http.StatusOK` if the handler never calls `WriteHeader`, we initially set the status code ourselves. Otherwise, it would be too late to capture the code. If the handler calls `WriteHeader` with a different code, `OnWriteHeader` updates the provided variable. In any case, we ensure that that variable is updated with a valid status code.

### 9.3.3 Integration
Now that we’ve implemented `StatusCode` to capture HTTP status codes, we can integrate it into our response recorder chain and the logger middleware to log HTTP status codes. In listing 9.8, we update `Response` to include a new `StatusCode` field and add our `StatusCode` middleware to the `RecordResponse` helper. Finally, we modify our logger middleware to log the captured HTTP status codes along with response durations.
- [Listing 9.8: Integrating `StatusCode`](../../all-listings/09-composition-patterns/08-integrating-statuscode.md)

We’ve updated the `Response` to include a new `StatusCode` field and integrated our `StatusCode` middleware into the `RecordResponse` helper. We’ve also updated the logger middleware to log the captured HTTP status codes alongside response durations.

When a request arrives, our logger middleware calls `RecordResponse`, which wraps the handler with both the Duration and StatusCode middleware. The handler serves the request, and each piece of middleware independently captures its respective metric, storing these metrics in a new `Response` struct. When the handler finishes, `Record­Response` returns the populated `Response` to our logger middleware, which logs both the status code and duration.

### 9.3.4 Demonstration
When the following HTTP requests come
```bash
$ curl -i localhost:8080/health
$ curl -i localhost:8080/nobody
```
we see the following server logs:
```bash
. . .path=/health method=GET duration=14.083µs status=200
. . .path=/nobody method=GET duration=11.012µs status=404
```
The flow of these incoming requests looks like this:
```bash
Request -> . . . -> Duration -> StatusCode -> . . . -> Handler
```
This section showed how to use field embedding to satisfy interfaces and intercept methods—practical techniques for programs of all sizes. We offered small and composable functionality that users can mix and match to create their own middleware. Alternatively, users can use helper functions like `Middleware` directly for out-of-the-box functionality.

> [!TIP]
>  Composable building blocks empower a package’s users to customize its functionality to their specific needs. This approach encourages reusability and reduces the need for package authors to make frequent modifications.