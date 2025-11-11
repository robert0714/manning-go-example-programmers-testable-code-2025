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
  * `${workspaceFolder}`: `07-responsive-and-efficient-programs\02-cancellation-propagation`
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
## 7.2 Cancellation propagation
> [!NOTE]
> `Ctrl+C` sends an interruption signal to the running foreground process, such as the HIT tool, telling it to stop immediately. It’s often handy for halting unresponsive or long-running tasks. Handling this signal enables a process to shut down gracefully.
### 7.2.1 What is Context?
The idiomatic way to cancel such long-running work safely is to use the `context.Context` type.

> [!NOTE]
> `Context` is the idiomatic way to propagate cancellation signals. Before that, gophers used other solutions, leading to code inconsistencies. The `context` package standardized consistent cancellation propagation across projects.

### 7.2.2 Context is like a tree
Because Contexts are hierarchical, we need a root Context to begin with:
```golang
root := context.Background()  #1
```
#1 Returns a root Context   

We can’t cancel a root `Context` or change it to be cancellable because a `Context` is immutable, but we can derive a new cancellable `Context` from it. The fact that a `Context` is immutable makes it easy to reason about its life cycle and propagate it safely.

`Context` is an interface with various implementations, which allows us to chain these implementations easily. We can derive a timeout `Context` like this:
```golang
child, cancel := context.WithTimeout(  #1
    root,  #2
    15*time.Second,  #3
)
```
#1 Returns a new Context and a func() to cancel it   
#2 Derives the timeout Context from the root Context   
#3 Automatically cancels in 15 seconds   

> [!NOTE]
> Always cancel a `Context` to free resources after you finish using it.

We can chain the previous `Context` further, building a tree:
```golang
grandChild, cancel := context.WithTimeout(child, 5*time.Second)
```

This `Context` has a lifetime of 5 seconds before it is canceled. Notice that it cannot surpass its parent `Context`’s lifetime, which is 15 seconds. The cancellation in a parent `Context` also cancels its children, but a child cannot affect its parents.

Functions such as `WithTimeout` and `WithCancel` return an explicit `cancel` function for two reasons:
* Not every `Context` can be canceled.
* An explicit result value requires us to receive it (i.e., assign it to a variable), making the function’s intent clear. If the cancellation function were a method on `Context`, such as `Context.Cancel`, it would be implicit, and we might forget calling that method.


Now that we understand how Context works, let’s look at its most important methods. We can check whether a `Context` is canceled by [receiving from its Done channel](./hit/pipe.go#L26C1-L27C1):
```golang
<-grandChild.Done()  #1
```
#1 Blocks until the Context is canceled   

### 7.2.3 Context in practice
- [Listing 7.1: Cancelable pipeline](../../all-listings/07-responsive-and-efficient-programs/01-cancelable-pipeline.md)
> [!TIP]
> Conventionally, `Context` is passed as the first parameter for consistency. Avoid storing a `Context` in a struct (https://go.dev/blog/context-and-structs).

We used a `select` statement to wait on two channels at the same time. When the `Context` is canceled, the producer stage closes the output channel and stops. Subsequent stages (throttler or dispatcher) detect the channel’s close signal, close their output channels, and stop. If the `Context` is not canceled, the producer stage sends the next message to its output channel. The dispatcher listens to this output channel, receives a `*Request`, and uses it to send an HTTP request to the target HTTP server. We clone this `*Request` to include the `Context` using the `Request` type’s `Clone` method. This way, we propagate cancellation signals to HTTP requests and stop them automatically even if they are in flight after the `Context` is canceled.

> [!NOTE]
> `Request.Clone` makes a shallow copy of the `Request` with a `Context`. It doesn’t clone the `Request.Body`, for example. We’ll learn about `Body` in section 7.3.


### 7.2.4 Deriving a new Context
- [Listing 7.2: Deriving `Context`s](../../all-listings/07-responsive-and-efficient-programs/02-deriving-contexts.md)

### 7.2.5 Is Ctrl+C the end?
- [Listing 7.3: Catching interruption signals](../../all-listings/07-responsive-and-efficient-programs/03-catching-interruption-signals.md)
```bash
$ go run ./hit/cmd/hit -n 1_000_000 -c 100 https://x.com/inancgumus

 __  __     __     ______
/\ \_\ \   /\ \   /\__  _\
\ \  __ \  \ \ \  \/_/\ \/
 \ \_\ \_\  \ \_\    \ \_\
  \/_/\/_/   \/_/     \/_/

Sending 1000000 requests to "https://x.com/inancgumus" (concurrency: 100)
^C  #1
Summary:
    Success:  100%
    RPS:      997.0
    Requests: 25300
    Errors:   0
    Bytes:    253000
    Duration: 25.383s
    Fastest:  100ms
    Slowest:  100ms

error occurred: context canceled #2
exit status 1
```
#1 Pressing Ctrl+C after a while   
#2 Err() returns an error because the NotifyContext is canceled.