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
  * `${workspaceFolder}`: `06-synchronous-apis-for-concurrency\05-integration`
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

 
# 6 Synchronous APIs for concurrency
## 6.5 Integration
### 6.5.1 Printing a summary

- [Listing 6.10: Printing a summary](../../all-listings/06-synchronous-apis-for-concurrency/10-printing-a-summary.md)

### 6.5.2 Integration
- [Listing 6.11: Integrating the HIT client](../../all-listings/06-synchronous-apis-for-concurrency/11-integrating-the-hit-client.md)
### 6.5.3 Demonstration

```bash
go run ./hit/cmd/hit -n 10 #1
➥                     https://x.com/inancgumus  #1

 __  __     __     ______
/\ \_\ \   /\ \   /\__  _\
\ \  __ \  \ \ \  \/_/\ \/
 \ \_\ \_\  \ \_\    \ \_\
  \/_/\/_/   \/_/     \/_/

Sending 10 requests to "https://x.com/inancgumus" (concurrency: 1)

Summary:
    Success:  100%
    RPS:      10.0
    Requests: 10
    Errors:   0
    Bytes:    100
    Duration: 1.004s
    Fastest:  100ms
    Slowest:  100ms
```
#1 Sends 10 requests to the target URL one at a time

Each request among 10 requests takes 100 ms because the `Send` function sleeps for 100 ms (recall `Send` from listing 6.2). The total duration is about 1 second. All requests are successful. `RPS` is `10` because HIT sends 10 requests per second.