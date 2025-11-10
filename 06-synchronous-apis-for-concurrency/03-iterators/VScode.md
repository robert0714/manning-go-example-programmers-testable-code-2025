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
  * `${workspaceFolder}`: `06-synchronous-apis-for-concurrency\03-iterators`
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

 
# 6 Synchronous APIs for concurrency
## 6.3 Iterators
### 6.3.1 Push iterators
#### DECLARING A CUSTOM ITERATOR TYPE
- [Listing 6.3: Implementing the `Results` iterator](../../all-listings/06-synchronous-apis-for-concurrency/03-implementing-the-results-iterator.md)
  
This function type declares a push iterator that will allow `SendN` to push each `Result` value to consumers. Next, we’ll start explaining the `iter.Seq` type. We’ll discuss how to return a `Results` iterator from the `SendN` and push values from the iterator to consumers.
#### USING PUSH ITERATORS
Our custom push iterator is based on the following iter.Seq type. Push iterators push values from a sequence of values into a yield function that the iterator’s consumer provides. The return value of this yield function tells the iterator to push more values or stop:
```golang
package iter  #1
type Seq[V any] func(yield func(V) bool)  #2
```
#1 Declared in the standard library’s iter package  
#2 Declares a generic function type  
> [!TIP]DEFINITION
> `Seq` is short for sequence and pronounced “seek.”

`Seq` is a generic type representing a push iterator that can push values to consumers. The type parameter, `V`, can be any type. Think of `V` as a placeholder for any type. In our case, we use the `[Result]` type next to `Seq` to instantiate an iterator for `Result` values:
```golang
type Results iter.Seq[Result]  #1
```
#1 Declares an iterator type that can iterate on Result values  
This expression, `iter.Seq[Result]`, is identical to the following type:
```golang
type Results func(yield func(Result) bool)
```
This function type takes another function named `yield` as input. The `yield` function takes a `Result` value and returns a `bool`. This iterator allows us to push `Result` values to consumers.

> [!NOTE]
>  Consumers provide a `yield` function to an iterator to receive values.

#### PUSHING VALUES TO CONSUMERS
Now that we know what the `Seq` and `Results` types look like, let’s go deeper into how to produce and consume values. In our case, `SendN` returns the `Results` iterator, and consumers provide the `yield` function to the iterator. The iterator generates a `Result` for each request and then calls the consumer’s `yield` function, pushing that `Result` to the consumer. The iterator continues doing so until it pushes each `Result` or the consumer’s `yield` returns `false`. Figure 6.2 shows how to produce and consume `Result` values using the `Results` iterator:
1. Calling `SendN` returns a `Results` iterator as a closure.
2. Consumers pass their `yield` function to this closure.
3. The closure calls `yield` to push the next `Result` to the consumer.
4. Step 3 repeats until all values have been pushed or `yield` returns `false`.

![architecture diagram](pics/CH06_F02_Gumus.drawio.svg)   
Figure 6.2 The `Results` iterator pushes values by calling the consumer’s `yield`.
### 6.3.2 Producing values
Now that we’ve implemented the `Results` iterator type and discussed how iterators push values to consumers using a yield function, we’re ready to implement the `SendN` function. By implementing `SendN`, we’ll learn how to push `Result` values through an iterator. In section 6.3.3, we’ll add the `Summarize` function to consume these values.

As listing 6.4 shows, `SendN` returns an iterator. This iterator sends HTTP requests using `Send`, pushing each `Result` to the consumer’s `yield` function to deliver it to the consumer.

> [!NOTE]
> Chapter 7 explains why using the `http.DefaultClient` isn’t ideal.

- [Listing 6.4: Returning a push iterator](../../all-listings/06-synchronous-apis-for-concurrency/04-returning-a-push-iterator.md)

`Send` returns a `Result`, whereas `SendN` returns a `Results` iterator, which allows consumers to walk over each `Result`. The iterator stops when `yield` returns `false` or the iterator’s loop ends.

As we saw in section 6.3.1, the underlying type of `iter.Seq[Result]` is `func(yield func(Result) bool)`. That’s why we return a closure with the same signature from the `SendN` function as a `Results` iterator. Because this closure is a higher-order function that takes a `yield` function, consumers can pass a `yield` function to the closure.

We push a `Result` value to the consumer by calling the `yield` function and receive a `bool` value from the consumer indicating whether to produce more values. Because the consumer passes the `yield`, they can stop our iterator by returning `false` from their `yield` function. We’ll see this behavior in action after implementing the `Summarize` function as a consumer in section 6.3.3.

For now, `SendN`’s iterator processes requests sequentially, waiting for one to complete before sending the next, until we refactor `SendN` to send requests concurrently. This approach forces us to keep the `SendN`’s API sequential even after we switch to a concurrent pipeline.

### 6.3.3 Consuming values
#### SUMMARY
- [Listing 6.5: `Summary` type](../../all-listings/06-synchronous-apis-for-concurrency/05-summary-type.md)

#### SUMMARIZING
- [Listing 6.6: Summarizing the results](../../all-listings/06-synchronous-apis-for-concurrency/06-summarizing-the-results.md)
> [!NOTE]
> Iterating with a `nil` iterator results in a panic. `Summarize` prevents this panic by checking for a nil iterator. If `results` are `nil`, `Summarize` returns a zero `Summary`.

### 6.3.4 Testing
- [Listing 6.7: Testing `Summarize`](../../all-listings/06-synchronous-apis-for-concurrency/07-testing-summarize.md)
```bash
$ go test ./hit -v
--- PASS: TestSummarizeFastestResult
--- PASS: TestSummarizeNilResults
```
> [!NOTE]
> `slices.Values` returns an iterator that yields the given slice’s elements.

