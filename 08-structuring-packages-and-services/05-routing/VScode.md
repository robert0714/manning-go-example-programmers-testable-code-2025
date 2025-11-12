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
  * `${workspaceFolder}`: `08-structuring-packages-and-services\05-routing`
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
## 8.5 Routing
### 8.5.1 ServeMux
When a client requests the route
* `/shorten—Server` calls ServeMux.ServeHTTP, which calls Shorten.
* `/r/go—Server` calls ServeMux.ServeHTTP, which calls Resolve.

> [NOTE!]
> I keep it simple by saying “ . . . calls `Shorten`.” A technically correct way of putting it is “ . . . calls the http.Handler returned by the Shorten method.”

We’ll see how dynamic routes like `/r/` work after we learn how to register routes:
* ServeMux.Handle() registers a Handler directly.
* ServeMux.HandleFunc() converts a function to a Handler before registering it.
We can register our handlers like so:
```golang
mux := http.NewServeMux()
mux.Handle("POST /shorten", Shorten(. . .))  #1
mux.HandleFunc("GET /health", Health)  #2
```
#1 Registers the Handler returned by Shorten to handle incoming POST requests to the /shorten route  
#2 Converts the Health function to a Handler and registers it for GET requests to the /health route  

We use `Handle` to register `Shorten` because calling it returns a `Handler`. But we use `Handle­Func` to convert and register `Health` because `Health` is a function, not a `Handler`.

For the `Resolve` handler’s route, we need to use a wildcard segment, such as`/r/{key}`:
```golang
mux.HandleFunc("GET /r/{key}", Resolve(. . .))  #1
```
#1 Registers the Resolve handler for any route matching `/r/{key}`

When a request like `/r/go` comes, `ServeMux` loads "`go`" into the `Request` and sends it to `Resolve`, which retrieves the "`go`" after calling the `Request.PathValue` with "`key`":
```golang
func(w http.ResponseWriter, r *http.Request) {
    . . .r.PathValue("key"). . .  #1
}
```
#1 Returns “go”  

ServeMux parses "`/r/go`" and calls the Request.SetPathValue method to set "`go`". Otherwise, `PathValue` returns an empty string. `SetPathValue` is also helpful for testing a handler because we may not always use `ServeMux` to set segments automatically.
### 8.5.2 Routes
- [Listing 8.13: Routing with `ServeMux`](../../all-listings/08-structuring-packages-and-services/13-routing-with-servemux.md)
- 
We have registered the `link/rest` API handlers on the `ServeMux` and registered the `ServeMux` as the `Server`’s handler. All incoming requests go first to the ServeMux:
```golang
http.Server -> ServeMux.ServeHTTP -> Handler.ServeHTTP (e.g., Shorten())
```
Incoming requests go through a chain of handlers until one of our handlers, such as the `Shorten` handler, receives the requests.

### 8.5.3 Demonstration
```bash
$ go run ./link/cmd/linkd 
```
Now that we have a router and handlers, let’s give it a try:
```bash
$ curl -i localhost:8080/shorten -d 'url=https://github.com/inancgumus'
HTTP/1.1 201 Created
OQzXIiL4
$ curl -i localhost:8080/shorten -d 'url=https://github.com/inancgumus'
HTTP/1.1 409 Conflict  #1
shortening: saving: conflict   #1
$ curl -i localhost:8080/r/OQzXIiL4  #2
HTTP/1.1 302 Found
Location: https://github.com/inancgumus
$ curl -i localhost:8080/r/inanc
HTTP/1.1 404 Not Found
resolving: retrieving: not found
$ curl -i localhost:8080/shorten -XGET  #3
HTTP/1.1 405 Method Not Allowed
$ curl -i localhost:8080/shorten  #4
➥     -d 'url=http://'   #4
shortening: validating: empty host: bad request   #4
```
#1 Shorten calls Shortener.Shorten, gets link.ErrConflict, and calls httpError().   
#2 Sends an HTTP GET request   
#3 Intentionally making a GET request to a POST route   
#4 Intentionally sending an invalid URL and getting an error from rest.Shorten

> [!WARNING]
> Relaunch the server after modifications to avoid running stale code.  

Now we can route requests to specific handlers based on route patterns. This approach simplifies route handling and prepares our server for more complex scenarios.

> [!TIP]
> Visit https://pkg.go.dev/net/http#ServeMux for more details on routing.