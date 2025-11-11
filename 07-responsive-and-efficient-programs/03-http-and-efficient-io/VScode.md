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
  * `${workspaceFolder}`: `07-responsive-and-efficient-programs\03-http-and-efficient-io`
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
## 7.3 HTTP and efficient I/O operations
### 7.3.1 Round-tripping
The `net/http` package has two prominent types:
* `http.Client` to send requests to an HTTP server
* `http.Server` to serve HTTP requests to clients


> [!NOTE]
> Sending a request and returning a response is an `HTTP round trip`. In section 7.4, we customize the `Client`’s round-trip behavior to provide a more performant `Client`.

![architecture diagram](pics/CH07_F04_Gumus.drawio.svg)   
Figure 7.4 Sending a request to a server using an HTTP client and receiving a response. When we receive a response, we can read its body as a byte stream.  
We know that we can create a new `Request` using the `http.NewRequest` function:
```golang
req, err := http.NewRequest(  #1
    http.MethodGet, "http://...", http.NoBody,
)
```
#1 Returns a new *http.Request and an error if NewRequest fails

Recall that we used the Request’s Clone method earlier to attach a Context to a Request. We can also use NewRequestWithContext to get a new Request with a Context right away:
```golang
req, err := http.NewRequestWithContext(  #1
    context.Background(),   #1
    http.MethodGet, "http://...", http.NoBody,
)
```
#1 Returns a new *http.Request with a Context attached

Now we can call `Client.Do` to send this Request to a server and receive a Response:
```golang
res, err := http.DefaultClient.Do(req)  #1
```
#1 Sends the HTTP request to the server and returns an *http.Response

> [!NOTE]
> `DefaultClient` is a variable with the http package’s default Client.

One way to check whether a request is successful is to look at its HTTP status code:
```golang
if res.StatusCode == http.StatusOK { . . . }
```
Although we can access information like an HTTP status code, the returned `Response` does not include the server’s full response. For that, we need to dig into the `Body` field.

### 7.3.2 Interface composition
The Response type’s Body field is an io.ReadCloser, a composition of two interfaces:
```golang
type ReadCloser interface {
    Reader  #1
    Closer  #2
}
```
#1 ReadCloser has a Read method because the Reader interface has a Read method.    
#2 ReadCloser has a Close method because the Closer interface has a Close method.   

ReadCloser embeds the existing io.Reader and io.Closer interfaces. Embedding an interface promotes its methods to the embedder. So ReadCloser has Read and Close because it embeds the interfaces that have these methods. Reader looks like this:
```golang
type Reader interface {
    Read(p []byte) (n int, err error)  #1
}
```
#1 Specifies the behavior of reading a chunk of bytes into a []byte from a stream   

`Closer` looks like this:
```golang
type Closer interface {
    Close() error	  #1
}
```
#1 Specifies the behavior of closing an underlying resource (e.g., TCP connections or files)

> [!TIP]
> When we’re embedding interfaces, the convention is to name the new interface with an `-er` suffix only once. We use `ReadCloser` instead of `ReaderCloser`, for example, Similarly, `ReadWriteSeeker` embeds `Reader`, `Writer`, and `Seeker` interfaces.

Now that we understand `ReadCloser`, let’s consider a hypothetical example to see why `ReadCloser` embeds `Reader` and `Closer` instead of declaring their methods directly:
```golang
type ReadCloser interface {
    Read(p []byte) (n int, err error)
    Close() error
}
```

Although this code behaves identically to the original `ReadCloser`, Go favors composition, and we often declare larger interfaces by embedding smaller ones. The embedding approach used in the original `ReadCloser` instantly shows that a type must satisfy both Reader and Closer to qualify as a `ReadCloser`. Because `Reader` and `Closer` are widely used, embedding prevents duplication, clarifies the `ReadCloser`’s intent, and improves readability and understanding.

Standard library interfaces such as `Reader` and `Closer` are central to Go’s approach to system programming, prioritizing efficient use of I/O resources. Calling `Client.Do`, for example, returns a `Response` without downloading the response body immediately.

The `Response.Body` field is a `Reader`, allowing us to read the response incrementally in smaller chunks of bytes. This approach avoids loading large responses into memory at the same time, and `Body`, as a `Closer`, lets us release resources when finished.

> [!TIP]
> Closing the `Body` allows the `http.Transport` to reuse HTTP connections.

Composition and efficient stream processing are two key themes in Go. Interfaces like `ReadCloser` are built from smaller standard interfaces to clarify their intent. Interfaces such as `Reader` and `Closer` also establish explicit standard protocols for handling byte streams efficiently. They promote incremental reading of data and explicit release of resources, ultimately leading to improved performance and reliability in programs.

### 7.3.3 ReadAll: Eat it all

### 7.3.4 Copy: Eat small, be small

### 7.3.5 Putting it all together
- [Listing 7.4: Processing a request and response](../../all-listings/07-responsive-and-efficient-programs/04-processing-a-request-and-response.md)

### 7.3.6 Demonstration
```bash
$ go run ./hit/cmd/hit -n 10 -c 10 https://duckduckgo.com
. . .
Sending 100 requests to "https://duckduckgo.com" (concurrency: 10)

Summary:
    Success:  100%
    RPS:      285.0
    Requests: 100
    Errors:   0
    Bytes:    4926500
    Duration: 351ms
    Fastest:  11ms
    Slowest:  88ms
```