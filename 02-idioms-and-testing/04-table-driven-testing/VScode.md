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
  * `${workspaceFolder}`: `02-idioms-and-testing\04-table-driven-testing`
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

 
# 2.4 Table-driven tests
Instead of duplicating this logic for each URL we want to test, the idiomatic approach uses table-driven testing, which separates the test data from logic and reuses that logic to run separate test cases. Table-driven tests make it easy to add new test cases, reduce duplication through reuse, and improve maintainability.

See [codes](../04-table-driven-testing/url/url_test.go#L49C1-L93C1). 

> [!NOTE]
> Check out [appendix C](../../all-listings/ac-arrays-slices-and-maps/) for more information about slices and [appendix D](../../all-listings/ad-object-oriented-programming/) for details about anonymous structs.


[This test](../04-table-driven-testing/url/url_test.go#L81C1-L93C1) is similar to the preceding parsing tests; the most significant difference is that the test logic fetches data from a slice of anonymous structs.

> [!TIP]
> Repeating letters ([`tt`](../04-table-driven-testing/url/url_test.go#L82C8-L91C12)) is a convention that differentiates variables or refers to multiples of something, making it easy to read and type as long as the surrounding context clarifies the use. We use [`tt`](../04-table-driven-testing/url/url_test.go#L82C8-L91C12) for each test case to differentiate from the t variable, for example.


## 2.4.3 Running a specific test
```bash
$ go test ./url -v -run=TestParseTable$
=== RUN   TestParseTable
    run full  #1
    run without_path  #2
--- PASS: TestParseTable
```
> [!TIP]
> The `run` flag runs only the specified tests and supports regular expressions using the RE2 syntax. Visit https://github.com/google/re2/wiki/Syntax for more information.

## 2.4.4 Identifying problems with table-driven tests
This test case should cause our test to call Fatalf and skip the remaining test cases:
```bash
$ go test ./url -v -run=TestParseTable$
=== RUN   TestParseTable
    url_test.go:83: run with_data_scheme
    url_test.go:87: Parse("data:text/plain;base64,R28gYnkgRXhhbXBsZQ==") err = missing scheme, want <nil>
--- FAIL: TestParseTable (0.00s)
FAIL
FAIL    github.com/inancgumus/gobyexample/url   0.052s
FAIL
```
Our test calls [`Fatalf`](../04-table-driven-testing/url/url_test.go#L87C4-L87C50) during the data test case step, causing the remaining test cases to be skipped. Another problem—perhaps a less significant one in our case—is that we can’t selectively run a specific test case using the run flag because we have a single test function.

> [!NOTE]
> In a table-driven test, if one test case encounters a fatal failure (`Fatalf`), it stops the entire test function, preventing the remaining test cases from running.

We’re familiar with table-driven testing:
* Test cases are scenarios with the input and output data we want to test.
* Table-driven tests reuse the same test logic, reducing duplication and improving maintainability and test coverage.
  
Table-driven testing also has downsides:
* We can’t run test cases individually because they run within the same test function.
* A fatal test failure causes the remaining test cases to be skipped.
