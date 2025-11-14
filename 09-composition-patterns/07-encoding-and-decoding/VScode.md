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
  * `${workspaceFolder}`: `09-composition-patterns\07-encoding-and-decoding`
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
## 9.7 Encoding and decoding
We built chainable HTTP handlers and response helpers earlier, but our handlers responded only with plain text. Real-world services often exchange structured data formats such as JSON, making parsing and validation straightforward, which is especially important for clients and servers.

In this section, we’ll learn practical and reliable techniques to decode client requests and encode responses in JSON. We’ll use the standard library’s `encoding/json` package and introduce reusable helpers to simplify serialization tasks, error handling, and input validation. We’ll also dive deeper into anonymous interfaces and type assertions to extract and use behaviors like validation, keeping our handlers concise and composable.

> [!NOTE] 
> Visit https://go.dev/blog/json for more details about `json`. Go will soon have a more performant `json/v2` package. Visit https://pkg.go.dev/encoding/json/v2 for details (which is fully compatible with the current `json` package).

### 9.7.1 Encoding JSON
The standard library’s `encoding/json` package allows us to do the following:
* Encode a value in JSON using the `json.Marshal` function
* Decode JSON in a value using the `json.Unmarshal` function

In this section, we’ll use these functions to implement a `JSON` handler to write JSON responses to clients. In section 9.7.2, we’ll use a `DecodeJSON` helper to the JSON payloads from clients.

As listing 9.22 shows, we add a `JSON` response helper similar to the earlier ones, like `Text`. This helper simplifies responding with JSON payloads. We’ll implement it like other response helpers, terminating the chain at the end and handling encoding errors with `Error`.
- [Listing 9.22: JSON response helper](../../all-listings/09-composition-patterns/22-json-response-helper.md)

We’ve added a `JSON` helper method to our `Responder` type, which encodes the provided value into JSON in the `[]byte` variable, data. If encoding fails, it delegates response handling to our `Error` handler; otherwise, it returns a closure that writes the JSON payload to the client.

As with every response helper, such as `Text`, returning `nil` afterward terminates the handler chain and prevents handling the rest of the response, eliminating unintentional mistakes. We call `Marshal` directly outside the returned `Handler` closure because we want to fail fast on errors, and that block of code does not require using a `ResponseWriter` or a `*Request`.
### 9.7.2 Decoding JSON
Besides sending JSON payloads to clients, handlers receive JSON payloads. As listing 9.23 shows, we add a helper function to decode JSON payloads from incoming requests bodies (or any `Reader`). We also validate the decoded data automatically if the underlying type of the `to` variable has a `Validate` method; in section 9.7.3, we’ll use it to verify inputs in handlers.
- [Listing 9.23: Decoding JSON](../../all-listings/09-composition-patterns/23-decoding-json.md)

We’ve declared `DecodeJSON` to decode JSON requests. This helper does the following:
* Reads the entire JSON payload into memory and decodes it into the provided variable.
* Checks the underlying type of `to` using an anonymous interface. If the underlying type has a `Validate` method, we call `Validate` to validate the decoded data.
* 
With `DecodeJSON`, we avoid repeating decoding and validation in our handlers. But `DecodeJSON` can overwhelm our server as it reads the entire JSON payload into memory without limit. Before fixing this problem, let’s integrate our new helpers into `Shorten`.
> [!TIP] 
> Using anonymous interfaces for type assertions minimizes dependencies on external types or packages, improves maintainability, and reduces coupling.

### 9.7.3 Speaking JSON
We added helpers to respond to JSON and decode incoming JSON data. These helpers simplify handling JSON and prevent repetitive encoding and decoding logic in handlers. Next, we’ll integrate these helpers into our existing `Shorten` handler. As the following listing shows, we update `Shorten` to decode the incoming JSON payload from `Request.Body` using `DecodeJSON` and respond using our new `JSON` response helper.

- [Listing 9.24: Speaking JSON](../../all-listings/09-composition-patterns/24-speaking-json.md)

We’ve updated `Shorten` to decode and respond with JSON; now it uses our new helpers (`DecodeJSON` and `JSON`) to simplify handling structured JSON data. Suppose that the `Shorten` handler receives the following JSON payload:
```json
{"url": "https://go.dev", "key": "go"}
```
Because we export the Link’s fields, Shorten can decode incoming data into a Link:
```golang
type Link struct {
    URL string  #1
    Key Key  #2
}
```
#1 This field is exported and will be mapped to the JSON data’s url field.   
#2 This field is exported and will be mapped to the JSON data’s key field.    

Otherwise, the `json.Unmarshal` function wouldn’t decode JSON into a `Link`. In this case, `Link`’s `URL` field will hold "`https://go.dev`", and the `Key` field will store "`go`". As an example, let’s post a JSON payload and receive a JSON response:
```bash
$ curl -i localhost:8080/shorten  #1
➥     -d '{"url": "https://go.dev", "key": "go"}'  #1
HTTP/1.1 201 Created
{"key":"go"}  #2
```
#1 Sends JSON data to the handler    
#2 Receives JSON data from the handler   

Let’s also check whether DecodeJSON calls the Link’s Validate method automatically:
```bash
$ curl localhost:8080/shorten -i -d '{"url":"not a URL"}'
HTTP/1.1 400 Bad Request
decoding: validating: parse "not a URL": #1
➥        invalid URI for request: bad request
```
#1 DecodeJSON detects and calls the Validate method because we pass a Link variable.   

We explored how to implement small helpers to encode and decode JSON, integrating them into our `Shorten` handler to make it speak JSON. But as noted earlier, `DecodeJSON` uses `io.ReadAll` to read the `Request.Body` without limit. Fortunately, it takes an `io.Reader`, which means we can wrap the `Reader` with another one that limits how many bytes it can read to improve server reliability and security. We’ll explore this approach in section 9.8.

> [!TIP] 
> `json` considers only exported fields while encoding and decoding.
