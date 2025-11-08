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
  * `${workspaceFolder}`: `04-command-line-interfaces\04-value-parsers`
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
# 4 Command-line interfaces 
## 4.4 Value parsers
```bash
$ go run ./hit/cmd/hit -n -1
Sending -1 requests to "" (concurrency: 1)
$ go run ./hit/cmd/hit -c 0
Sending 100 requests to "" (concurrency: 0)

```
### 4.4.1 Value interface
![architecture diagram](./pics/CH04_F06_Gumus.drawio.svg)   
Figure 4.6 A `Flag` has a `Value` field with the type of the `Value` interface. Value parsers, such as `stringValue` and `intValue`, implement the `Value` interface.

The `flag` package declares the `Flag` type as follows:
```golang
type Flag struct {
    Name     string  #1
    Usage    string  #2
    DefValue string  #3
    Value    flag.Value  #4
}
```
#1 Flag’s name   
#2 Flag’s usage message   
#3 Flag’s default value   
#4 Flag’s flag value parser   

The `flag` package declares the Value interface as follows:
```golang
type Value interface {
    String() string  #1
    Set(string) error  #2
}
```
#1 Queries the associated variable and returns its value as a string  
#2 Parses a flag’s value and sets it to the associated variable   

A flag value parser must implement the following methods of the `Value` interface:
* `String` returns the variable’s value associated with a `Flag`. A `FlagSet` calls this method once to set the Flag’s default value to show it in the usage text later.
* `Set` parses a flag’s value and sets it to the variable.

![architecture diagram](./pics/CH04_F07_Gumus.drawio.svg)   
Figure 4.7 The `intValue` is a Value implementation that can handle `int` variables. The variable `c` is an integer variable linked to an `intValue` parser.

In summary, `Flag` values have a `Value` field to handle any variable type via a value parser that satisfies the `Value` interface. The `Value` interface has many built-in implementations, including `stringValue`, `intValue`, and `boolValue`. We’ll add our custom implementation to validate positive numbers.
> [!TIP]
> Idiomatic interfaces like `Value` are lean and focus on shared behavior. In a `FlagSet`, `Flag` instances are concrete types, not interfaces. The `Value` field within a `Flag` is the only interface that reflects its polymorphic behavior. The remaining fields, such as `Name`, remain concrete because they don’t require the flexibility of an interface.
>
### 4.4.2 Satisfying the Value interface
### 4.4.3 Using Var
- [Listing 4.8: Satisfying `flag.Value`](../../all-listings/04-command-line-interfaces/08-satisfying-flagvalue.md)
- [Listing 4.9: Using `FlagSet.Var`](../../all-listings/04-command-line-interfaces/09-using-flagsetvar.md)