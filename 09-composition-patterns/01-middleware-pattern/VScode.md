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
  * `${workspaceFolder}`: `09-composition-patterns\01-middleware-pattern`
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
In his “Less is exponentially more” article (https://mng.bz/JwN0), Rob Pike says, “If C++ and Java are about type hierarchies and the taxonomy of types, Go is about composition.” We may be used to inheritance-based designs, large frameworks, or large manager types when we come from other languages. By contrast, Go’s philosophy is similar to UNIX’s: do one thing well. Our packages provide simple, reusable pieces that allow users to compose them together on top of what we provide.
## 9.1 Middleware pattern
In Go, functions are first-class citizens. We can pass them to functions or return them from functions. In chapter 4, we learned that we call such functions higher-order functions. As we’ll soon discover, middleware is similar, but it takes an interface and returns another. Still, we take advantage of both higher-order functions and interfaces.

As the first version, we’ll start with middleware that logs requests but skips logging responses. This version will focus on the core idea of wrapping a handler to add extra functionality. The next section adds support for capturing and logging responses.
### 9.1.1 What is HTTP middleware?
Middleware wraps handlers to add functionality such as logging and tracing without altering the original handler code. Middleware can preprocess requests and postprocess responses. The middleware A in figure 9.1 wraps B, which wraps a handler. Incoming requests pass through them to the handler, and the responses travel back through them before reaching the client.

![architecture diagram](pics/CH09_F01_Gumus.drawio.svg)  
Figure 9.1 Middleware layers process both incoming HTTP requests and outgoing HTTP responses, enhancing functionality without altering the handler code.

Let’s look at a code snippet that shows what middleware looks like:
```golang
func(next http.Handler) http.Handler  #1
```
#1 Returns the middleware’s Handler that wraps the given Handler

The middleware takes a handler and returns another. The returned handler can serve requests using the original handler while performing extra tasks: logging, tracing, and so on. This signature also allows us to chain multiple middleware:
```golang
handler := MiddlewareA(MiddlewareB(Handler))#1
```
#1 The resulting variable’s type is http.Handler.  
### 9.1.2 Example
Suppose that we want to log every incoming request to our HTTP handlers in the `link/rest` package (from chapter 8), such as URL paths (e.g., `/shorten`). Instead of adding logging in each handler, we can use middleware to wrap our ServeMux once and log all requests (figure 9.2).

![A diagram of a computer  AI-generated content may be incorrect.](pics/CH09_F02_Gumus.drawio.svg)  
Figure 9.2 The middleware’s handler forwards requests to `ServeMux` and then logs.

Our middleware returns a handler that sits between incoming requests and our handlers. The requests go to our middleware first; the middleware forwards them until they reach one of our handlers, such as `Shorten`. After the handler serves the request, middleware logs it.
#### INTERFACES IN, INTERFACES OUT
Suppose that our middleware function looks like this:
```golang
func Middleware(next http.Handler) http.Handler { #1
  return http.HandlerFunc( #2
    func(w http.ResponseWriter, r *http.Request) {
      next.ServeHTTP(w, r) #3
      . . . #4
    },
  )
}
```
#1 Returns a handler that wraps the given handler   
#2 Returns the closure as an http.HandlerFunc, which implements http.Handler   
#3 The wrapped handler (next) serves the incoming request.   
#4 Logs the request details here   

This function returns a handler that runs the next handler and then logs the request. To make sure that every incoming request goes through our middleware, we register it:
```golang
http.ListenAndServe(. . ., Middleware(mux)) #1
```
#1 Registers the middleware’s handler that logs requests by wrapping the ServeMux

All incoming requests go through middleware and eventually to handlers. Middleware logs requests when a handler finishes serving a request.

#### GLOBAL LOGGERS
Our middleware logs with a global logger because we don’t pass a logger to it.

> [!TIP]
>  Avoid global loggers. Instead, inject loggers explicitly into middleware or handlers because direct injection simplifies testing and enhances flexibility.

We prefer not to use a global logger because it makes testing harder. (See chapter 5 for the reasons.) Instead, we can configure and pass a specific logger to our function:
```golang
func Middleware(lg *slog.Logger) func(next http.Handler) http.Handler
```
This function is similar to the previous one but returns middleware. Its usage is more complicated, but the benefits outweigh the costs (allowing us to use a specific logger):
```golang
wrap := Middleware(/* pass the logger here */)
http.ListenAndServe(. . ., wrap(mux))
```
This time, we pass a specific logger to `Middleware`, get a middleware function, and finally wrap our ServeMux. This setup operates as before but without a global logger.

Our new function returns the `func(Handler) Handler` middleware signature. That way, we can chain multiple middleware and use third-party ones without adding extra code to be compatible. Doing so also teaches us a valuable lesson that we’ve seen many times: keeping function signatures is extremely important for maintainable, reusable, and composable code.

> [!TIP]
>  Be consistent in your function (methods are functions too) signatures.

### 9.1.3 Practice
Now that we’re familiar with middleware, we’ll implement one (see listing 9.1) that logs request details, such as paths. Our `hlog` package provides HTTP-specific logging to minimize boilerplate and ensure consistent logging. Expert packages like `hlog` improve maintainability. By contrast, generic packages may introduce unnecessary dependencies.
- [Listing 9.1: Logger middleware](../../all-listings/09-composition-patterns/01-logger-middleware.md)

We wrote a middleware function that logs request details, such as the URL path. Before seeing this middleware in action in section 9.1.4, let’s walk through the code:

1. We use nested closures to provide access to the variables `lg`, `next`.
2. Middleware takes a logger and returns a middleware function.
3. That function returns the middleware’s handler, which is its main logic.
4. The middleware’s handler runs `next` to serve requests and then logs the requests.
5. The handler repeats the preceding step for every incoming request.
   
We also declared `MiddlewareFunc` to maintain consistent function signatures and group middleware functions in the documentation to better understand the package’s use.

### 9.1.4 Learning more about the slog package and the any interface
For structured logging, we use the standard library’s `slog` package. The `slog.String` function returns a `slog.Attr` struct value (a log attribute), which turns into a key-value pair, such as "`method=GET`", when we pass the result of the `String` function to the `Logger`’s `LogAttrs` method to log. Although we use a `String` attribute, there are many others, such as `Int` for integers. We use our `lg` variable, which is a `*slog.Logger`, to call methods such as `LogAttrs` for logging.

> [!NOTE]
> `slog.Attr` is a log attribute with a string key and a value that can store any type. See the `slog` documentation at https://pkg.go.dev/log/slog for more information.

Because we log every incoming request, efficiency is crucial for optimal performance; because we can pass `LogAttrs` type-safe values, it won’t have to guess their types. By contrast, the `Logger.Log` method takes variadic any arguments and has to guess the underlying types of these arguments at runtime:
```golang
lg.Log(. . ., "status", 404)  #1
```
#1 Accepts any types of arguments: func Log(. . ., args ...any)

> [!TIP]
> Prefer `slog.LogAttrs` for efficient and type-safe logging.
The `any` type is an interface type without methods, also known as an empty interface:

```golang
type any interface {
    #1
}
```
#1 Does not declare methods

Because `any` has no methods, every other type satisfies it. This flexibility allows methods like `Log` to accept various types of arguments. There is a cost, however: the compiler can’t know the underlying type an empty interface value wraps inside beforehand. As a result, `Log` needs to inspect the underlying type at runtime. Languages like Python, Ruby, and JavaScript don’t complain until runtime if we pass the wrong type to a function. Go catches these mistakes much earlier, during compilation—unless we start using `any`. Excessive use of `any` may create surprises in our code’s runtime behavior, undermining Go’s strong typing advantages.

As we discussed, using `any` may introduce runtime overhead and weaken Go’s type safety guarantees. In our logger middleware, however, using the `slog.Any` function, which takes an `any` value argument, provides an advantage over `slog.String`. We log `Request.URL` using `slog.Any` to be more efficient. slog.Logger detects and internally calls `URL.String` when the `Logger` is enabled. If we used `slog.String`, we would need to call `URL.String` ourselves even when the logger is off, causing unnecessary overhead:
```golang
slog.String("path", r.URL.String()) #1
```
#1 r.URL.String() runs even if the logger won’t log.   

Sometimes, deferring a value’s resolution to runtime, despite the usual compile-time advantages, leads to better efficiency, as demonstrated by `slog.Any` in this example.
### 9.1.5 Integration
Now that we have middleware, let’s wrap the `ServeMux` from chapter 8. As shown in the next listing, we set middleware on the `Server` as its `Handler` after wrapping the `ServeMux`. Every incoming HTTP request goes to our middleware before reaching our `link/rest` handlers.

- [Listing 9.2: Activating the middleware](../../all-listings/09-composition-patterns/02-activating-the-middleware.md)

This integration was straightforward. We added logging to the link/rest API without changing any handler code, reducing bugs and improving maintainability. Next, let’s send an example HTTP request to see the logs:
```bash
$ curl -i localhost:8080/health
```
Here are the server logs:
```bash
. . .level=INFO msg=request path=/health method=GET #1
```
#1 Logs using slog.Any(“path”, r.URL) and slog.String(“method”, r.Method)

The flow of the incoming request looks like this:
```bash
Request -> Server -> Middleware -> ServeMux -> Health
```
When the request arrives, the `Server` forwards it to the middleware’s handler by calling its `ServeHTTP` method. The handler forwards the request to the `ServeMux` and then to our health handler. The middleware’s handler logs request details after the request is served.

In this section, we combined first-class functions with interfaces. Our middleware can easily be composed with others because it uses a consistent middleware signature. Idiomatic code values efficiency, so we optimized our code to eliminate unnecessary work. Next, we’ll improve our middleware by capturing additional response metrics.

> [!TIP]CONFIGURABLE AND TESTABLE TYPES
> `Middleware` uses the concrete `slog.Logger` type to log incoming requests, so we keep our logger middleware flexible by leaving the `Logger`’s configuration to the caller. Many programmers overcomplicate by hiding loggers behind interfaces up front, assuming that this approach makes testing or later replacing the logger straightforward. Despite being a concrete type, however, `Logger` is configurable and designed to integrate with other logger solutions when necessary. Using `Logger` directly simplifies implementations.