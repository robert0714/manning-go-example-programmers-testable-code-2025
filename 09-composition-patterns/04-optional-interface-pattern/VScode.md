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
  * `${workspaceFolder}`: `09-composition-patterns\04-optional-interface-pattern`
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

 
# 9 Composition patterns  
## 9.4 Optional interface pattern
We implemented the `Interceptor` type to embed `ResponseWriter` and capture status codes. In this section, we’ll learn more about field embedding and discover type assertions. We’ll also see how dangerous field embedding interfaces can be in real-world usage if we’re not careful. Finally, we’ll look at modern solutions for these issues specific to our case.

### 9.4.1 Type asserting for optional functionality
Because the Go standard library cannot extend existing interfaces like `ResponseWriter` without breaking Go 1 backward compatibility, the http package introduced optional interfaces over time. These interfaces allow a `ResponseWriter`’s underlying types to provide extra features that the `ResponseWriter` does not. Here are a few common examples:
* `http.Flusher` enables sending buffered response data to clients immediately.
* `http.Pusher` enables HTTP/2 server push functionality.
* `http.Hijacker` takes over the underlying TCP connection.

> [!NOTE]
> Concrete types behind interfaces may have additional methods.

Suppose that a `ResponseWriter`’s underlying type implements some or all of these interfaces. In that case, handlers can detect their existence using type assertion and do sophisticated things like stream downloads or hijack the TCP connection for a WebSocket. Here’s how the http package declares the `Flusher` interface, for example:
```golang
type Flusher interface {
  // Flush sends any buffered data to the client.
  Flush()
}
```
Handlers can detect the `Flush` method in the underlying type using a type assertion:
```golang
func MyHandler(w http.ResponseWriter, r *http.Request) {
  if flusher, ok := w.(http.Flusher); ok {  #1
    flusher.Flush()  #2
  }
}
```
#1 Returns true if the underlying type of w has a Flush method   
#2 The flusher variable’s type is http.Flusher, and it has a Flush method.   

We call `Flush` after detecting that the underlying `ResponseWriter` has a `Flush` method. Otherwise, the `ok` variable would be `false`, and our handler wouldn’t call the method.

Now that we’ve seen type assertions, let’s return to our program to discuss embedding issues.

Go promotes the `ResponseWriter`’s methods to our `Interceptor` because we embed it. But methods like Flush, from the original underlying type we can assign at runtime, do not exist in our `Interceptor`. When we pass the `Interceptor` as a `ResponseWriter` to handlers, they won’t have access to the Flush method because it’s not part of our `Interceptor` type (nor the `ResponseWriter` interface), but rather the underlying type’s method.

As a result, handlers that want to use the `Flush` method revert to other behavior, thinking that flushing isn’t supported even if the original underlying type has the `Flush` method. We can solve this problem by adding an `Unwrap` method to the `Interceptor` that handlers can call to expose the original underlying type.

### 9.4.2 Unwrapping all the way down
We know that field embedding an interface can hide the underlying type’s methods. This issue is more complicated than it seems. The underlying type of a `ResponseWriter` can be another `ResponseWriter`, and it can go like that—turtles all the way down. Eventually, there may be an underlying type with `Flush`, but it may be buried in this `ResponseWriter` chain.

We may have to look for the `Flush` method recursively, perhaps from the outermost `ResponseWriter` in the `ResponseWriter` chain to the innermost `ResponseWriter` that wraps the original one. Another issue, however, is that the `ResponseWriter` interface doesn’t support recursive searching of what wraps it. But we can introduce another optional interface with an Unwrap method and use type assertions to access that method. That interface looks as follows:
```golang
type Unwrapper interface { #1
  Unwrap() http.ResponseWriter
}
```
#1 An interface we can use to extract the underlying ResponseWriter from another   

Introducing another optional interface would make everything more complicated, so Go 1.20 introduced a concrete type to look for optional methods like `Flush` recursively. This type is `ResponseController`, and it can search for optional interfaces recursively in a `ResponseWriter` chain:
```golang
func MyHandler(w http.ResponseWriter, r *http.Request) {
  err := http.NewResponseController(w).Flush()  #1
  if errors.Is(err, http.ErrNotSupported) { #2
      . . .
  }
}
```
#1 Calls Flush if it detects that this ResponseWriter or its wrappers support(s) flushing   
#2 Returns ErrNotSupported if neither this ResponseWriter nor its wrappers has a Flush method   

The `NewResponseController` function returns a new `ResponseController`. Calling `Flush` recursively searches for a `Flush` method in the `ResponseWriter` chain. If none of the `ResponseWriter` wrappers has the Flush, that call returns `ErrNotSupported`; otherwise, it calls the original Flush method on the first `ResponseWriter` that supports it.

Unlike `ResponseWriter`, `ResponseController` is a concrete type that can evolve with new methods without breaking existing implementations. It eliminates the obscurity of optional interfaces, makes new functionality discoverable in the documentation, and allows checking for optional functionality through `ResponseWriter` with standard error handling. See https://pkg.go.dev/net/http#ResponseController for details.

This approach shows how to extend an interface without changing the interface itself and breaking existing code or relying on fragile type assertions. By wrapping the original interface with a concrete type like `ResponseController`, we isolate type assertions inside that type. This type can provide new methods and internally handle underlying optional behaviors. Such a pattern keeps existing code intact while supporting new functionality. We’ll dive into a similar pattern, the driver pattern, in chapter 10.

### 9.4.3 Unwrap
Because our Interceptor type doesn’t have an `Unwrap` method and calling the `Response­Controller.Flush` method would return an error, in the next listing, we add the `Unwrap` method to allow `ResponseController` to access the underlying `ResponseWriter`.

- [Listing 9.9: Unwrap](../../all-listings/09-composition-patterns/09-unwrap.md)

Adding `Unwrap` ensures that the `Interceptor` doesn’t block access to optional interfaces like `Flusher` and avoids breaking handlers that depend on them. A handler that streams data to a client might call `Flush` to send buffered chunks, for example. Without implementing `Unwrap`, such handlers would fail if we pass them the `Interceptor`.

> [!TIP]
> Wrapping a `ResponseWriter` multiple times is common in middleware. `Unwrap` lets us unearth the underlying ones to access potential optional behavior through them.

Supporting `Unwrap` is helpful but doesn’t guarantee that we won’t break existing code. Older code may rely directly on type assertions to check whether optional interfaces are supported. Such code can break unless it uses a type assertion to call `Unwrap` and access the underlying wrappers.

We discovered that embedding `ResponseWriter` can hide optional interfaces like `Flusher`. We also adopted the modern approach of implementing an `Unwrap` method to allow modern handlers to unearth and use optional functionality when it exists. In section 9.8, we’ll use this `Unwrap` method to stop malicious clients from overwhelming our HTTP service.