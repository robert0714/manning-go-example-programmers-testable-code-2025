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
  * `${workspaceFolder}`: `08-structuring-packages-and-services\04-http-handlers`
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

 
# 8 Structuring packages and services 
## 8.4 HTTP handlers 
### 8.4.1 Handler closures
- [Listing 8.10: `Shorten` handler (link/rest/shortener.go)](../../all-listings/08-structuring-packages-and-services/10-shorten-handler.md)
- 
[`Shorten`](./link/rest/shortener.go#L13C1-L14C1) takes its dependencies directly and returns a `Handler`. `Shorten` takes a logger, for example. This approach makes testing straightforward because we can replace it with a no-op logger to suppress or capture logs for inspection. By contrast, using a global logger would make correct use and testing harder and more likely lead to unexpected issues if another part of the code uses and modifies the same logger.

> [!TIP]
> Explicitly specifying what a function requires to work improves clarity.
`Shorten` uses `PostFormValue` to extract the key and URL from the request body. If the body is parsed and the link is shortened, the closure responds with `StatusCreated`. If an error occurs, the handler stops processing the request after responding with a
status code.

> [!NOTE]
> Always return from a handler after an error to stop further processing. Otherwise, we risk unintended side effects, such as security issues, duplicate operations, or inconsistent responses.

Before wrapping up, let’s see how we might register `Shorten` on a `Server`:
```golang
http.ListenAndServe(
    "localhost:8080",
    Shorten(. . .),  #1
)
```
#1 Registers the returned closure from Shorten as the http.Server’s Handler

Because `Shorten` returns a `Handler` and `HandlerFunc` implements the `Handler`, Go allows directly returning the closure (a `HandlerFunc`) as a `Handler`. `Server` remains unaware of the closure’s underlying type (`HandlerFunc`), calling it for each incoming HTTP request.

### 8.4.2 Redirecting
- [Listing 8.11: `Resolve` handler (link/rest/shortener.go)](../../all-listings/08-structuring-packages-and-services/11-resolve-handler.md)

The [`Resolve`](./link/rest/shortener.go#L31C1-L32C1) function’s signature is similar to that of the `Shorten` function. When `Resolve` gets a `Link` from the `Shortener`, it calls `http.Redirect` to send a `302` HTTP status code and a `Location` header with the `Link`’s original URL. If the key is "`go`" and the URL is "`https://go.dev`", the request and response will look like this:
```bash
$ curl -i localhost:8080/r/go
HTTP/1.1 302 Found
Location: https://go.dev
```
> [!NOTE]
>  Wildcard segments allow us to define dynamic URL routes.

This request won’t work until we route HTTP requests to `Resolve`, however. The `PathValue` method can extract wildcard segments from a `Request`, but only if they’ve been preloaded. With a URL pattern like "`/r/{key}`", calling `Path­Value("key")` on a request for "`/r/go`" returns "`go`". Without preloading, it returns an empty string. We’ll use a router from the standard library to preload these details into the `Request` automatically.


### 8.4.3 HTTP status codes
- [Listing 8.12: Errors to HTTP status codes](../../all-listings/08-structuring-packages-and-services/12-errors-to-http-status-codes.md)
> [!NOTE]
> Calling `Error` on an `error` returns the error message as a string.