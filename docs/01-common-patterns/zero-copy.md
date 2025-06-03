# Zero-Copy Techniques

When writing performance-critical Go code, how memory is managed often has a bigger impact than it first appears. Zero-copy techniques are one of the more effective ways to tighten that control. Instead of moving bytes from buffer to buffer, these techniques work directly on existing memory—avoiding copies altogether. That means less pressure on the CPU, better cache behavior, and fewer GC-triggered pauses. For I/O-heavy systems—whether you’re streaming files, handling network traffic, or parsing large datasets—this can translate into much higher throughput and lower latency without adding complexity.

## Understanding Zero-Copy

In the usual I/O path, data moves back and forth between user space and kernel space—first copied into a kernel buffer, then into your application’s buffer, or the other way around. It works, but it’s wasteful. Every copy burns CPU cycles and clogs up memory bandwidth. Zero-copy changes that. Instead of bouncing data between buffers, it lets applications work directly with what’s already in place—no detours, no extra copies. The result? Lower CPU load, better use of memory, and faster I/O, especially when throughput or latency actually matter.

## Common Zero-Copy Techniques in Go

### Using `io.Reader` and `io.Writer` Interfaces

Using interfaces like `io.Reader` and `io.Writer` gives you fine-grained control over how data flows. Instead of spinning up new buffers every time, you can reuse existing ones and keep memory usage steady. In practice, this avoids unnecessary garbage collection pressure and keeps your I/O paths clean and efficient—especially when you’re dealing with high-throughput or streaming workloads.

```go
func StreamData(src io.Reader, dst io.Writer) error {
	buf := make([]byte, 4096) // Reusable buffer
	_, err := io.CopyBuffer(dst, src, buf)
	return err
}
```

`io.CopyBuffer` reuses a provided buffer, avoiding repeated allocations and intermediate copies. An in-depth `io.CopyBuffer` explanation is [available on SO](https://stackoverflow.com/questions/71082021/what-exactly-is-buffer-last-parameter-in-io-copybuffer).

### Slicing for Efficient Data Access

Slicing large byte arrays or buffers instead of copying data into new slices is a powerful zero-copy strategy:

```go
func process(buffer []byte) []byte {
	return buffer[128:256] // returns a slice reference without copying
}
```

Slices in Go are inherently zero-copy since they reference the underlying array.

### Memory Mapping (`mmap`)

Using memory mapping enables direct access to file contents without explicit read operations:

```go
import "golang.org/x/exp/mmap"

func ReadFileZeroCopy(path string) ([]byte, error) {
	r, err := mmap.Open(path)
	if err != nil {
		return nil, err
	}
	defer r.Close()

	data := make([]byte, r.Len())
	_, err = r.ReadAt(data, 0)
	return data, err
}
```

This approach maps file contents directly into memory, entirely eliminating copying between kernel and user-space.

## Benchmarking Impact

Here's a basic benchmark illustrating performance differences between explicit copying and zero-copy slicing:


```go
{%
    include-markdown "01-common-patterns/src/zero-copy_test.go"
    start="// bench-start"
    end="// bench-end"
%}
```

In `BenchmarkCopy`, each iteration copies a 64KB buffer into a fresh slice—allocating memory and duplicating data every time. That cost adds up fast. `BenchmarkSlice`, on the other hand, just re-slices the same buffer—no allocation, no copying, just new view on the same data. The difference is night and day. When performance matters, avoiding copies isn’t just a micro-optimization—it’s fundamental.

!!! info
	These two functions are not equivalent in behavior—`BenchmarkCopy` makes an actual deep copy of the buffer, while `BenchmarkSlice` only creates a new slice header pointing to the same underlying data. This benchmark is not comparing functional correctness but is intentionally contrasting performance characteristics to highlight the cost of unnecessary copying.

	| Benchmark                | Time per op (ns) | Bytes per op | Allocs per op |
	|--------------------------|---------|--------|------------|
	| BenchmarkCopy            | 4,246   | 65536 | 1          |
	| BenchmarkSlice           | 0.592   | 0     | 0          |


### File I/O: Memory Mapping vs. Standard Read

We also benchmarked file reading performance using `os.ReadAt` versus `mmap.Open` for a 4MB binary file.

```go
{%
    include-markdown "01-common-patterns/src/zero-copy_test.go"
    start="// bench-io-start"
    end="// bench-io-end"
%}
```

??? info "How to run the benchmark"
	To run the benchmark involving `mmap`, you’ll need to install the required package and create a test file:

	```bash
	go get golang.org/x/exp/mmap
	mkdir -p testdata
	dd if=/dev/urandom of=./testdata/largefile.bin bs=1M count=4
	```

Benchmark Results

| Benchmark                | Time per op (ns) | Bytes per op | Allocs per op |
|--------------------------|---------|------|------------|
| ReadWithCopy             | 94,650  | 0    | 0          |
| ReadWithMmap             | 50,082  | 0    | 0          |

The memory-mapped version (`mmap`) is nearly 2× faster than the standard read call. This illustrates how zero-copy access through memory mapping can substantially reduce read latency and CPU usage for large files.

??? example "Show the complete benchmark file"
    ```go
    {% include "01-common-patterns/src/interface-boxing_test.go" %}
    ```

## When to Use Zero-Copy

:material-checkbox-marked-circle-outline: Zero-copy techniques are highly beneficial for:

- Network servers handling large amounts of concurrent data streams. Avoiding unnecessary memory copies helps reduce CPU usage and latency, especially under high load.
- Applications with heavy I/O operations like file streaming or real-time data processing. Zero-copy allows data to move through the system efficiently without redundant allocations or copies.

!!! warning
	:fontawesome-regular-hand-point-right: Zero-copy isn’t a free win. Slices share underlying memory, so reusing them means you’re also sharing state. If one part of your code changes the data while another is still reading it, you’re setting yourself up for subtle, hard-to-track bugs. This kind of shared memory requires discipline—clear ownership and tight control. It also adds complexity, which might not be worth it unless the performance gains are real and measurable. Always benchmark before committing to it.

### Real-World Use Cases and Libraries

Zero-copy strategies aren't just theoretical—they're used in production by performance-critical Go systems:

- [fasthttp](https://github.com/valyala/fasthttp): A high-performance HTTP server designed to avoid allocations. It returns slices directly and avoids `string` conversions to minimize copying.
- [gRPC-Go](https://github.com/grpc/grpc-go): Uses internal buffer pools and avoids deep copying of large request/response messages to reduce GC pressure.
- [MinIO](https://github.com/minio/minio): An object storage system that streams data directly between disk and network using `io.Reader` without unnecessary buffer replication.
- [Protobuf](https://github.com/protocolbuffers/protobuf) and [MsgPack](https://github.com/vmihailenco/msgpack) libraries: Efficient serialization frameworks like `google.golang.org/protobuf` and `vmihailenco/msgpack` support decoding directly into user-managed buffers.
- [InfluxDB](https://github.com/influxdata/influxdb) and [Badger](https://github.com/hypermodeinc/badger): These storage engines use `mmap` extensively for fast, zero-copy access to database files.

These libraries show how zero-copy techniques help reduce allocations, GC overhead, and system call frequency—all while increasing throughput.
