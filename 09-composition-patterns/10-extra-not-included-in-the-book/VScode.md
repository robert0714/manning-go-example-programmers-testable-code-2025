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
  * `${workspaceFolder}`: `09-composition-patterns\10-extra-not-included-in-the-book`
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
## 9.10 Exercises
1. Write middleware from scratch that logs HTTP requests and responses.
2. Write middleware that adds an HTTP header to every HTTP response.
3. Write middleware that limits the incoming JSON request body to 1 KB.
4. Write `BodySize` middleware that records the `Request.Body` size. Update `Response` and `RecordResponse` functions accordingly. Test this middleware function.
5. Write middleware that propagates user IDs using `Context` values.
6. Write a `slog.Handler` that always logs with the injected user IDs.
7. Improve `traceid.Middleware` to generate trace IDs from the request header’s `X-Trace-ID` header (use: `r.Header.Get(..)`). Otherwise, use `New()`.
8. Use `ResponseController` to call the `http.Flusher.Flush` in a handler.
9. Add an `XML` handler in the hio package that responds with XML. In your function, use the standard library’s https://pkg.go.dev/encoding/xml package.