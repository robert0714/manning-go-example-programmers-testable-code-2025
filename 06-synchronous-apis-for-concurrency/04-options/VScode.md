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
  * `${workspaceFolder}`: `06-synchronous-apis-for-concurrency\04-options`
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
## 6.4 Options
### 6.4.1 Providing options
- [Listing 6.8: Providing options](../../all-listings/06-synchronous-apis-for-concurrency/08-providing-options.md)
As listing 6.8 shows, we declare the Options type with the following fields:
* `Concurrency` determines how many goroutines to use while sending requests.
* `RPS` limits the number of requests that can be sent per second.
* `Send` allows users to specify a custom SendFunc to use while sending requests. The default request sender remains as the `hit` package’s `Send` function. In chapter 7, we’ll switch to a custom sender as the default for more efficiency.


### 6.4.2 Accepting options
- [Listing 6.9: `SendN` with options](../../all-listings/06-synchronous-apis-for-concurrency/09-sendn-with-options.md)

We’ve made three changes to the SendN function:
* We added an `opts` parameter of type `Options` to the function.
* `SendN` calls `withDefaults` to fill any unset options with defaults.
* `SendN` sends requests using the `Options.Send` function. `SendN` no longer needs to pass an HTTP client to `Send` because the `Options` takes care of setting it.


> [!TIP]
> See appendix F for a deep explanation of this issue. The appendix also discusses an alternative pattern for handling options: Rob Pike’s self-referential functions.