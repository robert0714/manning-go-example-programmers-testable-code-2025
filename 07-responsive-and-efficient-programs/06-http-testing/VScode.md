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
  * `${workspaceFolder}`: `07-responsive-and-efficient-programs\06-http-testing`
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
## 7.6 HTTP testing
### 7.6.1 Package httptest
Before we start testing the HIT client, let’s learn how to run a test HTTP server:
* The `httptest.Server` type runs an `http.Server` that is tuned for testing.
* The `Server` type serves HTTP requests with an `http.Handler`.
* The `Handler` is an interface with a `Handle` method.
  
To test the HIT client against a test HTTP server, we need to pass a `Handler` to the `Server`. Instead of implementing the `Handler` interface, we’ll use an adapter function type called HandlerFunc to turn our function into a `Handler` that can serve HTTP requests. We can start a test HTTP server using the `httptest.NewServer`:
```golang
func NewServer( #1
    handler http.Handler,  #1
) *httptest.Server  #1
```
#1 Starts and returns an *httptest.Server that serves HTTP requests with the http.Handler

This function requires a `Handler` to serve HTTP requests:
```golang
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```
We can convert a closure to a `Handler` using `HandlerFunc` and pass it to `NewServer` to start a new HTTP test server:
```golang
server := httptest.NewServer(
  http.HandlerFunc(  #1
    func(_ http.ResponseWriter, _ *http.Request) {  #2
      . . .  #3
    },
  ),
)
```
#1 Converts the closure to an http.Handler implementation   
#2 Matches the signature of the http.HandlerFunc type   
#3 Serves the HTTP request here   

We’ve converted a closure to a `Handler` using `HandlerFunc`. `HandlerFunc` works similarly to the `roundTripperFunc` we implemented earlier. The latter converts a function with the right signature to a `RoundTripper`, and HandlerFunc converts a function to a `Handler`. When we have a `Handler`, we pass it to the `NewServer`, which launches a new HTTP server for testing purposes. This test server calls our closure to serve every incoming HTTP request. After we launch a test server, we can get its URL using the `URL` method:
```golang
server.URL()  #1
```
#1 Returns “http://127.0.0.1:61507” (automatically chooses the port number)

When we are done working with the `Server`, we can close it to free resources:
```golang
server.Close()
```
Now we’re ready to test the HIT client.

### 7.6.2 Testing the client
- [Listing 7.8: Testing `SendN`](../../all-listings/07-responsive-and-efficient-programs/08-testing-sendn.md)
We’ve written an HTTP test that runs the HIT client against a test HTTP server. Because `Server` runs a `Handler` in a new goroutine, we use an atomic integer to count the number of requests to prevent race-condition issues. Declaring the atomic integer starts its value at `0`. The `Add` method increments it, and `Load` retrieves its current value.
> [!TIP]
> Visit https://pkg.go.dev/sync/atomic to learn more about `sync/atomic`.

We use `defer` to set the `Server` to close when the test function returns. When we call `Close`, the Server stops listening for connections and closes any idle or new connections. Then we call `SendN` to send 10 requests to the test HTTP `server` over five goroutines. The `*testing.T.Context` method returns a `Context` that is canceled after the test finishes. Finally, we drain the results from the iterator and compare the results. Run the test:
```bash
$ go test ./hit -run=TestSendN
ok      github.com/inancgumus/gobyexample/hit   0.198s
```
