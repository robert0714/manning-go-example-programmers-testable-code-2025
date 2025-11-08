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
  * `${workspaceFolder}`: `05-dependency-injection\04-cli-tests`
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
## 5.4 CLI tests

- [Listing 5.5: Test environment](../../all-listings/05-dependency-injection/05-test-environment.md)
- [Listing 5.6: Test runner helper](../../all-listings/05-dependency-injection/06-test-runner-helper.md)
- [Listing 5.7: Adding CLI tests](../../all-listings/05-dependency-injection/07-adding-cli-tests.md)

```bash
$ go test ./hit/cmd/hit -v
--- PASS: TestRunValidInput
--- PASS: TestRunInvalidInput
```