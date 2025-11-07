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
  * `${workspaceFolder}`: `04-command-line-interfaces\03-the-flag-package`
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
## 4.3 The flag package
![architecture diagram](./pics/CH04_F05_Gumus.drawio.svg)   
Figure 4.5 `FlagSet` maps flag names to `Flag`s. Its Parse method `parses` the flags from command-line arguments, and the value parsers update the variables.


- [Listing 4.6: Parsing with `FlagSet`](../../all-listings/04-command-line-interfaces/06-parsing-with-flagset.md)
- [Listing 4.7: Removing usage text](../../all-listings/04-command-line-interfaces/07-removing-usage-text.md)