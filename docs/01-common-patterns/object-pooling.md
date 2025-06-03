# Object Pooling

Object pooling helps reduce allocation churn in high-throughput Go programs by reusing objects instead of allocating fresh ones each time. This avoids repeated work for the allocator and eases pressure on the garbage collector, especially when dealing with short-lived or frequently reused structures.

Go’s `sync.Pool` provides a built-in way to implement pooling with minimal code. It’s particularly effective for objects that are expensive to allocate or that would otherwise contribute to frequent garbage collection cycles. While not a silver bullet, it’s a low-friction tool that can lead to noticeable gains in latency and CPU efficiency under sustained load.

## How Object Pooling Works

Object pooling allows programs to reuse memory by recycling previously allocated objects instead of creating new ones on every use. Rather than hitting the heap each time, objects are retrieved from a shared pool and returned once they’re no longer needed. This reduces the number of allocations, cuts down on garbage collection workload, and leads to more predictable performance—especially in workloads with high object churn or tight latency requirements.

### Using `sync.Pool` for Object Reuse

#### Without Object Pooling (Inefficient Memory Usage)
```go
package main

import (
    "fmt"
)

type Data struct {
    Value int
}

func createData() *Data {
    return &Data{Value: 42}
}

func main() {
    for i := 0; i < 1000000; i++ {
        obj := createData() // Allocating a new object every time
        _ = obj // Simulate usage
    }
    fmt.Println("Done")
}
```

In the above example, every iteration creates a new `Data` instance, leading to unnecessary allocations and increased GC pressure.

#### With Object Pooling (Optimized Memory Usage)
```go
package main

import (
    "fmt"
    "sync"
)

type Data struct {
    Value int
}

var dataPool = sync.Pool{
    New: func() any {
        return &Data{}
    },
}

func main() {
    for i := 0; i < 1000000; i++ {
        obj := dataPool.Get().(*Data) // Retrieve from pool
        obj.Value = 42 // Use the object
        dataPool.Put(obj) // Return object to pool for reuse
    }
    fmt.Println("Done")
}
```

### Pooling Byte Buffers for Efficient I/O

Object pooling is especially effective when working with large byte slices that would otherwise lead to high allocation and garbage collection overhead.

```go
package main

import (
    "bytes"
    "fmt"
    "sync"
)

var bufferPool = sync.Pool{
    New: func() any {
        return new(bytes.Buffer)
    },
}

func main() {
    buf := bufferPool.Get().(*bytes.Buffer)
    buf.Reset()
    buf.WriteString("Hello, pooled world!")
    fmt.Println(buf.String())
    bufferPool.Put(buf) // Return buffer to pool for reuse
}
```

Using `sync.Pool` for byte buffers significantly reduces memory pressure when dealing with high-frequency I/O operations.

## Benchmarking Impact

To prove that object pooling actually reduces allocations and improves speed, we can use Go's built-in memory profiling tools (`pprof`) and compare memory allocations between the non-pooled and pooled versions. Simulating a full-scale application that actively uses memory for benchmarking is challenging, so we need a controlled test to evaluate direct heap allocations versus pooled allocations.

??? example "Show the benchmark file"
    ```go
    {% include "01-common-patterns/src/object-pooling_test.go" %}
    ```

| Benchmark               | Iterations  | Time per op (ns) | Bytes per op | Allocs per op |
|-------------------------|-------------|------------------|---------------|----------------|
| BenchmarkWithoutPooling-14 | 1,692,014   | 705.4            | 8,192         | 1              |
| BenchmarkWithPooling-14    | 160,440,506 | 7.455            | 0             | 0              |

The benchmark results highlight the contrast in performance and memory usage between direct allocations and object pooling. In `BenchmarkWithoutPooling`, each iteration creates a new object on the heap, leading to higher execution time and increased memory consumption. This constant allocation pressure triggers more frequent garbage collection, which adds latency and reduces throughput. The presence of nonzero allocation counts per operation confirms that each iteration contributes to GC load, making this approach less efficient in high-throughput scenarios.

## When Should You Use `sync.Pool`?

:material-checkbox-marked-circle-outline: Use sync.Pool when:

- You have short-lived, reusable objects (e.g., buffers, scratch memory, request state). Pooling avoids repeated allocations and lets you recycle memory efficiently.
- Allocation overhead or GC churn is measurable and significant. Reusing objects reduces the number of heap allocations, which in turn lowers garbage collection frequency and pause times.
- The object’s lifecycle is local and can be reset between uses. When objects don’t need complex teardown and are safe to reuse after a simple reset, pooling is straightforward and effective.
- You want to reduce pressure on the garbage collector in high-throughput systems. In systems handling thousands of requests per second, pooling helps maintain consistent performance and minimizes GC-related latency spikes.

:fontawesome-regular-hand-point-right: Avoid sync.Pool when:

- Objects are long-lived or shared across multiple goroutines. `sync.Pool` is optimized for short-lived, single-use objects and doesn’t manage shared ownership or coordination.
- The reuse rate is low and pooled objects are not frequently accessed. If objects sit idle in the pool, you gain little benefit and may even waste memory.
- Predictability or lifecycle control is more important than allocation speed. Pooling makes lifecycle tracking harder and may not be worth the tradeoff.
- Memory savings are negligible or code complexity increases significantly. If pooling doesn’t provide clear benefits, it can add unnecessary complexity to otherwise simple code.