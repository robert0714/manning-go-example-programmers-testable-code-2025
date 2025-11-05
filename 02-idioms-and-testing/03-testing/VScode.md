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
  * `${workspaceFolder}`: `02-idioms-and-testing\03-testing`
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

## COMPARING COMPLEX VALUES AND PRINTING DIFFS
Comparing struct values can be tricky. If we have a complex type with many nested fields and pointers to follow, we can use the `reflect` package’s `DeepEqual` method. A much better approach, however, is to use `go-cmp`. We can get the package as follows:
```bash
$ go get github.com/google/go-cmp/cmp
```
Then we can import the cmp package in the test file and compare values like this:
```golang
if diff := cmp.Diff(want, got); diff != "" {
    t.Errorf("Parse(%q) mismatch (-want +got):\n%s", uri, diff)
}
```
The failure message looks like this after the test runs (and is customizable):
```diff
Parse("https://github.com/inancgumus") mismatch (-want +got):
          &url.URL{
                Scheme: "https",
                Host:   "github.com",
        -       Path:  "",
        +       Path:  "inancgumus,
          }
```          
Check out the documentation at https://pkg.go.dev/github.com/google/go-cmp/cmp to learn more.