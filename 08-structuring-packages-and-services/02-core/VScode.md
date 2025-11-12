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
  * `${workspaceFolder}`: `08-structuring-packages-and-services\02-core`
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
## 8.2 Core
> [!TIP] ADDITIONAL SERVICES 
> 
> Although we provide a `link.Shortener` service, extra services we could provide later could also fit into our package structure. Imagine a `link.Analytics` service that provides link statistics. Its core logic could go into the `link` package, the REST API into `rest`, and persistence into `sqlite`. Rather than creating new packages, using the existing structure can help us avoid import cycles and provide a nice structure for expanding the project. See https://github.com/inancgumus/gobyexample/blob/source/link/analytics.go . 
### 8.2.1 Errors
- [Listing 8.1: Standardized errors](../../all-listings/08-structuring-packages-and-services/01-standardized-errors.md)
> [!TIP] DEEP DIVE: SATISFYING THE ERROR INTERFACE
> Using error variables (e.g., `ErrNotFound`) is enough for our project because we would map them only to HTTP status codes. If we need extra context, it might be helpful to declare custom error types that satisfy the `error` interface, adding extra behavior and data. The following `NotFoundError` satisfies `error`, letting us store an `ID`. Naming it with an `Error` suffix is idiomatic, clarifying that it implements the `error` interface:
> ```golang
> type NotFoundError struct{ ID string }
> func (e *NotFoundError) Error() string {  #1
>     return fmt.Sprintf("%q not found", e.ID)
> }
> ```
> #1 Satisfies the error interface          
> 
> Because the `*NotFoundError` type satisfies the `error` interface with its `Error` method, we can return a `*NotFoundError` from a function as an `error` value:
> ```golang
> func Resolve() error { return &NotFoundError{ID: "42"} }
> ```
> Then we can match this error using errors.As 
> ```golang
> var nfe *NotFoundError
> if err := Resolve(); errors.As(err, &nfe) {  #1
>     fmt.Printf("unknown item: %s\n", nfe.ID)
> }
> ```
> #1  Extracts the custom error from the error value      
>
> Custom errors are nice, but we must be mindful of a subtle trap. The following code, for example, returns a `nil *NotFoundError` instead of returning `nil` directly for success:
> ```golang
> func ResolveBad() error {
>     var err *NotFoundError
>     return err  #1
> }
> ```
> #1 Returning a nil *NotFoundError for success      
>
> We should have returned `nil` for success directly, not as a `nil` custom error. Otherwise, the returned `error` would never be `nil`. The following use panics, for example:
> ```golang
> if err := ResolveBad(); errors.As(err, &nfe) { #1
>      fmt.Printf("unknown item: %s\n", nfe.ID) #2
> }
> ```
> #1 Detects the error as a non-nil error and runs Printf with a nil *NotFoundError      
> #2 Panic: invalid memory address or nil-pointer dereference  
> 
> An interface value, such as an `error` value, has a type and a value part. The `nfe` variable stores a non-`nil *NotFoundError` type and a `nil` value. Because its type part is not `nil`, `nfe` is not `nil`, even though its value part is a `nil *NotFoundError` value.
> 
> You can try this example at https://go.dev/play/p/Fsb3iiBK1lX to better understand this misuse. You can also read appendix D to learn more about interface values.

### 8.2.2 Core
#### CORE TYPES
- [Listing 8.2: Core types](../../all-listings/08-structuring-packages-and-services/02-core-types.md)
 
> [!NOTE] 
> Learn more about untyped constants at https://go.dev/blog/constants, and see appendix D to learn more about types and underlying types.

#### VALIDATION
- [Listing 8.3: Validation](../../all-listings/08-structuring-packages-and-services/03-validation.md)
> [!TIP] 
> Keep method signatures aligned to improve reusability.

#### URL SHORTENER
- [Listing 8.4: `Shorten`](../../all-listings/08-structuring-packages-and-services/04-shorten.md)

### 8.2.3 Service
#### MAKING A TYPE’S ZERO VALUE USEFUL
- [Listing 8.5: `Shortener` service](../../all-listings/08-structuring-packages-and-services/05-shortener-service.md)
> [!TIP] 
> Design the types so that their zero value is useful. If forcing a zero value to be useful adds complexity, however, it might be better to use a constructor.

> [!TIP] DIFFERENT WAYS OF INITIALIZING A MAP
> There are two ways to initialize a map:
> * With a map literal—`map[Key]Link{}`
> * With the built-in make function—`make(map[Key]Link)`
> 
> Both approaches are the same. But `make` is helpful if we also want to preallocate a map:
> ```golang
> links := make(map[Key]Link, 1_000)  #1
> ```
> #1 Preallocates and returns a map with 1,000 empty entries
>
> You can learn more about the basic use of map types in appendix C.
#### READING FROM A NIL MAP IS SAFE
- [Listing 8.6: `Shortener.Resolve`](../../all-listings/08-structuring-packages-and-services/06-shortenerresolve.md)

> [!NOTE] 
> Reading from a map doesn’t require initialization and returns its element type’s zero value. `map[string]int`’s element type, for example, is `int`. Reading from it returns `0` if the map is `nil` because the `int` type’s zero value is `0`. Also, reading from a `nil map[int]bool` returns `false` because the `bool` type’s zero value is `false`.

#### 8.2.4 Mutex
> [!TIP] 
> Writing efficient code is idiomatic as long as it doesn’t hurt clarity.

##### PROTECTING ACCESS TO A RESOURCE WITH A MUTEX
A mutex has two main operations:
* `Locking`—Grants exclusive access to a goroutine, blocking others waiting to lock
* `Unlocking`—Allows one of the goroutines to lock the mutex
  
> [!NOTE] 
>  A mutex is initially in an unlocked state.

A goroutine that locks a mutex blocks others by saying, in essence, “Wait until I finish my work.” When it unlocks the mutex, one of the other goroutines can lock it and tell others the same thing. Because locked mutexes block other goroutines, however, read and write throughput may be reduced. To improve efficiency for specific scenarios, Go provides `Mutex` and `RWMutex` types:
* `Mutex` provides exclusive access to a single goroutine at a time.
* `RWMutex` allows multiple goroutines to read, but only one goroutine at a time can write.
  
Put simply, `RWMutex` allows multiple goroutines to read a shared state concurrently. It blocks others only when a write operation is in progress. Because we expect more reads than writes in our project, we’ll go with the `RWMutex` type for efficiency reasons.

> [!NOTE] DEEP DIVE: MUTEX VS. RWMUTEX
>  `Mutex` might be more useful if reads and writes are balanced or if writes are more frequent. If reads are more frequent, however, `RWMutex` can improve throughput. Its benefits diminish if the write operations are frequent or long-running because readers can starve. Although `RWMutex` might improve performance in read-heavy cases, it’s worth benchmarking both mutex types to see which one provides the best efficiency for a specific case.
##### MUTEXES IN PRACTICE
- [Listing 8.7: `RWMutex` protection](../../all-listings/08-structuring-packages-and-services/07-rwmutex-protection.md)

We protect the map from unsafe concurrent use:
* Only one goroutine can write into the map at a time.
* Multiple goroutines can read the map unless there’s a writer.

We use the following `RWMutex` methods for locking and unlocking:
* `Lock` locks `RWMutex` for exclusive write access, and `RLock` locks it for read access.
* For unlocking, we call `Unlock` after `Lock` while we call `RUnlock` after `RLock`.

If we forget to unlock the mutex, goroutines waiting to lock it will wait forever. Eventually, our program will become unresponsive and use so much memory that it may crash. To eliminate these issues, we use a `defer` to reduce our chances of forgetting to unlock the mutex.

> [!NOTE]
> It’s idiomatic to add a comment if a type or a function is safe for concurrent use, such as “It’s safe for concurrent use from multiple goroutines.”