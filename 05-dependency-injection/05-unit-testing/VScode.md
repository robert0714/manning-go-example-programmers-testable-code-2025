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
  * `${workspaceFolder}`: `05-dependency-injection\05-unit-testing`
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
## 5.5 Unit testing
- [Listing 5.8: Adding a shared test case type](../../all-listings/05-dependency-injection/08-adding-a-shared-test-case-type.md)
- [Listing 5.9: Testing with valid flags](../../all-listings/05-dependency-injection/09-testing-with-valid-flags.md)
  


```bash
$ go test ./hit/cmd/hit -run=TestParseArgs -v
--- PASS: TestParseArgsValidInput
   --- PASS: TestParseArgsValidInput/all_flags
--- PASS: TestParseArgsInvalidInput
    --- PASS: TestParseArgsInvalidInput/n_syntax
    --- PASS: TestParseArgsInvalidInput/n_negative
    --- PASS: TestParseArgsInvalidInput/n_zero
```