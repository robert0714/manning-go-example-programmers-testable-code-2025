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
  * `${workspaceFolder}`: `08-structuring-packages-and-services\06-timeouts`
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
## 8.6 Timeouts
![architecture diagram](pics/CH08_F08_Gumus.drawio.svg)  

We have two critical server timeout settings: `ReadTimeout` and `IdleTimeout`. These settings protect the server from potential issues caused by slow or unresponsive clients:
* `ReadTimeout` starts a timer when the server accepts a connection and stops when it receives the request, ensuring that the server doesn’t waste time on slow requests.
* `IdleTimeout` sets the total time the server keeps a connection open while waiting for a new request. If no request comes, the server closes the connection.
* 
Although `ListenAndServe` provides a simple way to start a server, it doesn’t allow us to configure timeouts. To enable these protections, we should configure a `Server` as follows.
- [Listing 8.14: Setting server timeouts](../../all-listings/08-structuring-packages-and-services/14-setting-server-timeouts.md)

> [!NOTE]
>  We protect the server from unresponsive or malicious clients. We can also use the standard library’s `http.TimeoutHandler` to protect the server from long-running handlers. See https://pkg.go.dev/net/http#TimeoutHandler for more information.

> [!NOTE]USING HTTPS
> To protect against menace-in-the-middle attacks, we can launch the server using the `ListenAndServeTLS` method (similar to `ListenAndServe`) to listen and respond over HTTPS connections. Visit https://mng.bz/QwZG to learn more.
> 
> Still, it is common to run a server over HTTP and use a reverse proxy to handle HTTPS. A reverse proxy can manage Transport Layer Security (TLS) termination, decrypt HTTPS requests, and pass them as HTTP to the server listening to HTTP. This can simplify both development and HTTPS setup.