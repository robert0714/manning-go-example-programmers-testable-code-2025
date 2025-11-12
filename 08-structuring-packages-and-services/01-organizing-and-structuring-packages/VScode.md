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
  * `${workspaceFolder}`: `08-structuring-packages-and-services\01-organizing-and-structuring-packages`
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
## 8.1 Organizing and structuring packages
### 8.1.1 Avoiding import cycles
### 8.1.2 Structuring packages in practice
Figure 8.4 shows a layered package structure for our link-services project in which each package imports only from below, never from above. The `link/rest` package can depend on the `link` package, for example, but `link` must not depend on `link/rest`. As we go up the layers, packages become more specialized (except `linkd`, which is not importable). Going down leads to more foundational ones.   
![architecture diagram](pics/CH08_F04_Gumus.drawio.svg)   
Figure 8.4 A layered approach to organizing packages

Each package is an expert in its area and provides something unique:
```bash
link  #1
├─ rest  #2
├─ sqlite  #3
├─ kit  #4
│  ├─ hio  #5
│  ├─ hlog  #6
│  └─ traceid  #7
└─ cmd  #8
   └─ linkd  #9
```
#1 Provides core logic for link management services   
#2 Provides a link management service over a REST API   
#3 Provides an SQLite-backed link persistence service   
#4 A directory to group infrastructural helper packages   
#5 Provides HTTP request and response-handling helpers   
#6 Provides HTTP request and response-logging helpers   
#7 Provides trace ID generation and propagation   
#8 A directory to group executable commands   
#9 package main that wires all packages together and runs the link REST API server over HTTP   

This structure prevents import cycles by maintaining explicit dependencies. The `link` package provides core link management services without depending on our other packages. The `link/rest` package builds on top of the link package and provides a REST API over HTTP. Finally, the `linkd` command (`main` package) wires all packages to run an HTTP server that serves our REST API.

> [!TIP] DEEP DIVE: THE INTERNAL DIRECTORY CONVENTION
> Go has a special directory called `internal` that restricts package imports. Suppose that our directory structure looked like this:
> ```bash
> svc
> ├─ link  #1
> │  ├─ internal  #2
> │  │  ├─ rest  #3
> │  │  └─ sqlite  #3
> │  └─ cmd
> │    └─ linkd  #3
> └─ stats  #4
> ```
> #1 The parent of svc/link/internal     
> #2 Restricts the import of the packages underneath      
> #3 Can import packages under svc/link/internal      
> #4 Cannot import packages under svc/link/internal        
> 
> The parent directory of `internal` is `link`. Any package below link or its subfolders can import packages in `internal` because they share a root directory: `link. cmd/linkd` can import `internal/rest`, for example, but `stats` cannot because it sits outside `link`. In this case, `stats` is an external package and doesn’t have the same root as our `internal`.
> 
> An `internal` directory enforces encapsulation, signaling that certain packages aren’t part of a module’s API. This reduces the risk of breaking external dependencies with internal changes and allows packages inside to evolve without introducing breaking changes. In our scenario, extra boundaries add little value because our module is for our use. But if we decide to share it, we’ll likely use `internal` so that later changes to our code won’t break its dependent code. Check out https://go.dev/doc/modules/layout for more information.