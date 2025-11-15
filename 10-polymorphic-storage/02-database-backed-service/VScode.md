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
  * `${workspaceFolder}`: `10-polymorphic-storage\02-database-backed-service`
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
## 10.2 Database-backed service
Now that we can use Dial to connect to SQLite, we’ll use it in our link service. First, though, let’s revisit the link.Shortener service type from chapter 8. The HTTP handlers in our link/rest package use it to save and retrieve links from its map field:
```golang
package link
type Shortener struct { . . .links map[Key]Link }
func (*Shortener) Shorten(context.Context, Link) (Key, error) {. . .}
func (*Shortener) Resolve(context.Context, Key) (Link, error) {. . .}
```
Our next goal is to persist links with an SQLite-backed `Shortener` service. Figure 10.4 shows the new service. It’s similar to `link.Shortener` but can interact with SQLite.

![architecture diagram](pics/CH10_F04_Gumus.drawio.svg)   
Figure 10.4 Handlers use `Shortener` to shorten and resolve links from SQLite; internally, `Shortener` uses a `*DB` handle to execute queries on the database.

The new `Shortener` lives in the `link/sqlite` package. Similar to `link.Shortener`, `sqlite.Shortener` has the following methods to shorten and resolve links:
* `Shorten` shortens and inserts a link into the database using `DB.ExecContext`.
* `Resolve` uses `DB.QueryRowContext` to resolve a link from the database.

After we implement the new `Shortener` service, we’ll modify the `link/rest` package’s handlers to support the `link.Shortener` and `sqlite.Shortener` services.

### 10.2.1 Insertion
The `link.Shortener` type didn’t require a constructor because its zero value was useful. As the following listing shows, however, we’ll add a constructor and a `*DB` field to the `sqlite.Shortener` type because it uses a `*DB` to interact with SQLite. After it shortens a link, it uses its `*DB` field to insert the link into the database using `DB.ExecContext`.
- [Listing 10.5: Shortening links](../../all-listings/10-polymorphic-storage/05-shortening-links.md)

Our `Shortener` type has a DB field that we can initialize using the `NewShortener` function. The `Shorten` method takes a `Context. ExecContext` will cancel the database operation after that `Context` is canceled, which might happen if, say, an HTTP client disconnects.

> [!TIP]
> Pass `Context` downstream to allow the cancellation of ongoing operations.

After shortening the link, we execute the SQL query on the database using `Exec­Context`. The dollar signs in the query are placeholders, allowing us to substitute the Key and URL. This way, we avoid handcrafted SQL parameters, making SQL injection attacks less likely. Although we use standard SQL, the placeholder syntax is specific to a database. Still, our SQL query is PostgreSQL-compatible, but it may be incompatible with others (e.g., MySQL).

> [!TIP] 
> The community-provided `sqlx` package can generate placeholders that work across SQL dialects. Visit https://github.com/jmoiron/sqlx to learn more.

### 10.2.2 Test database
Now that we can persist links in SQLite, let’s test our new `sqlite.Shortener` service. First, we’ll add a test helper to get a unique in-memory SQLite database for convenience, as shown in listing 10.6. We’ll also call `testing.TB.Cleanup` to close the connection pool after the caller test ends. As discussed earlier, we generally don’t need to close a connection pool. We may not need to take advantage of a pool in tests, however, because we might run many isolated tests that don’t share a pool.
- [Listing 10.6: Adding a test database helper](../../all-listings/10-polymorphic-storage/06-adding-a-test-database-helper.md)

Our `DialTestDB` helper gives the caller test a unique in-memory database (`mode=memory`) that is shared (`cache=shared`) by all database connections in the same test. The TB.Name method returns the caller test’s name (e.g., "`TestFoo`" if `TestFoo` calls `DialTestDB`). This way, we can run future tests concurrently. Each database operation in the same test uses the same database as long as the test calls `DialTestDB` once. We could have set the database name with a random identifier, but using the test name is good enough for our current purposes.

> [!TIP] 
> See chapter 8 for more information about test helpers.
 
Calling `Cleanup` is useful for after-test cleanup. It’s similar to the `defer` statement, but there is a difference: a function that we register with `Cleanup` is called when the parent test (the one that calls the test helper) finishes rather than when the test helper, such as `Dial­TestDB`, returns. Otherwise, if we used a `defer` statement, the database connection would close prematurely before the test ends.

### 10.2.3 Integration testing
Now that we can create unique test databases for each test, we can test the `Shortener.Shorten` method using our new `sqlite.DialTestDB` test helper. As the next listing shows, we test whether Shorten can `shorten` a link and return the correct key. Near the end, we verify that `Shorten` disallows shortening links with duplicate keys.
- [Listing 10.7: Testing `Shortener.Shorten`](../../all-listings/10-polymorphic-storage/07-testing-shortenershorten.md)

We’ve added a test for the Shorten method to test it against a new in-memory database.

> [!NOTE] 
> For simplicity, we use multiple assertions in the test, but it can be more effective to run subtests for each assertion. See chapter 2 for more information.

We connect to a unique database and pass the `*DB` handle to `NewShortener` to save the handle in the `Shortener.DB` field. Then we call the `Shorten` method. `Shorten` requires us to pass it a `Context`. Instead of creating a new one, we use `Context`, which is canceled automatically when the test finishes (but before any T.Cleanup functions run). Using `T.Context` ties everything to the test’s lifetime and eliminates manual `Context` management. Finally, we verify that `Shorten` can shorten a link and disallow duplicate links. Let’s run the test to see whether `Shorten` works correctly:
```bash
$ go test ./link/sqlite -run=TestShortenerShorten -v
got err = saving: constraint failed: UNIQUE constraint failed:
➥                                   links.short_key (1555)
want ErrConflict for duplicate key
```

The test fails because `Shorten` cannot detect duplicate link keys. SQLite returns a constraint error because the `short_key` column is a primary key, and we try to add a duplicate key. Even worse, `Shorten` exposes sensitive database-specific details in the error message. We’ll fix these issues by returning our application-specific `link.ErrConflict` from `Shorten` instead of the driver’s error. But first, we need to detect the driver’s error using the `errors.As` function and return `Err­Conflict` on duplicate keys.

### 10.2.4 errors.As
We can use the standard library’s `errors.As` function to detect whether the returned error is an SQLite driver error and then check whether the error code is `1555`— a unique-key violation. Detecting specific driver errors allows us to return more meaningful application-level errors and prevents accidental leaks of sensitive internal database details when errors occur.

> [!TIP] 
> `As` extracts a specific `error` from an error chain and assigns it to a variable.

We’ll add a new convenience function for isolating SQLite-specific error-detection code before detecting duplicate keys using the `Shorten` method. As the following listing shows, we use `errors.As` to extract the driver error from the error chain. Then we query for the error code and return `true` if the code is `1555`.
- [Listing 10.8: Detecting constraint errors](../../all-listings/10-polymorphic-storage/08-detecting-constraint-errors.md)

We change the side-effect import to a regular import, which enables us to use the `sqlite` package name in the `sqlite.go` file. Now that we can find primary-key errors, let’s update Shorten to identify duplicate keys. If we find any, we’ll return our `ErrConflict` error. Let’s check the primary-key violation error to prevent `Shorten` from returning an internal error.

- [Listing 10.9: Detecting duplicate keys](../../all-listings/10-polymorphic-storage/09-detecting-duplicate-keys.md)

Let’s test whether `Shorten` can detect duplicate keys now:
```bash
$ go test ./link/sqlite -run=TestShortenerShorten -v
--- PASS: TestShortenerShorten. . .
```
`Shorten` successfully detected the key conflicts and returned `ErrConflict`. Later, our HTTP handlers will forward `ErrConflict` to clients instead of an SQLite-specific internal error that clutters our logs and can be confusing. Now we return an informative ErrConflict message, telling them that the key already exists so they can retry with another key.

### 10.2.5 Retrieval
Having looked at inserting links into the database, let’s explore retrieving a single row from SQLite using our new `Shortener`’s `Resolve` method. When we want to pull a single row from the database, we use the `DB.QueryRowContext` method.

As figure 10.5 shows
* `Resolve` calls `QueryRowContext` to query the database.
* `QueryRowContext` queries the database through a connection (existing or new).
* `Scan` injects the URL into the `uri` variable and returns the connection to the pool.
  
> [!TIP]
> If we didn’t call Scan, we would risk leaking connections. `Scan` reduces resource use by allowing other queries to reuse the same connection from the pool.

In short, `Resolve` calls `QueryRowContext` and then `Scan` to assign the URL to `uri`.

![architecture diagram](pics/CH10_F05_Gumus.drawio.svg)
Figure 10.5 `Resolve` calls `QueryRowContext` to retrieve the URL from the database. The connection is reserved until `Scan` is called. Calling `Scan` copies the previously loaded raw row data into the `uri` variable and returns the connection to the pool, allowing other database tasks to reuse the connection.

Now that we understand how `DB.QueryRowContext` works, let’s use it. The following listing adds a new `Resolve` method to the `Shortener` type.
- [Listing 10.10: Resolving links](../../all-listings/10-polymorphic-storage/10-resolving-links.md)

`Resolve` checks whether the shortened key is valid and then queries the `links` table for a key to get the link’s original URL. Next, it scans the URL into the `uri` variable by passing that variable’s pointer to `Scan`, releasing the connection for later reuse. Finally, it returns a `Link`.

`Scan` returns `ErrNoRows` if the short key doesn’t exist in the database. Finally, we return this error as our `ErrNotFound`. This approach prevents our code from depending on `sql.ErrNoRows` because not every `Shortener` (e.g., `link.Shortener`) is SQL-dependent. Instead, all `Shortener` implementations return consistent errors, enabling consumer code (such as our HTTP handlers) to handle different `Shortener`s uniformly without code changes. This approach will also help us to unlock consumer-driven interfaces, as explained in section 10.4.

We’ve added a new `Shortener` service that works with SQLite:
* `Shorten` uses `ExecContext` to insert a link into the database.
* `Resolve` uses `QueryRowContext` and `Scan` to fetch a URL from the database.
  
Both are protected against SQL injection attacks using placeholders.