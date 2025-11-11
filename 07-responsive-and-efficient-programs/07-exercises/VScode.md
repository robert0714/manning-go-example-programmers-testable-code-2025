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
  * `${workspaceFolder}`: `07-responsive-and-efficient-programs\07-exercises`
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
## 7.7 Exercises
1. As explained in section 7.2.4, some stages may never stop if users don’t receive all the pipeline results from the iterator. Pass the `Context` to all stages. Stop each stage when the `Context` is canceled, allowing each stage to stop even if users don’t consume all the results. You can find the solution in the book’s GitHub repository at https://mng.bz/EwOO.
1. In runHit, pass a `Context` from `WithTimeoutContext` to `NotifyContext`.
1. Satisfy the `Reader` interface with a type that reads from a `[]byte`.
1. Satisfy the `Writer` interface by implementing a type that counts the number of bytes written to it. Provide a field to retrieve the number of bytes written.
1. Call `Copy` with the two types you created in the preceding two exercises.
1. Use `io.ReadAll` in Send instead of `io.Copy`, and benchmark the difference.
1. Remove `response.Body.Close` from `Send`, and benchmark the difference.
1. Add a `hit.Options.Client` for setting the `http.Client` used by the HIT client.
1. Declare a `LogRoundTripper` type that implements `RoundTripper` and takes another `RoundTripper` (e.g., `http.DefaultTransport`). It should log incoming request URLs and delegate request and response handling to the given `RoundTripper`. Set this new `RoundTripper` through the HIT client’s new `hit.Options.Client` to test it.
1. Add more tests and benchmarks for the hit package. Verify the edge case conditions for the `SendN` function, for example, such as returning different HTTP status codes.
1. Write a test that turns off the `dryRun` mode and tests the HIT tool by running it against a test HTTP server, passing it flags, and checking the results.

- [Exercise 1: Solution](../../all-listings/07-responsive-and-efficient-programs/09-exercise-1-solution.md)