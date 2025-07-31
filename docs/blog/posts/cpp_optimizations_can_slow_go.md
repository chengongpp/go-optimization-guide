---
date:
  created: 2025-07-31
categories:
  - optimizations
---

# When C++ Optimization Slows Down Your Go Code

When you have years of C++ experience, you definitely obtain some habits. These habits are good for C++, but could cause you some surprises in Go. In C++, you usually preallocate everything, avoiding unnecessary allocations, caching values aggressively, and always thinking of CPU cache misses. So when I rewrote a simple algorithm in Go—finding the number of days until the next warmer temperature—I reached for the same tricks. But this time, they backfired.

Here’s how applying familiar C++ optimizations ended up making my Go code slower and heavier.
<!-- more -->

## The C++ Context

The original problem: for each day, figure out how many days pass until a warmer temperature appears. A classic use case for a monotonic stack. Here's the performance data [from a C++ implementation](https://github.com/astavonin/perf-tests/blob/main/daily-temps/cpp/daily_temperatures.cpp):

??? example "dailyTemperatures, basic implementation"
    ```cpp
	std::vector<int> dailyTemperatures( const std::vector<int> &temperatures )
	{
	    std::vector<int> result( temperatures.size(), 0 );
	    std::stack<int>  s;
	    for( int i = 0; i < temperatures.size(); ++i ) {
	        while( !s.empty() && temperatures[i] > temperatures[s.top()] ) {
	            int prev = s.top();
	            s.pop();
	            result[prev] = i - prev;
	        }
	        s.push( i );
	    }
	    return result;
	}
    ```
??? example "dailyTemperatures, optimized implementation"
    ```cpp
	std::vector<int> dailyTemperaturesOpt( const std::vector<int> &temperatures )
	{
	    std::vector<int> res( temperatures.size(), 0 );
	    std::vector<int> track; // (1)
	    track.reserve( temperatures.size() ); // (2)

	    for( int i = 0; i < temperatures.size(); ++i ) {
	        int currTemp = temperatures[i]; // (3)
	        while( !track.empty() && currTemp > temperatures[track.back()] ) {
	            int prev = track.back();
	            track.pop_back();
	            res[prev] = i - prev;
	        }
	        track.push_back( i );
	    }
	    return res;
	}
    ```
	{ .annotate }

	1. `std::vector` is used instead of `std::stack` for better control and performance. Unlike `std::stack`, `std::vector` allows preallocation, random access, and avoids the overhead of an adapter layer. It's more cache-friendly and directly exposes the underlying data layout.

	2. We reserve the full capacity of the `track` stack to avoid dynamic memory reallocations during growth. Since the number of pushed elements can never exceed `temperatures.size()`, this is a safe and efficient optimization.

	3. `currTemp` holds the value of `temperatures[i]` so we don’t hit the same memory location multiple times inside the loop. While the access pattern is sequential and usually cache-friendly, reading the same slice element repeatedly adds work for the compiler and the CPU. By assigning it to a local variable, we give the compiler a clear signal that this value won’t change. That usually means it stays in a register, which can reduce instruction count and make the inner loop tighter—especially when you’re benchmarking or pushing for low-latency behavior.

| Benchmark                        | Time per op (ns) |
|----------------------------------|------------------|
| BM_DailyTemperatures/100000      | 206,340          |
| BM_DailyTemperaturesOpt/100000   | 115,490          |

Standard C++ optimization tactics can cut the runtime almost in half. But even if something works in C++ pretty well, it does not mean that the same approach will not make your Go code slower.

## Translating to Go

Here’s what a clean idiomatic Go version looks like:

```go
func DailyTemperatures(temperatures []int) []int {
	result := make([]int, len(temperatures))
	var stack []int

	for i, temp := range temperatures {
		for len(stack) > 0 && temp > temperatures[stack[len(stack)-1]] {
			prevIndex := stack[len(stack)-1]
			stack = stack[:len(stack)-1]
			result[prevIndex] = i - prevIndex
		}
		stack = append(stack, i)
	}

	return result
}
```

Pretty typical: no preallocation, straightforward stack growth via append.

I rewrote this with “optimizations”: preallocate the stack slice, replace variables early, and reduce slice bounds checks. Classic C++-style low-level thinking.

```go
func DailyTemperaturesOpt(temperatures []int) []int {
	n := len(temperatures)
	result := make([]int, n)
	stack := make([]int, 0, n/4) // (1)

	for i := 0; i < n; i++ {
		curr := temperatures[i] // (2)
		for len(stack) > 0 && curr > temperatures[stack[len(stack)-1]] {
			prev := stack[len(stack)-1]
			stack = stack[:len(stack)-1]
			result[prev] = i - prev
		}
		stack = append(stack, i)
	}
	return result
}
```
{ .annotate }

1.	The `stack` is preallocated with a capacity of `n/4`, not the full `n`, to strike a balance between reducing allocations and avoiding unnecessary heap growth. In Go, over-allocation can backfire due to GC sensitivity and increased memory footprint. A smaller initial capacity avoids paying for memory that may never be used. **NOTE:** The result will be even worse if we preallocate `len(temperatures)` here.
2.	`curr` stores `temperatures[i]` so we don’t keep looking it up inside the loop. Go’s compiler might optimize that on its own, but being explicit gives it less to guess about. It also helps the value stay in a register instead of bouncing back to memory, which can matter in tight loops. If you’re running benchmarks or chasing small gains, this kind of local caching can reduce register pressure and cut down on subtle overhead.

And the Result? It got worse.

| Benchmark                                 | Time per op (ns) | Bytes per op | Allocs per op |
|-------------------------------------------|------------------|---------------|----------------|
| BenchmarkDailyTemperatures/Baseline-14    | 174,419          | 862,847       | 15             |
| BenchmarkDailyTemperatures/Optimized-14   | 175,021          | 1,007,620     | 2              |

Same logic, but now slower and heavier. One fewer allocation, but an extra 150KB of memory usage. Why? Because in Go, allocating a 100,000-capacity slice (even if you barely use it) is expensive. The runtime doesn’t treat that lightly.

## Why Go Behaves Differently

The runtime is more opinionated. Memory is GC-managed. There’s no real benefit to preallocating more than you need, especially if your code doesn’t end up using it. `append()` is cheap, and in many cases more efficient than second-guessing the allocator.

On top of that, Go’s escape analysis doesn’t work the way C++’s stack allocation does. What you think is local might end up on the heap, just because of one indirect reference.

Key Takeaways

* Preallocation helps in C++ because memory layout and growth are under your control. In Go, the runtime handles it differently.
* Trust the idioms of the language. If you’re writing Go, let Go be Go.
* Measure everything. Some optimizations only look good on paper—or in other languages.

---

I still write C++. I still optimize memory, inlining, and stack frames. But when I’m in Go, I’ve learned to lean into the model that Go is designed for. Sometimes performance comes from understanding how much less you need to do—not how much you can tweak.
