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
  * `${workspaceFolder}`: `03-test-coverage-and-optimization\02-benchmarking-and-optimization`
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

 
# 3.2 Benchmarking and optimization 
Now that we’re familiar with benchmarking, let’s create [one for our String method](./url/url_test.go#L146C1-L155C1). 
```bash
$ go test ./url -bench=. -benchmem
goos: windows
goarch: amd64
pkg: github.com/inancgumus/gobyexample/url
cpu: AMD Ryzen 5 5600G with Radeon Graphics
BenchmarkURLString-12           37129286                29.09 ns/op           32 B/op          1 allocs/op
```
The `BenchmarkURLString` function ran on `12` CPU cores and returned the following results:
* The benchmark called String about `37` million times.
* Each operation took an average of `29.09` nanoseconds.
* Each operation requested 32 bytes of heap memory in total by four calls.

We can also change the default benchmark time using the benchtime flag:
```bash
$ go test ./url -bench=. -benchmem -benchtime=2s
BenchmarkURLString-12           77828284                31.09 ns/op           32 B/op          1 allocs/op
```
The benchmark made about 77 million calls. Other measurements per operation are similar to those of the preceding run because we didn’t change anything except the benchmark duration.

The `benchtime` flag controls how long benchmarks run and provides more accurate results by ensuring that performance is measured over a longer period. Also, it may minimize the effect of environmental noise factors (such as machine load).


> [!TIP]
> Tests are run before benchmarks (and we can use the `-v` flag to prove it). We can run benchmarks without running tests using an empty `run` flag: `-bench=. -run=' '`.

* [Writing a benchmark for string](../../all-listings/03-test-coverage-and-optimization/08-writing-a-benchmark-for-string.md) 
  
## 3.2.2 Using sub-benchmarks
* [Writing sub benchmarks](../../all-listings/03-test-coverage-and-optimization/09-writing-sub-benchmarks.md) 

```bash
$ go test ./url -bench=BenchmarkURLStringLong -benchmem
goos: windows
goarch: amd64
pkg: github.com/inancgumus/gobyexample/url
cpu: AMD Ryzen 5 5600G with Radeon Graphics
BenchmarkURLStringLong/10-12    36894582                32.97 ns/op           48 B/op          1 allocs/op
BenchmarkURLStringLong/100-12           15800179                71.30 ns/op          320 B/op          1 allocs/op
BenchmarkURLStringLong/1000-12           3087636               389.7 ns/op          3072 B/op          1 allocs/op
PASS
ok      github.com/inancgumus/gobyexample/url   3.598s
```

## 3.2.3 Profiling: Chasing out memory allocations
We have to use original code of [url.go](../01-test-coverage/url/url.go#L39C1-L52C1) .

```bash
$ go test ./url -bench=BenchmarkURLStringLong -benchmem -memprofile=mem.out 
```
Now that we have a memory profile file, we can pass this file to the performance profiling tool (`pprof`) and inspect `String`’s memory allocations using the list flag:
```bash
$ go tool pprof -list String mem.out
   13.01GB    13.01GB (flat, cum)   100% of Total
         .          .     35:func (u *URL) String() string {
         .          .     36:   if u == nil {
         .          .     37:           return ""
         .          .     38:   }
         .          .     39:   var s string
         .          .     40:   if sc := u.Scheme; sc != "" {
         .          .     41:           s += sc #1
    1.70GB     1.70GB     42:           s += "://" #2
         .          .     43:   }
         .          .     44:   if h := u.Host; h != "" {
    3.26GB     3.26GB     45:           s += h #2
         .          .     46:   }
         .          .     47:   if p := u.Path; p != "" {
    3.12GB     3.12GB     48:           s += "/" #3
    4.93GB     4.93GB     49:           s += p #3
         .          .     50:   }
         .          .     51:   return s
         .          .     52:}
ROUTINE ======================== github.com/inancgumus/gobyexample/url.BenchmarkURLStringLong.func1 in D:\Data\workspaces\golang\manning-go-example-programmers-testable-code-2025\03-test-coverage-and-optimization\02-benchmarking-and-optimization\url\url_test.go
         0    13.01GB (flat, cum)   100% of Total
         .          .    164:           b.Run(fmt.Sprintf("%d", n), func(b *testing.B) {
         .          .    165:                   for b.Loop() {
         .    13.01GB    166:                           _ = u.String()
         .          .    167:                   }
         .          .    168:           })
         .          .    169:   }
         .          .    170:}
```

#1 Initialization does not allocate because we change only string headers (not the underlying bytes). See [appendix c](../../all-listings/ac-arrays-slices-and-maps/).

#2 Each concatenation allocates new memory because strings are immutable.

#3 Each concatenation allocates new memory because strings are immutable.


Before optimizing `String`, let’s save the current results to a file so we can compare them with the new results later. Because environmental noise (system load, hardware temperature, and so on) may affect the results, we should run benchmarks repeatedly to minimize environmental impact. For that purpose, we can use the `count` flag:
```bash
$ go test ./url -bench=BenchmarkURLStringLong -benchmem -count=20 > old.benchmark
```
This command runs the benchmark function 20 times. If we’re unsatisfied with the results, we can repeat with a larger number.

> [!TIP]
> See https://go.dev/doc/diagnostics for more about diagnosing Go programs.

## 3.2.4 Optimizing code
* [Optimizing the string method](../../all-listings/03-test-coverage-and-optimization/10-optimizing-the-string-method.md)

Now let’s optimize the `String` method. Instead of producing more work for the garbage collector and slowing `String` by making memory allocation requests from the operating system each time we combine `strings`, we can use the `strings` package’s `Builder` type to combine strings. This type accumulates bytes in its internal buffer without making additional allocations. [The new `String` code uses a `Builder` to reassemble a URL string](./url/url.go#L39C1-L64C1) .

### 3.2.5 Comparing benchmarks
We already have the old results in the old.benchmark file. Let’s benchmark the new implementation and save:
```bash
$  go test ./url -bench=BenchmarkURLStringLong -benchmem -count=20 > new.benchmark
```
The next step is using the `benchstat` tool to compare the results. Unlike other Go tools, this one doesn’t come with the Go tool chain out of the box, so we must install it ourselves:
```bash
$ go install golang.org/x/perf/cmd/benchstat@latest
```
Now we can compare the old and new results:
```bash
$  benchstat old.benchmark new.benchmark
goos: windows
goarch: amd64
pkg: github.com/inancgumus/gobyexample/url
cpu: AMD Ryzen 5 5600G with Radeon Graphics
                      │ old.benchmark │            new.benchmark            │
                      │    sec/op     │   sec/op     vs base                │
URLStringLong/10-12      107.65n ± 3%   33.52n ± 4%  -68.86% (p=0.000 n=20)
URLStringLong/100-12     242.30n ± 2%   70.54n ± 2%  -70.89% (p=0.000 n=20)
URLStringLong/1000-12    1520.0n ± 5%   413.8n ± 2%  -72.77% (p=0.000 n=20)
geomean                   341.0n        99.28n       -70.88%

                      │ old.benchmark │            new.benchmark             │
                      │     B/op      │     B/op      vs base                │
URLStringLong/10-12       112.00 ± 0%     48.00 ± 0%  -57.14% (p=0.000 n=20)
URLStringLong/100-12       848.0 ± 0%     320.0 ± 0%  -62.26% (p=0.000 n=20)
URLStringLong/1000-12    8.000Ki ± 0%   3.000Ki ± 0%  -62.50% (p=0.000 n=20)
geomean                    919.7          361.4       -60.71%

                      │ old.benchmark │           new.benchmark            │
                      │   allocs/op   │ allocs/op   vs base                │
URLStringLong/10-12        4.000 ± 0%   1.000 ± 0%  -75.00% (p=0.000 n=20)
URLStringLong/100-12       4.000 ± 0%   1.000 ± 0%  -75.00% (p=0.000 n=20)
URLStringLong/1000-12      4.000 ± 0%   1.000 ± 0%  -75.00% (p=0.000 n=20)
geomean                    4.000        1.000       -75.00%
```
We benchmarked the old and new `String` methods and see 3X improvement compared with the previous version:
* Generating URL strings drops from 341 nanoseconds (ns) to 99 ns.
* Memory per operation drops from 919 bytes to 361 bytes.
* Memory allocations drop from four to one.
> [!NOTE]
>  A nanosecond is a billionth of a second.

Generating URLs for different URL lengths takes the following times:

Generating 10-byte URLs drops from 107 ns to 33 ns.
Generating 100-byte URLs drops from 242 ns to 70 ns.
Generating 1,000-byte URLs drops from 1520 ns to 413 ns.

These improvements result in faster performance and more efficient memory use. The old method rebuilt strings with `+` on each step, allocating new buffers and scattering data across the heap. The new version reserves one contiguous slice per call, which fits in the CPU’s L1 cache. With only one allocation, Go’s garbage collector scans far fewer objects.

Suppose that we call `String` in a high-load service. With the new code, even the slowest processing (tail latency) finishes faster, and memory use stays low, leading to more consistent performance under load and fewer garbage-collection pauses. We’ll stick with this implementation.
> [!TIP]
> This output is simplified for clarity. See `benchstat`’s documentation at https://mng.bz/qROw for more information on formatting.

