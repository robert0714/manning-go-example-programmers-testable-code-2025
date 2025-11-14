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
  * `${workspaceFolder}`: `09-composition-patterns\06-handler-chaining-pattern`
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
## 9.6 Handler-chaining pattern
We’ve explored middleware patterns, using them to log HTTP requests and responses, capture metrics, and propagate trace IDs. Starting in this section, we’ll shift focus from middleware to HTTP handlers. We’ll introduce a handler-chaining pattern to eliminate common mistakes like forgetting to return after responding and simplify writing consistent handler responses.

As figure 9.7 shows, the `Shorten` handler (from chapter 8) now delegates response handling by explicitly returning either a Text handler (to respond with text) or an Error handler (to handle errors). This explicit return forms a handler chain. `Shorten` receives the request, and these subsequent handlers write the response to the clients.


![architecture diagram](pics/CH09_F07_Gumus.drawio.svg)  
Figure 9.7 Shorten delegates response processing to other handlers in the chain.

We’ll introduce a chainable handler type that enforces explicit returns from handlers. This type helps prevent subtle bugs such as unintentionally sending unnecessary responses or incorrect headers. Responding with errors, redirects, or JSON will become simpler, clearer, and safer.

In later sections, we’ll revisit language mechanics, such as detecting behaviors through interfaces using type assertions. We’ll demonstrate how these techniques help us handle practical concerns such as performing validation automatically and protecting our server from malicious clients. We’ll also see how these techniques align with Go’s philosophy of extending behavior without tight coupling or inheritance, keeping our code simple, composable, and reusable.

### 9.6.1 Chainable handlers
Consider the following handler code.
```golang
func redirect(w http.ResponseWriter, r *http.Request) {
  uri, err := . . .
  if err != nil {
    http.Error(w, err.Error(), http.StatusBadRequest)
  }
  http.Redirect(w, r, uri, http.StatusFound)
}
```
Did you spot the issue? If an error occurs, the handler should immediately return after calling `http.Error`. Missing this return means that the handler continues, unintentionally redirecting the client and potentially causing confusion or security issues. Remembering to return after each error response can be tedious and error-prone, however, particularly in large codebases. To prevent such problems, we’ll declare a custom handler type, as shown in listing 9.15. Unlike the `http.Handler`, this type requires returning a handler (`hio.Handler`), making it impossible to forget to return. Returning a handler ensures that handlers never unintentionally continue processing after responding, preventing subtle logic errors and unintended responses.
- [Listing 9.15: A chainable custom handler type](../../all-listings/09-composition-patterns/15-a-chainable-custom-handler-type.md)

An `hio.Handler` returns the next `hio.Handler` to enable chaining and guarantee explicit returns. Its `ServeHTTP` method implements the `http.Handler` interface. This method calls the current `Handler` function and, if that function returns a non-nil `Handler`, `ServeHTTP` runs the returned `Handler`. This process continues until the last `Handler` returns `nil`, terminating the chain and immediately ending further request and response handling. Because `hio.Handler` implements `http.Handler` with the `ServeHTTP` method, we can use it as an `http.Handler`.
### 9.6.2 Response handlers
We’ve introduced a chainable handler type (`hio.Handler`) that requires explicit handler returns, helping to eliminate subtle bugs. Next, we’ll add convenient response helpers on top of this type (and use them in the `link/rest` handlers in section 9.6.4):
* `Responder` groups response-helper methods.
* `Error` returns a `Handler` that calls an error handler to handle errors.
* `Redirect` returns a `Handler` that redirects clients to another URL.
* `Text` returns a `Handler` that responds with a text message.
  
> [!NOTE]
>  The `Handler` type here is our `hio.Handler`, not Go’s `http.Handler`.

#### RESPONDER

Let’s start by adding the Responder type to group response helpers in the next listing.
- [Listing 9.16: `Responder` type(link/kit/hio/response.go)](../../all-listings/09-composition-patterns/16-responder-type.md)

We declared the `Responder` type to group helpers. Its `err` field allows consumers to specify a custom error handler we can call if errors occur during the response. Later, we’ll use the `httpError` function from chapter 8 as an error handler for `Responder` in section 9.6.3.

With err, consumers can handle errors themselves, such as using slog to log errors. If the Responder used a specific error-handling mechanism directly, such as using slog to log errors, that would reduce its reusability and cause unnecessary slog dependency on the hio package. Reducing direct dependencies improves code modularity and flexibility.

#### ERROR HANDLER
Next, we’ll add our first response helper for writing formatted error responses.
- [Listing 9.17: `Error` response helper(link/kit/hio/response.go)](../../all-listings/09-composition-patterns/17-error-response-helper.md)

We’ve added `Error`, which formats an error message and calls and returns the result of the `err` function field to delegate error handling. We can declare a new `Responder` variable and then call the `Error` method to get a `Handler` we can use to respond to clients:
```golang
rs := NewResponder(func(err error) Handler {
    . . . #1
    return nil #2
})
h := rs.Error("bad input: %d", 42)
```
#1 Handles the error   
#2 Terminates the chain   

We set the `Responder`’s `err` field to a function that returns a `nil Handler` after handling the error, terminating the chain. We can run the returned `Handler` using `ServeHTTP`:
```golang
h.ServeHTTP(w, r)
```
This method runs the `Handler` that the `err` field returns. Whether a chain runs the next `Handler` depends on the returned Handler. The chain stops if consumers have configured `err` to return `nil`; otherwise, the chain continues running the next `Handler` until one returns `nil`. Because we set `err` to return `nil`, the chain terminates.

#### REDIRECT HANDLER
Next, we’ll add another response helper to simplify HTTP redirects. This helper is similar to `Error`, except that this helper returns a closure.
- [Listing 9.18: `Redirect` response helper](../../all-listings/09-composition-patterns/18-redirect-response-helper.md)

`Redirect` returns a `Handler` closure that redirects clients to the specified URL and status code when called. Returning Redirect from a `Handler` automatically calls this closure when a request comes (when we register that `Handler` on a `Server`, for example). When run, this closure, immediately redirects clients and returns `nil`, terminating the handler chain, and preventing accidental continuation.

#### TEXT HANDLER
Finally, we’ll simplify writing plain-text responses by adding one more response `Handler`.

- [Listing 9.19: `Text` response helper](../../all-listings/09-composition-patterns/19-text-response-helper.md)

`Text` returns a `Handler` closure, which sets the response content type, writes the specified status code, and writes a text message to clients. After responding, it returns `nil`, terminating the chain to prevent another `Handler` from setting the HTTP headers again. (Because HTTP headers come before response bodies, returning headers after responding the body indicates logic issues in our code.)

We’ve added reusable response helpers using our `Handler` type. They reduce code duplication, improve readability, and eliminate subtle bugs due to forgotten returns. Next, let’s apply the handler-chaining pattern with these helpers to our handlers.
### 9.6.3 Responder
Before moving to the handlers, let’s add a helper function to configure a Responder with an error handler in the next listing. We’ll reuse this helper within our handlers in section 9.6.4.
- [Listing 9.20: `Responder` helper](../../all-listings/09-composition-patterns/20-responder-helper.md)

The `newResponder` helper takes a logger and returns a new `Responder`. We set the `Responder`’s error handler to a closure (`err`) that returns a `Handler`. This closure calls `httpError` (chapter 8) and then terminates the handler chain. `httpError` maps common errors of our program, such as `ErrBadRequest`, to corresponding HTTP status codes and calls `http.Error` to respond to clients with those status codes and error messages.

### 9.6.4 Integration
As listing 9.21 shows, we’ll update our handlers and include them in the handler chain. In each handler, we create a `Responder` using `newResponder`. The `Shorten` handler outputs a text message by returning a `Text` handler, and `Resolve` redirects to a URL using `Redirect`. We return `Error` to write an error message to clients, terminating the chain.
- [Listing 9.21: Chainable handlers](../../all-listings/09-composition-patterns/21-chainable-handlers.md)

Let’s discuss this change step by step to see what happened:
1. `Responder` helps with responding and handling errors.
2. Handlers return an `hio.Handler` to respond, such as `Redirect` or `Text`.
3. Handlers return an `Error` if the processing should stop with an error.

Returning an `hio.Handler` explicitly makes each handler a step in the handler chain. Suppose an HTTP request comes in to shorten a URL. The returned `Handler` from `Shorten` runs and then returns another `Handler`, such as `Text`, to respond with text and status code. The request processing continues until the `Text` handler returns `nil`, stopping the handler chain.

From the outside, nothing has changed: `hio.Handler` is an `http.Handler`, so integration with the `http.Server` remains the same. But internally, we’ve significantly reduced potential risks such as forgetting to return after responding, improving correctness.