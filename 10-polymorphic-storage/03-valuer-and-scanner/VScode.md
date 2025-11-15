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
  * `${workspaceFolder}`: `10-polymorphic-storage\03-valuer-and-scanner`
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
## 10.3 Valuer and Scanner
We used `ExecContext` to insert links and `QueryRowContext` to retrieve them from SQLite. Because `sql` abstracts database communication, we can use PostgreSQL or another SQL database without many changes, but this abstraction has limits. Sometimes, we need extra features specific to an SQL database that the sql package doesn’t directly support. To bridge this gap, the `sql` and `driver` packages have the following `Valuer` and `Scanner` interfaces.

A `driver.Valuer` implementation can transform values before sending them to a database:
```golang
type Valuer interface {
    Value() (driver.Value, error) #1
}
```
#1 Returns a database-compatible value   

An `sql.Scanner` implementation can transform values retrieved from a database:
```golang
type Scanner interface {
    Scan(src any) error #1
}
```
#1 Converts database values to Go types  

These interfaces help us use special database types or custom behaviors.

### 10.3.1 Supporting custom database types
Suppose that we use PostgreSQL, which supports array types and stores their elements packed as binary data in a single database column. The `sql` package doesn’t natively support PostgreSQL arrays, but the `pq` driver provides the `pq.Array` type, which implements `Valuer` and `Scanner`. This type converts slices to PostgreSQL arrays and vice versa:
```golang
var scores []int #1
db.QueryRowContext(. . .,
   `SELECT ARRAY[42, 84]`,
).Scan(pq.Array(&scores)) #2
```
#1 Will contain [42, 84]   
#2 Retrieves, converts, and injects PostgreSQL’s array data into scores   

We retrieve a PostgreSQL array into the scores slice:
* `QueryRowContext` retrieves the PostgreSQL array.
* `pq.Array(&scores)` decodes the array into the `scores` slice.


We can also insert this slice as a PostgreSQL array using `ExecContext`:
```golang
_, err := db.ExecContext(. . .,
  `INSERT INTO results(scores) VALUES($1)`,
   pq.Array(scores))
```

In this example
* `pq.Array(scores)` converts `scores` to a PostgreSQL array.
* `ExecContext` inserts that array into the `results` table.
  
These examples show how `Valuer` and `Scanner` enable database-specific features without forgoing the abstraction provided by the `sql` package. We’ll apply these interfaces ourselves in section 10.3.2 to better understand how they work.

> [!NOTE] 
> See https://pkg.go.dev/github.com/lib/pq#Array for more information.

### 10.3.2 Satisfying Valuer and Scanner
Suppose that we want to automatically encode URLs in Base64 before saving and decode them after retrieving. To do that, we can declare a new type that satisfies the `Valuer` and `Scanner`. As listing 10.11 shows, we declare the `base64String` type, which uses the standard library’s `base64.StdEncoding` type for encoding and decoding. We use `EncodeToString` to encode a byte slice into a string and `DecodeString` to decode from a string to a byte slice.
- [Listing 10.11: `Valuer` and `Scanner`](../../all-listings/10-polymorphic-storage/11-valuer-and-scanner.md)

We’ve implemented a type that satisfies the `Scanner` and `Valuer` interfaces:
* The `Scan` method implements the `Scanner` interface.
* The `Value` method implements the `Valuer` interface.

We declared Value using a `value` receiver and `Scan` using a pointer receiver. Although it’s best to avoid mixing receiver types, it’s necessary sometimes. Using a pointer receiver for `Scan` allows us to inject the decoded string into the original `base64String` via the receiver, bs. Finally, because a database column can be of different types, `Scan` takes `any` type. Using a runtime type assertion (src.(string)), we make sure that what we receive is a string.

> [!NOTE]
> See appendix D for reasons to avoid mixing receiver types.

### 10.3.3 Encoding and decoding
Now that our type satisfies `Valuer` and `Scanner`, let’s use it in `Shortener`. In listing 10.12, we integrate `base64String` into `Shorten` to encode a URL before sending it to the database and `Resolve` to decode the encoded URL after retrieving it from the database. We use a `base64String` instead of a plain string to encode and decode URLs automatically.

- [Listing 10.12: Using `base64String`](../../all-listings/10-polymorphic-storage/12-using-base64string.md)

Now we can create and retrieve URLs in Base64-encoded and decoded formats. `Shorten` passes the `URL` as a `base64String` to ExecContext to encode the URL automatically before saving it in the database. Similarly, `Resolve` passes the `uri` as `*base64String` to `Scan` to decode the Base64-encoded URL and inject the decoded URL into the `uri`.

By using `Valuer` and `Scanner`, we extend the capabilities of `sql`. Check out the `uuid` package at https://go.dev/play/p/cDF3Zoiyq5o to see how it uses `Valuer` and `Scanner` to store and retrieve universally unique identifiers (UUIDs) as binary data from the database. Storing them as 16-byte binary data can reduce storage size and might improve performance.