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
  * `${workspaceFolder}`: `05-dependency-injection\03-decoupling`
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
# 5 Dependency injection 
## 5.3 Decoupling
- [Listing 5.2: Refactoring the `main` function](../../all-listings/05-dependency-injection/02-refactoring-the-main-function.md)
- [Listing 5.3: Setting `FlagSet`'s output](../../all-listings/05-dependency-injection/03-setting-flagsets-output.md)
- [Listing 5.4: Preparing for the HIT client](../../all-listings/05-dependency-injection/04-preparing-for-the-hit-client.md)

### 5.3.4 Demonstration

```bash
$ go run ./hit/cmd/hit https://github.com/inancgumus
 __  __     __     ______
/\ \_\ \   /\ \   /\__  _\
\ \  __ \  \ \ \  \/_/\ \/
 \ \_\ \_\  \ \_\    \ \_\
  \/_/\/_/   \/_/     \/_/

Sending 100 requests to "https://github.com/inancgumus" (concurrency: 1)

```
This confirms that our program still launches, prints the logo, and shows the correct request plan. It defaults to sending 100 requests with a concurrency of 1. Let’s check whether our tool can still handle incorrect flags:
```bash
$ go run ./hit/cmd/hit -n HIT
invalid value "HIT" for flag -n: strconv.ParseInt: parsing "HIT": invalid syntax
usage: hit [options] url
  -c value
        Concurrency level (default 1)
  -n value
        Number of requests (default 100)
  -rps value
        Requests per second
exit status 1
```
This output shows that our flag parser still works correctly. The `n` flag expects a number, but we gave it a string. The program prints a helpful error followed by the usage message. If the `-n` flag were accepted, the tool would have used the provided value instead of the default `100`.