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
  * `${workspaceFolder}`: `07-responsive-and-efficient-programs\05-testing`
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
## 7.5 Testing
We’ve seen that `Client` separates responsibilities by having a `RoundTripper` interface field called `Transport`. This field allows us to use any `RoundTripper` implementation to process HTTP requests and responses. In this section, we’ll implement this interface for testing. Testing with a fake `RoundTripper` shows how easily we can test code that depends on single-method interfaces by implementing these interfaces with simple function types.
### 7.5.1 Satisfying RoundTripper
Send takes a `Client` to send HTTP requests and receive responses. To modify its request and response-handling behavior, we can pass a fake `RoundTripper` to the Client’s `Transport` field. First, we’ll revisit the `RoundTripper` interface to refresh our memory:
```golang
type RoundTripper interface {
    RoundTrip(*http.Request) (*http.Response, error)
}
```
Unlike most other languages, Go allows us to satisfy an interface with a function type. We don’t always have to use a struct. A function type can satisfy `RoundTripper` and turn a regular function into a `RoundTripper` implementation. Then we can set that function to the Transport field as a `RoundTripper`, pass the `Client` to `Send`, and test `Send`’s behavior.

Also, we don’t have to state that our function type implements the `Round­Tripper` interface explicitly. Go can tell automatically whether a type meets the interface requirements by looking at its methods. Implementing the `RoundTrip` method for this function type is enough. The next listing shows the `roundTripperFunc` function type that satisfies `RoundTripper`.

- [Listing 7.6: Satisfying `RoundTripper`](../../all-listings/07-responsive-and-efficient-programs/06-satisfying-roundtripper.md)

We declared a function type matching the signature of the `RoundTrip` method. Then we attached a `RoundTrip` method that calls its receiver: a `roundTripFunc` that takes a `*Request` and returns a `*Response`. Let’s look at an example. First, we need a function that matches the signature of the `RoundTrip` method. When we have that function (or a closure), we can convert it to a `roundTripperFunc`:
```golang
rt := roundTripperFunc(  #1
    func(r *http.Request) (*http.Response, error) { #2
        . . . #3
    },
)
```
#1 Converts the closure function to a roundTripperFunc   
#2 A closure with a matching signature of roundTripperFunc   
#3 We can process the *Request and return a *Response here.   

This `rt` variable’s type is `roundTripperFunc`, and its value is a function pointer that points to the closure we use. Calling `RoundTrip` on `rt` (i.e., `rt.RoundTrip(. . .)`) calls that closure. Because the `rt`’s type satisfies `RoundTripper`, we can set it to the `Transport` field:
```golang
client := &http.Client{
    Transport: rt,  #1
}
```
#1 Sets rt as the http.Client’s RoundTripper   

This `Transport` field’s type is `RoundTripper`. We can set rt to the field because the `roundTripper` type satisfies the `RoundTripper` interface. When we send a request using this `client`, it passes a `*Request` to our `rt.RoundTrip()` and receives a `*Response`.

In short, any concrete type can implicitly implement an interface. `roundTripperFunc` is a function type that does not explicitly specify that it implements `RoundTripper`. Instead, it solely implements a `RoundTrip` method. Unlike most other programming languages, this approach leads to decoupled types and eliminates type hierarchies.

> [!NOTE]
> As we’ll learn in chapter 8, this pattern (testing with a function type) is idiomatic and used frequently in Go. `http.HandlerFunc` satisfies the `http.Handler` interface, for example, making it convenient to turn a regular function with a right signature into a `Handler` that can serve HTTP requests.

### 7.5.2 Testing with a RoundTripper
- [Listing 7.7: Testing with a fake `RoundTripper`](../../all-listings/07-responsive-and-efficient-programs/07-testing-with-a-fake-roundtripper.md)

```bash
$ go test ./hit -run=TestSendStatusCode
ok      github.com/inancgumus/gobyexample/hit   0.194s
```