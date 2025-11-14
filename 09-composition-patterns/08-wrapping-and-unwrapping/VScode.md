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
  * `${workspaceFolder}`: `09-composition-patterns\08-wrapping-and-unwrapping`
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
## 9.8 Wrapping and unwrapping
We implemented JSON encoding and decoding helpers, integrating them into the `Shorten` handler. Although these helpers simplified our handler code, our `DecodeJSON` helper currently reads the entire request payload into memory without any size restrictions, potentially leaving our HTTP service vulnerable to malicious requests or excessive resource use.

This section addresses these issues. We’ll wrap the `Request.Body` with another `Reader` to enforce a limit on the number of bytes we can read. To handle cases in which the limit is exceeded, we’ll use type assertions and the `Interceptor`’s `Unwrap` method (section 9.4) to recover the original `ResponseWriter` and disconnect malicious or overly demanding clients.

By the end of this section, we’ll improve our service’s robustness and understand practical techniques for decoupling the code. We’ll also know how to extend behavior by composing interfaces, such as changing the behavior of existing code without modifying it directly.

### 9.8.1 Safeguarding against denial-of-service attacks
Earlier, we learned that `DecodeJSON` reads the entire request payload without limit. We can use the standard library’s `http.MaxBytesReader` function to prevent large payloads. `MaxBytesReader` wraps an `io.ReadCloser` (e.g., `Request.Body`) and limits the number of bytes we can read. It also takes a `ResponseWriter` to disconnect clients when necessary:
```golang
func MaxBytesReader(ResponseWriter, io.ReadCloser, int64) io.ReadCloser
```
Instead of using this function directly, we’ll add a helper function, as shown in the following listing. Although our function is trivial for now, we’ll improve it to recover the original `ResponseWriter` and have `http.Server` disconnect clients when they reach the limit.
- [Listing 9.25: `MaxBytesReader` helper](../../all-listings/09-composition-patterns/25-maxbytesreader-helper.md)

We added a `MaxBytesReader` helper to our `hio` package. Next, as listing 9.26 shows, we’ll wrap the `Request.Body` with this function to limit the number of bytes the `Shorten` handler can read from the incoming `Request.Body`. When we read more than 4 KB (we set it as follows), our `MaxBytesReader` function will return an error, and we’ll stop reading.
- [Listing 9.26: Integrating `MaxBytesReader`](../../all-listings/09-composition-patterns/26-integrating-maxbytesreader.md)

We integrated our `MaxBytesReader` helper into the `Shorten` handler. `DecodeJSON` reads from the limited `ReadCloser` instead of directly from `Request.Body`. If a client exceeds our 4 KB limit, decoding will fail, but the server may not disconnect the client. (We’ll discover the reason in section 9.8.2.) To test whether our limit works, we can use `curl` to send a request that exceeds the 4 KB limit:
```bash
$ curl localhost:8080/shorten
➥ -d "{\"key\":\"$(printf 'a%.0s' {1..5000})\"}" #1
HTTP/1.1 400 Bad Request
Content-Type: text/plain; charset=utf-8
X-Content-Type-Options: nosniff
. . .

decoding: reading: http: request body too large: bad request
```
#1 We send 5,000 bytes plus a 10-byte JSON part {“key”:””}.

Now our handler effectively rejects oversize payloads. There’s another issue, however: clients can keep sending data even after the handler stops reading. To handle this situation properly, we need to recover the original ResponseWriter and disconnect these clients automatically.

### 9.8.2 Unwrapping the original
When clients exceed the limit, `MaxBytesReader` signals the `Server` through the original `ResponseWriter` to disconnect them. But our earlier `Interceptor` wraps the original `ResponseWriter`. Due to this wrapping, the disconnection mechanism no longer works.

We previously faced a similar issue and added an `Unwrap` method to our `Interceptor`. Now we’ll use that `Unwrap` method to retrieve the original `ResponseWriter` and restore the disconnection mechanism. We can’t control future middleware that could add more wrappers around the original `ResponseWriter`, however. To peel off these potential wrappings, we’ll modify our `MaxBytesReader` helper to unwrap `ResponseWriters`, as the next listing shows.
- [Listing 9.27: Unwrapping `ResponseWriter`s](../../all-listings/09-composition-patterns/27-unwrapping-responsewriters.md)

We’ve declared an interface to extract the `Unwrap` method from the underlying type of the next `ResponseWriter`. Our helper removes any middleware layers and tries to leave us with the original `ResponseWriter`. Passing the original to `MaxBytesReader` instructs the `Server` to disconnect clients when they reach our limit. If the body size exceeds 4,096 bytes, the `Server` stops reading the body and closes the connection, which protects the service from excessive resource use caused by oversize requests, improving server reliability:
```bash
$ curl localhost:8080/shorten
➥ -d "{\"key\":\"$(printf 'a%.0s' {1..5000})\"}" 
HTTP/1.1 400 Bad Request
Connection: close
Content-Type: text/plain; charset=utf-8
. . .

decoding: reading: http: request body too large: bad request
```
Now we get a `Connection: close` message. Also, `http.Server` disconnects the client, preventing the client from using the same connection. This disconnection mechanism works similarly to the unwrapping technique we used earlier. Visit https://mng.bz/wZ5q and https://mng.bz/mZRr for details if you’re curious.
