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
  * `${workspaceFolder}`: `03-test-coverage-and-optimization\04-parallel-testing`
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

 
# 3.4 Parallel testing
* [Parallel tests](../../all-listings/03-test-coverage-and-optimization/11-parallel-tests.md)
 
We have two parallel tests (`TestParallelOne` and `TestParallelTwo`) and one sequential test (`TestSequential`). Go runs the sequential test first and then the parallel ones:
```bash
$ go test ./url -run="Parallel|Sequential" -v
=== RUN   TestParallelOne
=== PAUSE TestParallelOne  #1
=== RUN   TestParallelTwo
=== PAUSE TestParallelTwo  #1
=== RUN   TestSequential   #2
--- PASS: TestSequential (0.00s)
=== CONT  TestParallelOne  #3
=== CONT  TestParallelTwo  #3
--- PASS: TestParallelTwo (5.00s)
--- PASS: TestParallelOne (5.00s)
PASS
ok      github.com/inancgumus/gobyexample/url   5.055s
```
#1 Pauses the parallel tests   
#2 Runs the sequential test   
#3 Resumes the parallel tests  

```bash
$ go test ./url -run="Parallel|Sequential" -v  -parallel 1
=== RUN   TestParallelOne
=== PAUSE TestParallelOne
=== RUN   TestParallelTwo
=== PAUSE TestParallelTwo
=== RUN   TestSequential
--- PASS: TestSequential (0.00s)
=== CONT  TestParallelOne
=== CONT  TestParallelTwo
--- PASS: TestParallelOne (5.00s)
--- PASS: TestParallelTwo (5.00s)
PASS
ok      github.com/inancgumus/gobyexample/url   10.055s
```
Each test ran sequentially, doubling the total run time to 10 seconds. This is the default `testing` package behavior when we don’t mark tests with the `Parallel` method. 

By default, the maximum level of parallelism is set to the number of CPU cores available on the machine or the number allocated to a virtual machine—effectively, the logical CPU count.

> [!TIP]
> We can set the `-parallel` flag to any number, but setting it to a number larger than the CPU cores can backfire and increase the test runtime instead of reducing it. Adjust the level of parallelism depending on the environment or available resources.

## 3.4.2 Running subtests in parallel
* [Parallel subtests](../../all-listings/03-test-coverage-and-optimization/12-parallel-subtests.md)
```bash
$ go test ./url -run=Query -v
=== RUN   TestQuery
=== PAUSE TestQuery
=== CONT  TestQuery
=== RUN   TestQuery/byName
=== PAUSE TestQuery/byName
=== RUN   TestQuery/byInventory
=== PAUSE TestQuery/byInventory
=== CONT  TestQuery/byName
=== CONT  TestQuery/byInventory
--- PASS: TestQuery (0.00s)
    --- PASS: TestQuery/byInventory (5.00s)
    --- PASS: TestQuery/byName (5.00s)
PASS
ok      github.com/inancgumus/gobyexample/url   5.054s
```
## 3.4.3 Detecting data races
* [Date race](../../all-listings/03-test-coverage-and-optimization/13-data-race.md)
There are significant issues with these tests and the code:
* Tests depend on one another.
* The `counter` is not reset between tests.
* The `counter` is not concurrent-safe.
* 
Recall that different goroutines run different tests. Because `incr` increments a package-level variable `counter`, and the tests are running in parallel, this should cause a data race issue. We can catch a potential data race by enabling Go’s race detector with the `-race` flag.

The race detector might not catch every data race, however, so we improve the hit ratio by setting the `count` flag to an arbitrarily high number:
```bash
$ go test ./url -run="TestIncr" -race -count=10
==================
WARNING: DATA RACE
Read at 0x000140429ea0 by goroutine 8:
  github.com/inancgumus/gobyexample/url.incr()
      D:/Data/workspaces/golang/manning-go-example-programmers-testable-code-2025/03-test-coverage-and-optimization/04-parallel-testing/url/parallel_test.go:35 +0x3b
  github.com/inancgumus/gobyexample/url.TestIncr.func1()
      D:/Data/workspaces/golang/manning-go-example-programmers-testable-code-2025/03-test-coverage-and-optimization/04-parallel-testing/url/parallel_test.go:41 +0x2f
  testing.tRunner()
      D:/Data/SDK/golang/go1.24.9.windows-amd64/src/testing/testing.go:1792 +0x1dc
  testing.(*T).Run.gowrap1()
      D:/Data/SDK/golang/go1.24.9.windows-amd64/src/testing/testing.go:1851 +0x44

Previous write at 0x000140429ea0 by goroutine 9:
  github.com/inancgumus/gobyexample/url.incr()
      D:/Data/workspaces/golang/manning-go-example-programmers-testable-code-2025/03-test-coverage-and-optimization/04-parallel-testing/url/parallel_test.go:35 +0x53
  github.com/inancgumus/gobyexample/url.TestIncr.func2()
      D:/Data/workspaces/golang/manning-go-example-programmers-testable-code-2025/03-test-coverage-and-optimization/04-parallel-testing/url/parallel_test.go:49 +0x2f
  testing.tRunner()
      D:/Data/SDK/golang/go1.24.9.windows-amd64/src/testing/testing.go:1792 +0x1dc
  testing.(*T).Run.gowrap1()
      D:/Data/SDK/golang/go1.24.9.windows-amd64/src/testing/testing.go:1851 +0x44

Goroutine 8 (running) created at:
  testing.(*T).Run()
      D:/Data/SDK/golang/go1.24.9.windows-amd64/src/testing/testing.go:1851 +0x8f8
  github.com/inancgumus/gobyexample/url.TestIncr()
      D:/Data/workspaces/golang/manning-go-example-programmers-testable-code-2025/03-test-coverage-and-optimization/04-parallel-testing/url/parallel_test.go:39 +0x44
  testing.tRunner()
      D:/Data/SDK/golang/go1.24.9.windows-amd64/src/testing/testing.go:1792 +0x1dc
  testing.(*T).Run.gowrap1()
      D:/Data/SDK/golang/go1.24.9.windows-amd64/src/testing/testing.go:1851 +0x44

Goroutine 9 (running) created at:
  testing.(*T).Run()
      D:/Data/SDK/golang/go1.24.9.windows-amd64/src/testing/testing.go:1851 +0x8f8
  github.com/inancgumus/gobyexample/url.TestIncr()
      D:/Data/workspaces/golang/manning-go-example-programmers-testable-code-2025/03-test-coverage-and-optimization/04-parallel-testing/url/parallel_test.go:47 +0x64
  testing.tRunner()
      D:/Data/SDK/golang/go1.24.9.windows-amd64/src/testing/testing.go:1792 +0x1dc
  testing.(*T).Run.gowrap1()
      D:/Data/SDK/golang/go1.24.9.windows-amd64/src/testing/testing.go:1851 +0x44
==================
--- FAIL: TestIncr (0.00s)
    --- FAIL: TestIncr/twice (0.00s)
        parallel_test.go:52: counter = 2, want 3
        testing.go:1490: race detected during execution of test
    --- FAIL: TestIncr/once (0.00s)
        parallel_test.go:43: counter = 3, want 1
        testing.go:1490: race detected during execution of test
==================
WARNING: DATA RACE
Read at 0x000140429ea0 by goroutine 12:
  github.com/inancgumus/gobyexample/url.incr()
      D:/Data/workspaces/golang/manning-go-example-programmers-testable-code-2025/03-test-coverage-and-optimization/04-parallel-testing/url/parallel_test.go:35 +0x3b
  github.com/inancgumus/gobyexample/url.TestIncr.func2()
      D:/Data/workspaces/golang/manning-go-example-programmers-testable-code-2025/03-test-coverage-and-optimization/04-parallel-testing/url/parallel_test.go:49 +0x2f
  testing.tRunner()
      D:/Data/SDK/golang/go1.24.9.windows-amd64/src/testing/testing.go:1792 +0x1dc
  testing.(*T).Run.gowrap1()
      D:/Data/SDK/golang/go1.24.9.windows-amd64/src/testing/testing.go:1851 +0x44

Previous write at 0x000140429ea0 by goroutine 11:
  github.com/inancgumus/gobyexample/url.incr()
      D:/Data/workspaces/golang/manning-go-example-programmers-testable-code-2025/03-test-coverage-and-optimization/04-parallel-testing/url/parallel_test.go:35 +0x53
  github.com/inancgumus/gobyexample/url.TestIncr.func1()
      D:/Data/workspaces/golang/manning-go-example-programmers-testable-code-2025/03-test-coverage-and-optimization/04-parallel-testing/url/parallel_test.go:41 +0x2f
  testing.tRunner()
      D:/Data/SDK/golang/go1.24.9.windows-amd64/src/testing/testing.go:1792 +0x1dc
  testing.(*T).Run.gowrap1()
      D:/Data/SDK/golang/go1.24.9.windows-amd64/src/testing/testing.go:1851 +0x44

Goroutine 12 (running) created at:
  testing.(*T).Run()
      D:/Data/SDK/golang/go1.24.9.windows-amd64/src/testing/testing.go:1851 +0x8f8
  github.com/inancgumus/gobyexample/url.TestIncr()
      D:/Data/workspaces/golang/manning-go-example-programmers-testable-code-2025/03-test-coverage-and-optimization/04-parallel-testing/url/parallel_test.go:47 +0x64
  testing.tRunner()
      D:/Data/SDK/golang/go1.24.9.windows-amd64/src/testing/testing.go:1792 +0x1dc
  testing.(*T).Run.gowrap1()
      D:/Data/SDK/golang/go1.24.9.windows-amd64/src/testing/testing.go:1851 +0x44

Goroutine 11 (running) created at:
  testing.(*T).Run()
      D:/Data/SDK/golang/go1.24.9.windows-amd64/src/testing/testing.go:1851 +0x8f8
  github.com/inancgumus/gobyexample/url.TestIncr()
      D:/Data/workspaces/golang/manning-go-example-programmers-testable-code-2025/03-test-coverage-and-optimization/04-parallel-testing/url/parallel_test.go:39 +0x44
  testing.tRunner()
      D:/Data/SDK/golang/go1.24.9.windows-amd64/src/testing/testing.go:1792 +0x1dc
  testing.(*T).Run.gowrap1()
      D:/Data/SDK/golang/go1.24.9.windows-amd64/src/testing/testing.go:1851 +0x44
==================
--- FAIL: TestIncr (0.00s)
    --- FAIL: TestIncr/once (0.00s)
        parallel_test.go:43: counter = 4, want 1
    --- FAIL: TestIncr/twice (0.00s)
        parallel_test.go:52: counter = 6, want 3
        testing.go:1490: race detected during execution of test
--- FAIL: TestIncr (0.00s)
    --- FAIL: TestIncr/once (0.00s)
        parallel_test.go:43: counter = 7, want 1
    --- FAIL: TestIncr/twice (0.00s)
        parallel_test.go:52: counter = 9, want 3
--- FAIL: TestIncr (0.00s)
    --- FAIL: TestIncr/once (0.00s)
        parallel_test.go:43: counter = 10, want 1
    --- FAIL: TestIncr/twice (0.00s)
        parallel_test.go:52: counter = 12, want 3
--- FAIL: TestIncr (0.00s)
    --- FAIL: TestIncr/once (0.00s)
        parallel_test.go:43: counter = 13, want 1
    --- FAIL: TestIncr/twice (0.00s)
        parallel_test.go:52: counter = 15, want 3
--- FAIL: TestIncr (0.00s)
    --- FAIL: TestIncr/twice (0.00s)
        parallel_test.go:52: counter = 17, want 3
    --- FAIL: TestIncr/once (0.00s)
        parallel_test.go:43: counter = 18, want 1
--- FAIL: TestIncr (0.00s)
    --- FAIL: TestIncr/once (0.00s)
        parallel_test.go:43: counter = 19, want 1
    --- FAIL: TestIncr/twice (0.00s)
        parallel_test.go:52: counter = 21, want 3
--- FAIL: TestIncr (0.00s)
    --- FAIL: TestIncr/twice (0.00s)
        parallel_test.go:52: counter = 23, want 3
    --- FAIL: TestIncr/once (0.00s)
        parallel_test.go:43: counter = 24, want 1
--- FAIL: TestIncr (0.00s)
    --- FAIL: TestIncr/twice (0.00s)
        parallel_test.go:52: counter = 26, want 3
    --- FAIL: TestIncr/once (0.00s)
        parallel_test.go:43: counter = 27, want 1
--- FAIL: TestIncr (0.00s)
    --- FAIL: TestIncr/twice (0.00s)
        parallel_test.go:52: counter = 29, want 3
    --- FAIL: TestIncr/once (0.00s)
        parallel_test.go:43: counter = 30, want 1
FAIL
FAIL    github.com/inancgumus/gobyexample/url   0.044s
FAIL
```
Because the detector adds assembly code to the final binary to monitor memory access, it can significantly slow our programs and use more resources. Therefore, we should use it only in test builds or testing environments, not in production deployments.

> [!TIP]
> Because the race detector works on the executed code paths, increasing the test coverage may also detect more data races.