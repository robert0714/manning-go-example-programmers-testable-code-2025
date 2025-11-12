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
  * `${workspaceFolder}`: `08-structuring-packages-and-services\06-timeouts`
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
## 8.7 Testing
Testing a handler involves providing it a `Request` and a `ResponseWriter` and observing its response. We’ll start by using `httptest.ResponseRecorder` to capture and inspect a handler’s response. Then we’ll dive into using test helpers.

### 8.7.1 Response recording
Because `ResponseRecorder` is a `ResponseWriter`, we can pass it to a handler to observe the responses it generates. The following example shows passing test-only values to `Health`:
```golang
w := httptest.NewRecorder()  #1
r := httptest.NewRequest(http.MethodGet, "/", nil)  #2
Health(w, r)
```
#1 ResponseRecorder records what the handler responds.   
#2 Instead of returning an error, this method panics if the inputs (e.g., HTTP method) are incorrect.   

We can inspect the handler’s response through w. We could log the handler’s response status code and response body using `*testing.T.Log` like this:
```golang
t.Log(w.Status)         // logs 200
t.Log(w.Body.String())  // logs OK  #1
```
#1 Body is a *bytes.Buffer. Calling String returns the buffer’s accumulated bytes as a string.

To recap, we can pass a `Request` and a `ResponseWriter`, like `ResponseRecorder`, to a handler, observe, and verify that its response matches what we expect.
### 8.7.2 Testing a handler
- [Listing 8.15: Testing a handler](../../all-listings/08-structuring-packages-and-services/15-testing-a-handler.md)
We created a ResponseRecorder to capture the Health handler’s response. Then we called Health and checked whether its status code was StatusOK and the response body contained "OK". Any mismatch will output an error with the actual and expected values. Let’s run the test:
```bash
$ go test ./link/rest -v
--- PASS: TestHealth
```
###　8.7.3 Test helpers
Using `httptest.NewRequest` in tests is more convenient than using `http.NewRequest` because `httptest.NewRequest` panics instead of returning an error. So we don’t need to check for an error after creating a request with `httptest.NewRequest`, but it would be more convenient if it were a test helper that makes tests fail, producing nice error output instead of stopping the whole test run with a panic.
#### IMPLEMENTING A TEST HELPER
Test helpers simplify tests and reduce repetitive code. The following `newRequest` is a test helper that internally calls `NewRequest` to create a new `*Request`.
- [Listing 8.16: Adding a test helper](../../all-listings/08-structuring-packages-and-services/16-adding-a-test-helper.md)
> [!TIP]
>  Using `testing.TB` allows us to use `newRequest` in both regular tests and benchmarks. `testing.TB` is an interface that `*testing.T` and `*testing.B` satisfy.
- [Listing 8.17: Testing with a test helper](../../all-listings/08-structuring-packages-and-services/17-testing-with-a-test-helper.md)

We’ve implemented a test-helper function and used it in our test. By calling `Helper()`, we mark `newRequest` as a test-helper function. When an error occurs, `newRequest` reports it from the line in the calling test (e.g., `TestHealth`) rather than inside `new­Request`, making it easier to locate the exact line where the test failed, making debugging easier.
#### TESTING WITH A TEST HELPER
Next, we’ll force our test helper to fail to see how test helpers can be useful in practice. We do so by passing an emoji as an HTTP method while calling `newRequest`. Suppose that `TestHealth` calls `newRequest`, and `newRequest` encounters the following error:
```bash
$ go test ./link/rest -v
. . .
=== CONT  TestHealth
    health_test.go:15: newRequest() err =
    ➥ net/http: invalid method "💣", want nil
```    
The log shows a line in `TestHealth` rather than a line in `newRequest`:
```golang
func TestHealth(t *testing.T) {
    . . .Health(w, newRequest(. . .)). . .  #1
}
```
#1 Line 15

Instead of showing a line in `newRequest`, where it fails while creating the request, the log shows line 15, which is the line where `TestHealth` called `newRequest` to create a new request. Getting an error report on the exact line of the test that calls the helper makes fixing issues easier and faster. Otherwise, we wouldn’t get the failure’s origin easily.

> [!NOTE]
> Test helpers make tests clearer and reduce duplicated test setup code.

As a final piece of advice, avoid returning errors from test helpers; not doing so goes against their purpose of reducing repetition. If errors are returned, each test that uses the helper will have to handle them. It’s more effective to use the methods from `testing.TB` (e.g., `Error` and `Fatal`) to handle errors and fail the test within the helper.

In this section, we learned effective techniques for testing HTTP handlers. We used a `ResponseRecorder` to capture handler responses. Then we implemented a test helper to create a new `Request` without panic and obtain accurate line information in case of failures.

