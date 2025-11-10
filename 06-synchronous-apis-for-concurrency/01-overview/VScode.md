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
  * `${workspaceFolder}`: `06-synchronous-apis-for-concurrency\01-overview`
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
## 6.1 Overview
![architecture diagram](pics/CH06_F01_Gumus.drawio.svg)   
Figure 6.1 The flow of requests and results. `SendN` uses `Send` to send concurrent HTTP requests to the server and returns an iterator that pushes each `Result` to `Summarize`, which consumes the iterator to produce a `Summary`. Although the figure shows two goroutines, the number depends on the concurrency level configured.

`SendN` initially provides functionality for sending sequential requests. Instead of starting with concurrency, we’ll send requests one after another. Beginning this way keeps the code manageable and straightforward, avoiding unnecessary complexity. When the sequential version is working well, we’ll introduce concurrency.

### 6.1.2 Package API
An API is the contract between the package and its users. Here are some guidelines for designing a
useful API:

* It should be synchronous by default.
* It should avoid unnecessary abstractions.
* It should be straightforward to use.
* It should be composable, allowing users to use it in creative ways.
We’ll follow the preceding guidelines for the hit package. Let’s explore the package’s API by looking at the declaration of the exported items:
```golang
type Result struct  #1
    func Send(*http.Client, *http.Request) Result  #2
    type SendFunc func(*http.Request) Result  #3
type Results iter.Seq[Result]  #4
    func SendN(  #5
        int, *http.Request, Options,   #5
    ) (Results, error)   #5
type Summary struct  #6
    func Summarize(Results) Summary  #7
type Options struct  #8
    func Defaults() Options  #9
``` 
#1 A single result from an HTTP request   
#2 Sends a single HTTP request and returns a performance result   
#3 A function type to specify and customize request processing   
#4 An iterator to iterate over Result values   
#5 Sends multiple requests and returns a Results iterator   
#6 A summary of results   
#7 Summarizes each Result value from the Results iterator into a Summary   
#8 Options to change the HIT client’s behavior (for SendN)   
#9 Returns the default options for convenience   

### 6.1.3 Directory structure
Before implementing hit, let’s get familiar with the directory structure:
```bash
hit  #1
└── hit.go  #2
└── hit_test.go  #3
└── option.go  #4
└── pipe.go  #5
└── result.go  #6
└── cmd
    └── hit  #7
```    
#1 Performance measurement logic (package hit)   
#2 HIT client’s core functionality   
#3 Tests for the HIT client   
#4 Functionality for configuring the HIT client   
#5 Functionality for running a concurrent pipeline   
#6 Functionality for interpreting request results   
#7 HIT CLI tool (package main)   