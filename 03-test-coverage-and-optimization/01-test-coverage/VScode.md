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
  * `${workspaceFolder}`: `03-test-coverage-and-optimization/01-test-coverage`
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

 
# 3.1 Test coverage
## 3.1.1 Measuring test coverage
We can measure the test coverage of the url package by using the `coverprofile` flag:
```bash
$ go test ./url -coverprofile cover.out #1
ok      github.com/inancgumus/gobyexample/url   0.699s  coverage: 100.0% of statements
```
#1 Measures the test coverage of the url package and outputs the result to cover.out


The tool analyzes the `url` package’s code and saves the resulting test coverage profile to the `cover.out` file in the same directory. Our tests have a high coverage rate: 87.5%. Let’s find out what parts of our code the tests cover by feeding this profile into the coverage tool:
```bash
$ go tool cover -html=cover.out
```
> [!NOTE]AVOIDING THE BROWSER
> If we want to avoid the browser and see the function-by-function coverage report from the command line, we can use the `func` flag as follows (methods are functions too):
> ```bash
> $ go tool cover -func=cover.out
> url/url.go:17:   Parse           87.5%
> url/url.go:36:   String          100.0%
> total:           (statements)    87.5%
> ```

Now let’s check the coverage percentage without a profile file:
```bash
$ go test ./url -cover
coverage: 100.0% of statements
```

### 3.1.3 100% test coverage != bug-free code
#### THE EMPTY SCHEME BUG
We add the method `func TestParseError(t *testing.T) ` of [original URL Test Code](../../02-idioms-and-testing/06-example-tests/url/url_test.go).The test belong subtest.
```bash
$ go clean -testcache
$ go test ./url -cover -count=1
--- FAIL: TestParseError (0.00s)
    --- FAIL: TestParseError/empty_scheme (0.00s)
        url_test.go:123: Parse("://github.com") err=nil; want an error
FAIL
coverage: 100.0% of statements
FAIL    github.com/inancgumus/gobyexample/url   0.058s
FAIL
```
> [!NOTE]
> `-count=1` means that does not use cache.
* [Original code](../../02-idioms-and-testing/06-example-tests/url/url.go#L19C2-L19C5)
* [Fix for empty schemes](./url/url.go#L18C2-L18C15)
* [Description](../../all-listings/03-test-coverage-and-optimization/03-fix-for-empty-schemes.md)

#### THE NIL RECEIVER BUG
* [Original code of `url.go`](../../02-idioms-and-testing/06-example-tests/url/url.go#L35C1-L38C1)
* We adjsut the method `func TestURLString(t *testing.T)` of [original `url_test.go`](../../02-idioms-and-testing/06-example-tests/url/url_test.go#L20C1-L32C1).The test belong [subtest](./url/url_test.go#L20C1-L45C1) of `url_test.go`.
```bash
$ go test ./url -v    
=== RUN   TestParse
--- PASS: TestParse (0.00s)
=== RUN   TestURLString
=== RUN   TestURLString/nil
--- FAIL: TestURLString (0.00s)
    --- FAIL: TestURLString/nil (0.00s)
panic: runtime error: invalid memory address or nil pointer dereference [recovered]
        panic: runtime error: invalid memory address or nil pointer dereference
[signal 0xc0000005 code=0x0 addr=0x0 pc=0xb88dff]

goroutine 8 [running]:
```
> [!NOTE] 
> `addr=0x0` is a `nil` pointer that does not point to a valid memory location.


* [Testing string with a nil pointer](../../all-listings/03-test-coverage-and-optimization/04-testing-string-with-a-nil-pointer.md)
* [Fix for a nil pointer](../../all-listings/03-test-coverage-and-optimization/05-fix-for-a-nil-pointer.md)
* [Fix for nil receiver bug](./url/url.go#L36C1-L52C1)

#### THE EMPTY URL BUG
We adjust the method `func TestURLString(t *testing.T) ` of [original URL Test Code](../../02-idioms-and-testing/06-example-tests/url/url_test.go#L20C1-L32).The test belong subtest.
```bash
$ go test ./url                  
--- FAIL: TestURLString (0.00s)
    --- FAIL: TestURLString/empty (0.00s)
        url_test.go:41: 
            got  ":///"
            want ""
            for  &url.URL{Scheme:"", Host:"", Path:""}
FAIL
FAIL    github.com/inancgumus/gobyexample/url   0.055s
FAIL
```
* [Original code of `url.go`](../../all-listings/03-test-coverage-and-optimization/05-fix-for-a-nil-pointer.md)
* We adjsut the method `func TestURLString(t *testing.T)` of [original `url_test.go`](../../02-idioms-and-testing/06-example-tests/url/url_test.go#L20C1-L32C1).The test belong [subtest](./url/url_test.go#L20C1-L45C1) of `url_test.go`.
```bash

``` 
* [Testing string with an empty url](../../all-listings/03-test-coverage-and-optimization/06-testing-string-with-an-empty-url.md)
* [Reassembling a url](../../all-listings/03-test-coverage-and-optimization/07-reassembling-a-url.md) 