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
  * `${workspaceFolder}`: `02-idioms-and-testing\05-subtests`
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

 
# 2.5 Subtests
On each iteration, we call the Run method to run a subtest. Each subtest gets a unique name and a closure that wraps the test logic that runs a test case. Because closures can access their surroundings, we can continue using [`tt.uri`](../05-subtests/url/url_test.go#L100C4-L100C18) and [`tt.want`](../05-subtests/url/url_test.go#L104C4-L104C13).

See [codes](../05-subtests/url/url_test.go#L95C1-L109C1). 

> [!NOTE]DEFINITION 
> `Closures` are functions that retain access to their surrounding scope, such as variables, even after their outer function returns.

> [!NOTE] 
> Similar to a test function, each subtest function runs in a new goroutine.
 
## 2.5.3 Understanding the role of specific T pointers
[The `testing` package’s `Run` function delivers a new `*T` to each closure so that each subtest can call `Fatalf` and `Errorf` only to report failure for itself](../05-subtests/url/url_test.go#L99C3-L99C28). That’s why even if one subtest fails with `Fatalf`, other tests continue.

> [!NOTE]WARNING 
> Each `*T` is specific to a test function, making the test function independent. Never share a test function’s `*T` with another test function (including subtests). Doing so could lead to unpredictable behavior and compromise the validity of overall test results.

> [!TIP] DEEP DIVE: CLOSURES AND MEMORY MANAGEMENT
> `Closures` are functions that can capture and hold pointers to variables from their surrounding scope even after their outer function returns. This means they can extend the lifetime of these variables beyond the function that created them. The garbage collector cannot reclaim memory for variables still referenced by actively referenced closures. If we keep closures around too long or share them improperly, we risk memory leaks, so we should be extra careful when sharing closures with other parts of our programs.


### 2.5.4 Running subtests
Use [code](../04-table-driven-testing/url/url.go) .

The parent test `TestParseSubtests` will run the subtests sequentially and wait for them to complete:
```bash
$ go test ./url -v -run=TestParseSubtests$
=== RUN   TestParseSubtests
=== RUN   TestParseSubtests/with_data_scheme
    url_test.go:102: Parse("data:text/plain;base64,R28gYnkgRXhhbXBsZQ==") err = missing scheme, want <nil>
=== RUN   TestParseSubtests/full
=== RUN   TestParseSubtests/without_path
--- FAIL: TestParseSubtests (0.00s)
    --- FAIL: TestParseSubtests/with_data_scheme (0.00s)
    --- PASS: TestParseSubtests/full (0.00s)
    --- PASS: TestParseSubtests/without_path (0.00s)
FAIL
FAIL    github.com/inancgumus/gobyexample/url   0.386s
FAIL
```
Moreover, because subtests have names, we see them in the test report, which enhances readability and maintainability, allowing us to understand the test’s purpose at a glance. As with regular tests, we can use their names to run individual subtests selectively:
```bash
$ go test ./url -v -run=TestParseSubtests/full$
=== RUN   TestParseSubtests
=== RUN   TestParseSubtests/full
--- PASS: TestParseSubtests (0.00s)
    --- PASS: TestParseSubtests/full (0.00s)
PASS
ok      github.com/inancgumus/gobyexample/url   0.051s
```
This time, the parent test passed because the subtest also passed. Although the parent test calls `Run` for all our subtests to run them, the `testing` package intentionally skips them because of our filter and runs only the `TestParseSubtests/full` subtest.


### 2.5.5 Fixing the code
To fix this problem,[ as the next listing shows](./url/url.go#L18C30-L26C46), we improve `Parse` to interpret both opaque (e.g., `data:...`) and regular URLs (e.g., `https://...`). We use string slicing to detect URL types.

Instead of cutting URLs at `"//"`, we do it at `":"` to be compatible with opaque and regular URLs. Let’s see what our variables would look like for both URL schemes:
* Opaque—`rawURL="data:text/plain", scheme="data", rest="text/plain"`
* Regular—`rawURL="https://github.com/inancgumus", scheme="https", rest="//github.com/inancgumus"`
> [!NOTE]
>  A URL is considered opaque if the part after the scheme doesn’t have a slash.

When the scheme is parsed, `Parse` stops if it encounters an opaque URL, returning a `URL` with only the scheme; otherwise, it proceeds to parse the host and path from `rest[2:]`. This string slicing omits the first 2 bytes. `"//github.com"` becomes `"github.com"`, for example.

As [appendix C](../../all-listings/ac-arrays-slices-and-maps/) explains, a `slice` is a view of a fixed array. Slicing returns a slice that sees all or some part of its underlying array. Similarly, slicing a string returns a string without copying the underlying byte data from its array. Also, `rest[2:]` is a shortcut for typing `rest[2:len(rest)]`. Similarly, `rest[:2]` is a shortcut for typing `rest[0:2]`.

Strings can be sliced because they are inherently byte slices. Be careful when slicing strings, however, because the underlying bytes might be used for Unicode encoding (multibyte code points). Slicing such strings might be dangerous because a single character can span multiple bytes. Read more about strings at https://go.dev/blog/strings. After adjusting Parse, we should rerun our tests to verify the fix:
```bash
$ go test ./url -v -run=TestParseSubtests$
--- PASS: TestParseSubtests
    --- PASS: TestParseSubtests/with_data_scheme
    --- PASS: TestParseSubtests/full
    --- PASS: TestParseSubtests/without_path
```

With these changes, `Parse` can interpret newer and traditional URL formats correctly. Although this example is good progress toward a robust parser, our focus will remain on testing rather than creating a full-fledged parser that can handle any URL format.

To prevent confusion and distraction, I didn’t explain that tests are also subtests. Internally, the `testing` package runs top-level tests (tests whose names start with `Test`) using the `Run` method in a new goroutine. Subtests are hierarchical, and one subtest can run another.