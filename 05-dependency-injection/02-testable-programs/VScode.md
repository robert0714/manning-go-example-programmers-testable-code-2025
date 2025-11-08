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
  * `${workspaceFolder}`: `05-dependency-injection\02-testable-programs`
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
# 5 Dependency injection 
## 5.2 Testable programs
### 5.2.1 Overview 
![architecture diagram](pics/CH05_F02_Gumus.drawio.svg)   
Figure 5.2 Moving `main's` code into a run function that accepts an `*env` struct. The `env` struct has fields for us to inject the global variables, such as the `os.Stdout`. This approach allows `main` to inject real globals to `run`, and tests can inject controlled fakes that we can observe from tests later.
### 5.2.2 A place to store dependencies
- [Listing 5.1: The `env` type](../../all-listings/05-dependency-injection/01-the-env-type.md)   

### 5.2.3 io.Writer’s role
Because we use the Writer interface in the env fields, let’s understand its role. We can satisfy the Writer interface by implementing its Write method. This method writes a byte slice to a destination stream and then returns the number of bytes written and an error:
```golang
type Writer interface {
    Write(p []byte) (n int, err error)
}
```
`Writer` standardizes and abstracts writing to any output stream, such as standard output, a file, or network connection. (Think of it as being like `OutputStream` in Java, `Stream.Write` in C#, or `RawIOBase.write` in Python.) `Writer` is one of the most ubiquitous interfaces in Go’s standard library, and countless implementations of the interface are available in the wild. We’ll use these `Writer` implementations for running and testing the HIT tool:

* `os.File`, which represents an operating system file handle
* `strings.Builder`, which provides an in-memory byte buffer

To make our tool work the same way as before, while users run it, we will assign `os.Stdout` and `os.Stderr` to the env struct’s stdout and stderr fields; they are an `*os.File`, which is a `Writer` implementation. While testing the tool, we will use the `strings.Builder` type, which is another `Writer`, to record the tool’s output.