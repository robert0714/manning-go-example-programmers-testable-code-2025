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
  * `${workspaceFolder}`: `08-structuring-packages-and-services\03-http`
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
## 8.3 HTTP
Adding a `ServeHTTP` method to a type satisfies the `Handler` interface. Then, as figure 8.6 shows, we can register a `Handler` implementation on the `Server` to serve requests.

![architecture diagram](pics/CH08_F06_Gumus.drawio.svg)  
Figure 8.6 The `http.Server` routes incoming HTTP requests to its `http.Handler` by calling the `Handler`’s `ServeHTTP` method, passing an `http.ResponseWriter` and `http.Request`. Then the `Handler` can use the `ResponseWriter` to respond to clients via the `Server`.  

To handle HTTP requests concurrently, Server runs the registered Handler’s ServeHTTP method in a separate goroutine. Unlike most other languages, in which launching a single thread for each request could be overkill, Go handles this process effortlessly because goroutines are lightweight, and Go’s scheduler multiplexes them onto operating system threads. It’s not surprising to see that a simple HTTP server can handle ~100,000 requests per second.

The `ServeHTTP` method takes `http.ResponseWriter` and `*http.Request` parameters. `Request` contains HTTP request details, such as URL and query parameters. `Response­Writer` is an interface with the following methods that we can use to send HTTP responses to clients:
* `Header()`—Sets HTTP response headers
* `WriteHeader(code int)`—Sets the HTTP status code (default is `200` if not called)
* `Write([]byte)`—Writes the response body to the client


### 8.3.1 Health check

Now that we know what to do, let’s implement the handler as shown in listing 8.8. `Health` takes a `ResponseWriter` to respond to clients and a `Request` to read the request. It responds "`OK`" by implicitly calling `ResponseWriter.Write` via `Fprintln`, which takes a `Writer`. Because `ResponseWriter` is a `Writer`, `Fprintln` can call its `Write` method.
- [Listing 8.8: A health check handler](../../all-listings/08-structuring-packages-and-services/08-a-health-check-handler.md)

We’ve implemented our first handler, which writes "`OK`" to clients. By default, handlers write a status code `200`, meaning success. Handlers do that automatically if we don’t specify a status code using `WriteHeader` before calling `Write`. This is useful because we don’t have to return a `200` status code manually. Our handler doesn’t write an HTTP status code using `WriteHeader`, for example, but it still responds with an HTTP status code of `200`.
### 8.3.2 Serving HTTP
Now that we have the `Health` function, we’re almost ready to serve HTTP requests. We can use the `http.ListenAndServe` function to start an HTTP server as follows:
```golang
http.ListenAndServe(
    "localhost:8080", /* http.Handler here */,  #1
)
```
#1 An http.Handler implementation serves all incoming HTTP requests at localhost:8080.   

Behind the scenes, `ListenAndServe` configures and starts a `Server` that routes incoming requests to the registered `Handler` that we set. We can use `Health` to serve HTTP requests after we convert it to a `Handler` using the `Handler­Func` type:
```golang
http.ListenAndServe(
    "localhost:8080", http.HandlerFunc(Health),  #1
)
```
#1 http.HandlerFunc converts the Health function to a Handler implementation.   

`HandlerFunc` converts `Health` to a Handler on the fly. `HandlerFunc` is like an adapter that converts a function to one that satisfies the `Handler` interface (and you may remember `HandlerFunc` from chapter 7’s testing section):
```golang
type HandlerFunc func(ResponseWriter, *Request)  #1

func (f HandlerFunc) ServeHTTP(  #2
    w ResponseWriter, r *Request,   #2
) {  
    f(w, r)  #3
}
```
#1 HandlerFunc can convert only a function with this signature (which Health’s signature matches).  
#2 HandlerFunc implements the Handler interface with the ServeHTTP method.  
#3 Calls the converted function (e.g., Health), passing it the ResponseWriter and Request  

Calling the `ServeHTTP` method calls the converted function—in this case, `Health`—so the `Server` started by `ListenAndServe` will call `Health` for every incoming HTTP request.

### 8.3.3 HTTP server
- [Listing 8.9: Running an HTTP server](../../all-listings/08-structuring-packages-and-services/09-running-an-http-server.md)

We created a program that runs an HTTP server, routing all HTTP requests to `Health`. The server’s default listening address is `localhost:8080`. We also set up a logger to announce that the server is starting. We’ll use this logger in our other handlers. Now let’s run this program to launch the server and start serving HTTP requests:
```bash
$ go run ./link/cmd/linkd 

time=2061/07/29 23:58:00 #1
➥ level=INFO msg=starting app=linkd addr=localhost:8080  
```
#1 We’ll omit the time prefix in the log message from now on for brevity.

Because the server waits for incoming connections, we can send a request to the server from another terminal using our favorite HTTP client (or a browser):
```bash
$ curl -i localhost:8080  #1
HTTP/1.1 200 OK  #2
OK  #3
```
#1 Sends an HTTP GET request to the HTTP server  
#2 Automatically writes a 200 OK HTTP status code  
#3 Written to ResponseWriter.Write by fmt.Fprintln  

We’ve implemented and run an HTTP server program that handles health check requests. Now we’re ready to add the link-shortener HTTP handlers to this program.

> [!TIP]
> Visit https://pkg.go.dev/net/http for more information on the `http` package.

> [!TIP] THE SLOG PACKAGE
Go 1.21 introduced `slog` for structured logging, making it easier to format logs as key–value pairs and integrate seamlessly into larger systems. It standardizes logging with Go’s standard library without third-party modules. We can instantiate a new `Logger` with the built-in slog handlers, such as `TextHandler` and `JSONHandler`, or craft our own. `TextHandler`, for example, formats logs in human- and machine-readable form:
> ```golang
> lg := slog.New(slog.NewTextHandler(os.Stderr, nil))
> lg.Info("server started", "address", "localhost:8080", "version", ➥"1.0.0")
> ```
> Output:
> ```bash
> time=2009-11-10T23:00:00.000Z level=INFO
> ➥msg="server started" address=localhost:8080 version=1.0.0
> ```
> Here’s another with JSONHandler:
> ```golang
> lg := slog.New(slog.NewJSONHandler(os.Stderr, nil))
> lg.Info("server started", "address", "localhost:8080", "version", ➥"1.0.0")
> ```
> Output:
> ```bash
> {"time":"2009-11-10T23:00:00Z","level":"INFO",
> ➥ "msg":"server started",
> ➥ "address":"localhost:8080","version":"1.0.0"}
> ```
> These examples show how `slog` adapts to different needs, making logging simple, efficient, and versatile. Visit https://go.dev/blog/slog for more information. 