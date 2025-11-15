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
  * `${workspaceFolder}`: `10-polymorphic-storage\04-implicit-interfaces`
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

 
# 10 Polymorphic storage
## 10.4 Implicit interfaces
Don’t design with interfaces; discover them. —Rob Pike

Now that the SQLite-backed `Shortener` service is ready, our team will help integrate the other team’s new `Shortener` into our REST API, as promised. We’ll explore how Go’s implicit interfaces can ease this integration with a few code changes.

Interfaces are extremely useful, but misusing them adds unnecessary complexity. Declaring an interface unnecessarily can complicate code and reduce clarity because interfaces add indirection and require additional maintenance. By focusing on concrete types first, we can recognize genuine needs and create meaningful interfaces.

So far, I’ve deliberately avoided introducing interfaces, but it’s time now because we have two different concrete types: `link.Shortener` and `sqlite.Shortener`. Let’s explore how and where to declare interfaces and the benefits of following idiomatic practices.

> [!TIP] 
> Every interface should earn its place in the codebase.
### 10.4.1 Consumers-first approach
In Go, it’s idiomatic to declare interfaces in the package that consumes them, not in the package that implements them. Instead of providing an interface in the `sqlite` package, consumers like the `rest` package can declare an interface with the methods they need.

Figure 10.6 illustrates this consumers-first approach, in which consumers define the interfaces they need rather than providers. The `rest` package (consumer) owns the `Shortener` interface, allowing us to pass any type with a Shorten method to the `rest.Shorten` function, such as our `sqlite` package’s (provider) `Shortener` type.

The `rest` package specifies what methods a `Shortener` should have (i.e., `Shorten`). We can use any type that fulfills those methods, such as `sqlite.Shortener`, with the `rest` package. The `sqlite` package doesn’t have to explicitly declare that it implements the `rest.Shortener` interface, which is possible due to Go’s implicit interfaces.

![architecture diagram](pics/CH10_F06_Gumus.drawio.svg)  
Figure 10.6 The consumer (`rest`) owns the interface rather than the provider (`sqlite`).

By focusing on consumer needs, we avoid bloated interfaces that are difficult to satisfy. Consumer-driven interfaces naturally remain smaller and laser-focused because they include only methods that consumers genuinely require today, rather than methods we think they might need later. Small interfaces clearly communicate their purpose, making them easier to understand and simpler to satisfy because implementations must fulfill fewer methods. Such minimal interfaces are easily embedded in larger ones, improving composability.

Also, by declaring only essential methods, these small interfaces reduce complexity and encourage loose coupling. If a provider later adds more methods to its concrete type, the consumer code remains unaffected because it depends only on methods declared in its local interface (e.g., `rest.Shortener`). Small interfaces help to reduce cascading breaking changes throughout our codebase, significantly improving maintainability.

The consumer-driven interface approach aligns well with Go’s core principles, emphasizing simplicity. This approach promotes decoupling, better testability, and ease of maintenance.

> [!IMPORTANT]
> DEEP DIVE: CENTRALIZING WIDELY USED BEHAVIORS   
> Declaring interfaces on the consumer side helps prevent unnecessary speculation. Consumers define only the behaviors they need, keeping interfaces minimal and relevant. Sometimes, however, multiple consumers need the same behavior. `json`, `xml`, and similar packages have functions to convert arbitrary types to text, for example. Rather than having each consumer package declare its own interface, we may find it more effective to centralize such common behaviors in a single package after they’ve emerged. The encoding package, for example, has a widely used TextMarshaler interface:
> ```golang
> package encoding
> type TextMarshaler interface {
>     MarshalText() (text []byte, err error)
> }
> ```
> Types like net.IP and slog.Level implement this interface without knowing how their encoded form will be used. They simply support encoding themselves as text:
> ```golang
> package net
> type IP []byte
> . . .
> func (ip IP) MarshalText() ([]byte, error) { . . . }
> ```
> Functions like json.Marshal recognize this interface. When we encode an IP value using Marshal, it delegates text encoding to the IP’s MarshalText method:
> ```golang
> type Server struct { Addr net.IP `json:"addr"` }
> data, err := json.Marshal(Server{ #1
>     Addr: net.IPv4(192, 168, 1, 1), #1
> })
> . . .
> fmt.Println(string(data))
> ```
> #1 Encodes the IP value to JSON by calling IP.MarshalText()
> 
> The result of this program is
> ```json
> {"addr":"192.168.1.1"}
> ```
> Centralizing widely used interfaces prevents repetition and ensures consistency. But we shouldn’t rush to centralize interfaces (e.g., put them in a single package) before discovering common behaviors through concrete uses. When we have multiple uses, we may realize that they share behavior and can see how to centralize it. Discover behaviors first through concrete implementations; then centralize these behaviors with shared interfaces. This practice prevents premature abstractions and unnecessary complexity.
> 
### 10.4.2 Providing an interface
Let’s put the consumer-driven interfaces approach into practice. We’ll provide two interfaces in the `rest` package for its current requirements: a `Shortener` interface with a Shorten method to shorten links and a `Resolver` interface with a Resolve method to resolve links.

> [!TIP] 
> Adding an `-er` prefix to a single-method interface is a convention. Not all interfaces require this prefix, especially if they don’t embed other interfaces.

Our HTTP handlers need these methods only to `shorten` and resolve links. The `Shorten` handler, for example, requires only a `Shorten` method, so it expects the `Shortener` interface rather than a bloated one with many methods. As the following listing shows, we maintain a clear separation of concerns and avoid bloating the interfaces with unnecessary methods.
- [Listing 10.13: Declaring interfaces](../../all-listings/10-polymorphic-storage/13-declaring-interfaces.md)

With these one-method interfaces in place, the `Shorten` handler can accept any type that implements the `Shorten` method (e.g., `sqlite.Shortener`) without modifying the rest of the handler code, or the `Resolve` handler can accept any type with a `Resolve` method. We replaced the HTTP handler input parameters with interfaces, but the rest of our code remained the same and still compiles. That’s one reason why we’ve avoided declaring interfaces so far. In Go, it’s straightforward to declare new interfaces when necessary rather than up front.

We decoupled the rest package from concrete Shortener implementations by declaring these interfaces. Both `link.Shortener` and `sqlite.Shortener` satisfy our interfaces, allowing us to pass either to our handlers. (Currently, we pass `link.Shortener`.) Now other teams can improve the link service’s storage backend with any new features they want, such as adding Redis as a cache or using an Amazon Web Services database to store links in the cloud. We don’t need to make any other changes to our handlers to support these features.

### 10.4.3 Activating the new implementation
Now that we have declared interfaces, we can activate the `sqlite.Shortener` service in our program. Recall that our `linkd` program sets up and runs the link HTTP server. Now we need only add a `dsn` flag and connect to SQLite, as follows. No other changes are necessary.

We connect to SQLite using `Dial`. We store the links in the `links.db` file. The `rwc` mode opens the file for reading and writing and creates the file if it doesn’t exist.

- [Listing 10.14: Integrating `sqlite.Shortener`](../../all-listings/10-polymorphic-storage/14-integrating-sqliteshortener.md)

We’ve updated the link server to use SQLite with minimal changes. The handlers will shorten and resolve links using the new `sqlite.Shortener`. Let’s try this new change by running the server:
```bash
$ go run ./link/cmd/linkd
level=INFO msg=starting app=linkd addr=localhost:8080
```
In another terminal session, type
```bash
$ curl -i localhost:8080/shorten -d '{"url": "https://x.com/inancgumus"}'
HTTP/1.1 201 Created
{"key":"gXyMWIga"}
```
Now we can restart the server. Because we use a persistent database, our program will preserve the links. We can resolve the preceding link after a restart:
```bash
$ curl -i localhost:8080/r/gXyMWIga
HTTP/1.1 302 Found
Location: https://x.com/inancgumus
```
In this book, we’ve explored many practical Go patterns and idioms through realistic examples, consistently choosing the path toward Mount Simplicity—the realm of idiomatic Go programming. (See the figure on the inside front cover of this book.) Although complexity isn’t always preventable or inherently harmful, maintainable code emerges when we handle complexity carefully behind simple and straightforward code. By recognizing pitfalls on Mount Complexity, we’ve aimed to write effective, testable code. Thank you for reading this far. I hope you found the journey enlightening. One final piece of advice: always keep it simple.