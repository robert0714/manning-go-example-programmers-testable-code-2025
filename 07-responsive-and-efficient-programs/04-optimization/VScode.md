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
            "program": "${workspaceFolder}/url/cmd", 
            "cwd": "${workspaceFolder}",
            "env": {},
            "args": []
        }
        ]
  }
  ```
  * `${workspaceFolder}`: `07-responsive-and-efficient-programs\04-optimization`
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

 
# 7 Responsive and efficient programs
## 7.4 Optimization
When a client establishes a TCP connection, it can send HTTP requests to the server. But establishing the initial TCP connection is quite chatty and expensive. It’s similar to the following (although TCP messages have more complex details, such as window sizes):
```bash
Server: Hello, would you like to hear a TCP joke? [SYN]
Client: Yes, I'd like to hear a TCP joke. [SYN, ACK]
Server: OK, I'll tell you a TCP joke. [ACK]
Client: OK, I'll hear a TCP joke. [ACK]
Server: Are you ready to hear a TCP joke? [DATA, ACK]
Client: Yes, I am ready to hear a TCP joke. [ACK]
Server: OK, I'm about to send the TCP joke.
        It will last 10 seconds. [DATA, ACK]
Client: OK, I'm ready to hear the TCP joke
        that will last 10 seconds. [ACK]
. . . #1
```
#1 HTTP request and response happen somewhere here.   

Fortunately, HTTP allows the server and client to keep previously established connections alive until they time out—a feature called `keep-alive`. Clients can reuse existing TCP connections to send HTTP requests without reconnecting using this feature.

Suppose that the HIT client sends 1 million requests with 10 dispatcher goroutines. Its performance would be subpar because it uses `http.DefaultClient` (the default `http.Client`), which keeps 100 connections open and allows only 2 to be reused for the same host. The HIT client would reconnect each time to send more than two requests to the same host.

Although two goroutines could send requests over these two TCP connections without reconnecting, others need to open new ones to send more. We can’t measure a server’s performance accurately using the `DefaultClient`; we’ll find another way.

In this section, we’ll explore the `http.Client` type so we can gain enough knowledge to optimize its performance. Then we’ll configure a more performant custom `Client` and use it to send HTTP requests.

### 7.4.1 Client and its RoundTripper
Go favors composition, but composition is not limited to interfaces. Structs can have interface fields for extensibility and reusability. The `Transport` field of the `http.Client` struct, for example, is of the `RoundTripper` interface type, enabling the `Client` to delegate the HTTP request and response handling to various `RoundTripper` implementations.

Figure 7.7 shows that `Client` doesn’t establish TCP connections or send HTTP requests by itself. Instead, it delegates this duty to a `RoundTripper`, which can handle TCP connections and HTTP requests and responses, as well as manage a pool for reusing TCP connections for efficiency.

![architecture diagram](pics/CH07_F07_Gumus.drawio.svg)   
Figure 7.7 The HIT client goroutines send concurrent HTTP requests to the HTTP server via the `http.Client`, with TCP connections in the `http.Transporter`’s pool.

The `http.Client` type and its `Transport` field look like this:
```golang
type Client struct {
    Transport     RoundTripper  #1
    CheckRedirect func(  #2
        req *Request, via []*Request,
    ) error
    Timeout       time.Duration  #3
    . . .
}
```
#1 Delegates HTTP request and response processing to a RoundTripper implementation   
#2 Specifies the HTTP redirect handling behavior   
#3 Specifies the maximum time limit for each round trip  

The `http.RoundTripper` interface looks like this:
```golang
type RoundTripper interface {
    RoundTrip(*http.Request) (*http.Response, error) #1
}
```
#1 Sends the http.Request and returns an http .Response and an error 

The `Client`’s `Transport` field, the `RoundTripper` interface, allows switching between protocol implementations, such as HTTP/1 and HTTP/2. We’ll implement this interface in section 7.5, set it to this field, and test the HIT client without running a server.

A `RoundTripper` implementation may establish TCP connections, send HTTP requests, and return HTTP responses. The `http.Transport` type, for example, is a `RoundTripper` with a connection pool. Because it has a connection pool to reuse connections for efficiency, we should reuse the same `Transport` while sending HTTP requests.

> [!NOTE]
> `http.DefaultTransport` is the `http` package’s default `RoundTripper`. `http.DefaultClient` uses the `DefaultTransport` for requests and responses.

Let’s look briefly at the `Client`’s remaining fields. `CheckRedirect` allows us to handle HTTP redirects; we can assign it a function to prevent redirects. The `Timeout` field allows us to set the maximum timeout for a round trip—the time between establishing a connection, sending an HTTP request, and processing the response. We’ll use both.

> [!NOTE]
> Many standard library types are designed with extension points, such as having interface fields and accepting interface parameters. These types are flexible, highly configurable, and inherently testable by their design.

### 7.4.2 Tweaking the connection pool
- [Listing 7.5: Optimizing `http.Client`](../../all-listings/07-responsive-and-efficient-programs/05-optimizing-httpclient.md)

> [!TIP]
> `Client` has other options that customize its behavior. You can find them all at https://pkg.go.dev/net/http#Client and https://pkg.go.dev/net/http#Transport.

#### 7.4.3 Demonstration
Now that we’ve tuned the HIT client’s `Client` and `Transport`, let’s benchmark the previous version of the HIT client that uses `DefaultClient` and then the one with our tuned version. We’ll run the following benchmarks against an unloaded local HTTP server. You can find the server code at https://go.dev/play/p/Bf4QuOC1xpg. First, run the previous HIT tool that uses `DefaultClient`:
```bash
$ go run ./hit/cmd/hit -n 100_000 -c 10 http://localhost:8082
    . . .
    Success:  84%
    RPS:      1552.0
    Errors:   15591  #1
    Duration: 1m4.451s
```
#1 The sheer number of TCP connections cause errors.

Run the new version of the HIT tool:
```bash
$ go run ./hit/cmd/hit -n 100_000 -c 10 http://localhost:8082
    . . .
    Success:  100%
    RPS:      60425.0
    Errors:   0
    Duration: 1.655s
```    
After the optimizations, the HIT client became significantly more performant, and users can measure HTTP server performance more accurately. We reuse the same `Transport` to exploit the connection pool when sending requests and avoid unnecessarily reestablishing TCP connections to the same host. The pool reacquires an idle connection after the request ends and releases that connection for another request, improving performance.