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
  * `${workspaceFolder}`: `04-command-line-interfaces\02-flag-parsing`
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
## 4.2 Flag parsing
### 4.2.1 Overview
![architecture diagram](./pics/CH04_F02_Gumus.drawio.svg)   
Figure 4.2 The `flag` package parses and saves the command-line arguments set in `os.Args` by the Go runtime into type-safe regular Go variables that we provide.
> [!NOTE]
> `n` in "`-n=100`" is the flag’s name, and `100` is the flag’s value.

> [!NOTE]
> Command-line arguments are string values. The `flag` package parses each argument into the specified type-safe variable that we can use in our programs.

### 4.2.2 Higher-order value parsers
![architecture diagram](./pics/CH04_F04_Gumus.drawio.svg)   
Figure 4.4 Calling `intVar` returns a closure that retains the variable’s pointer. Calling the closure updates the original variable’s value through the pointer.

Now that we understand how higher-order functions provide value parsers as closures, it’s time to implement these functions in code. As listing 4.2 shows, we declare two higher-order functions: `stringVar` and `intVar`. Each function accepts a pointer to the variable it will update and returns a value parser closure that parses and assigns flag values to that variable. We also introduce a function type called `parseFunc` that standardizes the signature for value parsers.

[See code](./hit/cmd/hit/config.go#L58C1-L59C1)
- [Listing 4.2: Flag value parsers](../../all-listings/04-command-line-interfaces/02-flag-value-parsers.md)

> [!NOTE]
>  `Atoi` is short for ASCII to integer. It’s a conventional name used since 1971.

### 4.2.3 Implementing a flag parser
> [!TIP]
> `Passing a pointer means sharing`. Avoid passing a pointer if you don’t want to share the original data with another function. Here, we share the `config` value with a pointer to allow the `parseArgs` function to update the original `config` value’s fields.
- [Listing 4.3: Flag parser](../../all-listings/04-command-line-interfaces/03-flag-parser.md)
- [Listing 4.3 Flag parser (hit/cmd/hit/config.go)](./hit/cmd/hit/config.go#L18C1-L46C1)
  ```golang
  package main

  import (
      "fmt"
      "strings"
      . . .
  )

  type config struct {
      url string
      n   int
      c   int
      rps int
  }

  func parseArgs(c *config, args []string) error {
      flagSet := map[string]parseFunc{  #1
          "url": stringVar(&c.url),  #2
          "n":   intVar(&c.n),  #3
          "c":   intVar(&c.c),   #3
          "rps": intVar(&c.rps),   #3
      }

      for _, arg := range args {
          name, val, _ := strings.Cut(arg, "=")  #4
          name = strings.TrimPrefix(name, "-")  #5

          setVar, ok := flagSet[name]  #6
          if !ok {
            return fmt.Errorf(
                  "flag provided but not defined: -%s",
                  name,
              )
          }
          if err := setVar(val); err != nil { #7
              return fmt.Errorf(  #8
                  "invalid value %q for flag -%s: %w",  #8
                  val, name, err,   #8
              )   #8
          }
      }

      return nil
  }
  ```
  #1 Declares a map of flag names and value parsers  
  #2 Binds the url flag to a string value parser that will update the `config.url` field  
  #3 Binds the flags to integer value parsers that will update the corresponding config fields  
  #4 Parses an argument into a flag name and value   
  #5 Removes the leading dash from the flag’s name (e.g., `-n` becomes `n`)   
  #6 Fetches the value parser from the map by flag name   
  #7 `setVar` parses the flag’s value and sets the converted value to a config field.   
  #8 Passes the `%w` verb to Errorf to get a new error value that adds context to the error   

* The flagSet map binds the flag names to value parsers:
  * "url"—stringVar(&c.url)
  * "n"—intVar(&c.n)
  * 
* Each function takes a pointer to a field of the `config` struct and returns a value parser as a closure. This closure retains the pointer, parses the flag’s value, and assigns that value to the provided field. After our flag parser parses the `url` flag, for example, it passes the flag’s value to the closure returned from `stringVar(&c.url)`. Then the closure updates the `config.url` field.

> [!TIP]
> Naming the struct fields to match the flag names makes it easier to see their relationship. In this case, we use shorter field names for convenience.

Also, passing the `%w` verb to `Errorf` returns an error that wraps the original:
```golang
if err := setVar(val); err != nil {
    return fmt.Errorf(
        ". . .: %w",  #1
        . . ., err,  #2
    )
}
```
#1 %w wraps the error.  
#2 err is the original error.  

> [!TIP]
> The statement `if err := ...; err != nil` limits the scope of the `err` variable to the `if` statement, preventing potential accidental reuse elsewhere.

`intVar` calls `strconv.Atoi` to parse a flag value. `Atoi` may return a `strconv.ErrSyntax` error if parsing fails. When that happens, our flag parser returns a new error that wraps the `ErrSyntax` error. Then we can detect the wrapped error as follows:
```golang
errors.Is(err, strconv.ErrSyntax)
```
The `errors.Is` function returns `true` if this `err` wraps the `strconv.ErrSyntax` error. This approach enables us to respond precisely by giving users more informative error messages or handling specific errors differently in our program logic. In chapter 8, we’ll deep-dive into handling errors using `errors.Is` and see its use in practice.

> [!TIP]
> `errors.Is` detects an underlying error wrapped in an error chain.

Before wrapping up this section, let’s discuss some conventions we use in our code. We separate the flags from their parsers to improve maintainability. We can add flags simply by adding new entries in the `flagSet` map. Also, the code is easy to read from top to bottom and maintains left-aligned formatting instead of using deeply nested conditional statements. We can easily see error conditions at a glance, which improves readability and clarity. We favor left-aligned code that returns early, like

```golang
setVar, ok := flagSet[name]
if !ok {
    return . . . 
}
if err := setVar(val); err != nil {
    return . . .
}
```

rather than

```golang
setVar, ok := flagSet[name]
if ok {
    if err := setVar(val); err != nil {
        return . . .
    }
} else {
    return . . .
}
```

The first version calls setVar outside the conditional that checks a flag in the map.

> [!TIP]
> Idiomatic code is left-aligned, which makes exit conditions clear. Avoid deeply right-aligning the code, which makes the code harder to grasp.

### 4.2.4 Integration and setting sensible defaults
- [Listing 4.4: Flag parser integration](../../all-listings/04-command-line-interfaces/04-flag-parser-integration.md)
- [Listing 4.5: Setting sensible defaults](../../all-listings/04-command-line-interfaces/05-setting-sensible-defaults.md)
