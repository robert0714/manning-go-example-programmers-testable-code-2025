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
  * `${workspaceFolder}`: `04-command-line-interfaces\01-groundwork`
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
# 4 Command-line interfaces
```bash
$ ./hit -n 1_000_000 -c 20 http://localhost:8082  #1
 __  __     __     ______
/\ \_\ \   /\ \   /\__  _\
\ \  __ \  \ \ \  \/_/\ \/
 \ \_\ \_\  \ \_\    \ \_\
  \/_/\/_/   \/_/     \/_/

Sending 1000000 requests to "http://localhost:8082" (concurrency: 20)

Summary:
    Success:  100%
    RPS:      97854.73
    Requests: 1000000
    Errors:   0
    Bytes:    12000000
    Duration: 10.22s
    Fastest:  1ms
    Slowest:  5ms
```
#1 The `-n` flag is the number of requests to send. The `-c` flag is the concurrency level.
> [!NOTE]
> As we’ve seen, flags customize a program’s behavior. The `-v` flag in `go test -v`, for example, instructs the tool to print verbose output.

In the HIT tool’s output, we see that the HIT client sends 1 million HTTP requests to an HTTP server. The entire operation finishes in roughly 10 seconds without errors, handling nearly 100,000 HTTP requests per second, with the quickest response taking 1 millisecond.


## 4.1 Groundwork
```bash
go run ./hit/cmd/hit
```
![architecture diagram](./pics/CH04_F01_Gumus.drawio.svg)
Figure 4.1 Separating the CLI tool from the `hit` package for reusability. We could add a REST API and reuse the same `hit` package, for example.

```bash
./hit #1
└── . . .   #1
└── cmd  #2
    └── hit #3
    |   └── config.go  #4
    |   └── config_test.go  #5
    |   └── hit.go  #6
    |   └── hit_test.go  #7
    └── hitd  #8
        └── . . .   #8
```
#1 HIT client (package hit)   
#2 Groups executable commands in a directory (not a package by itself)   
#3 HIT tool (package main)   
#4 HIT tool’s configuration, such as parsing flags   
#5 Tests for the HIT tool’s configuration   
#6 The program’s entry point (func main())   
#7 Tests for the HIT tool    
#8 Example: a REST API server could be here.    

The root directory contains the HIT client for performance measurement and reporting. The files in this directory belong to the `hit` package. Below that directory is a `cmd` directory that contains executable commands, with subdirectories like `hit`, which includes the HIT tool, and `hitd`, which could house a future HTTP server API. Both executables can reuse the same HIT client provided by the `hit` package and print results in the CLI or through the web.

### 4.1.1 Implementing the first version
[Listing 4.1 Printing usage (hit/cmd/hit/hit.go)](./hit/cmd/hit/hit.go)
For the values of these constants, we use backticks instead of double quotes. A backtick [(`)] defines a `raw string literal`. The compiler does not interpret escape characters (such as newlines) in raw strings, making it easy to create multiline strings.

### 4.1.2 Running the first version
```bash
$ go build -o bin/hit.exe ./hit/cmd/hit  #1
```
#1 The `-o` flag instructs go build to produce a `hit` (or `hit.exe` on Windows) executable under the bin directory.