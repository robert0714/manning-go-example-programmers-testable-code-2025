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
  * `${workspaceFolder}`: `08-structuring-packages-and-services\09-extra-not-included-in-the-book`
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
## 8.9 Exercises
Here are some exercises to hone your skills, from easy ones to difficult ones:
1. Remove the mutex from the `link.Shortener`. Inspect what happens if you share a `Shortener` between handlers. Run the server with the `-race` flag, and send requests to the handlers repeatedly to hit a race condition. Hint: to do that, use the `hit` tool we developed in previous chapters.
1. Test the remaining code in the `link` and `link/rest` packages.
1. Add support for updating the destination URL of an existing shortened URL.
4. Add an integer field to `link.Link`, incrementing with each Resolve call. Hint: be careful about race conditions. Use a mutex or an atomic integer (`sync/atomic`).
1. Find a way to prevent potential infinite redirects while resolving links.
1. Make `main` (`linkd`) testable by externalizing dependencies, as in chapter 5.
1. Develop a client API for the `link/rest`. See the next exercise for details.
1. Write a CLI tool that uses the client library. See `cmd/linkc` at https://mng.bz/rZ1D for details.
1. Improve `Shorten` to remove old links from the map. Otherwise, the map could grow so large that it crashes the machine due to out-of-memory errors. Feel free to resurrect your old computer science books to add better validation logic.
1. Write and test a REST API for the HIT client from chapter 7. For a starting point, see https://mng.bz/V920.