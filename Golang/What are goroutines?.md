
## What are goroutines?

A **goroutine** is a lightweight unit of execution managed by the Go runtime, not directly by the operating system.

In simple terms: goroutines let your Go program do many things at the same time.

Go was designed with concurrency in mind, and goroutines are one of its main features.

---

## Basic example

```go
package main

import (
	"fmt"
	"time"
)

func sayHello() {
	fmt.Println("Hello from goroutine")
}

func main() {
	go sayHello()

	time.Sleep(1 * time.Second)
	fmt.Println("Hello from main")
}
```

The keyword `go` starts a function in a new goroutine:

```go
go sayHello()
```

This means `sayHello()` runs concurrently with the rest of the program.

---

## Goroutines vs threads

Goroutines are often compared to OS threads, but they are much lighter.

| Feature | Goroutine | OS Thread |
|---|---|---|
| Managed by | Go runtime | Operating system |
| Creation cost | Very low | Higher |
| Memory usage | Small stack, grows dynamically | Usually larger fixed stack |
| Quantity | You can create thousands or millions | Usually limited |
| Scheduling | Go scheduler | OS scheduler |

A goroutine may start with only a few kilabytes of stack memory, while OS threads often use much more.

---

## How goroutines are scheduled

Go uses a scheduler inside the runtime. It multiplexes many goroutines onto a smaller number of OS threads.

A simplified model is:

```text
Goroutine 1 \
Goroutine 2  --> Go scheduler --> OS threads --> CPU cores
Goroutine 3 /
```

You do not usually need to manage threads manually. The Go runtime handles scheduling.

---

## Anonymous goroutines

You can start a goroutine using an anonymous function:

```go
package main

import "fmt"

func main() {
	go func() {
		fmt.Println("Running in a goroutine")
	}()

	fmt.Println("Running in main")
}
```

Be careful: when `main()` exits, the program exits, even if other goroutines are still running.

---

## Waiting for goroutines to finish

If you start goroutines and immediately exit `main`, they may not complete.

Bad example:

```go
func main() {
	go doWork()
	// program may exit before doWork finishes
}
```

A common way to wait is using `sync.WaitGroup`.

### Example with WaitGroup

```go
package main

import (
	"fmt"
	"sync"
)

func worker(id int, wg *sync.WaitGroup) {
	defer wg.Done()

	fmt.Println("Worker", id, "starting")
}

func main() {
	var wg sync.WaitGroup

	for i := 1; i <= 5; i++ {
		wg.Add(1)
		go worker(i, &wg)
	}

	wg.Wait()
	fmt.Println("All workers finished")
}
```

Explanation:

- `wg.Add(1)` increases the counter.
- `wg.Done()` decreases it.
- `wg.Wait()` blocks until the counter becomes zero.

---

## Communicating between goroutines

Go encourages communication through **channels**.

A channel allows goroutines to send and receive values safely.

### Basic channel example

```go
package main

import "fmt"

func main() {
	ch := make(chan string)

	go func() {
		ch <- "Hello from goroutine"
	}()

	msg := <-ch
	fmt.Println(msg)
}
```

Here:

```go
ch := make(chan string)
```

creates a channel that carries strings.

```go
ch <- "Hello"
```

sends a value into the channel.

```go
msg := <-ch
```

receives a value from the channel.

---

## Buffered channels

By default, channels are unbuffered. That means sending blocks until another goroutine is ready to receive.

You can create a buffered channel:

```go
ch := make(chan int, 2)
```

Example:

```go
package main

import "fmt"

func main() {
	ch := make(chan int, 2)

	ch <- 1
	ch <- 2

	fmt.Println(<-ch)
	fmt.Println(<-ch)
}
```

The buffer allows sends to happen without an immediate receiver, up to the buffer size.

---

## Closing channels

You can close a channel when no more values will be sent.

```go
package main

import "fmt"

func main() {
	ch := make(chan int)

	go func() {
		for i := 1; i <= 5; i++ {
			ch <- i
		}
		close(ch)
	}()

	for value := range ch {
		fmt.Println(value)
	}
}
```

`range ch` keeps receiving values until the channel is closed.

Important rule:

- Only the sender should close a channel.
- Do not close a channel if receivers may still send to it.
- Closing a channel twice causes a panic.

---

## Select statement

The `select` statement lets a goroutine wait on multiple channel operations.

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	ch1 := make(chan string)
	ch2 := make(chan string)

	go func() {
		time.Sleep(1 * time.Second)
		ch1 <- "from ch1"
	}()

	go func() {
		time.Sleep(2 * time.Second)
		ch2 <- "from ch2"
	}()

	for i := 0; i < 2; i++ {
		select {
		case msg1 := <-ch1:
			fmt.Println(msg1)
		case msg2 := <-ch2:
			fmt.Println(msg2)
		}
	}
}
```

`select` is similar to `switch`, but for channels.

---

## Goroutines and shared memory

Goroutines can share variables, but shared mutable state can cause data races.

Bad example:

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	counter := 0

	var wg sync.WaitGroup

	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			counter++
		}()
	}

	wg.Wait()
	fmt.Println(counter)
}
```

This may not print `1000` because multiple goroutines update `counter` at the same time.

Use synchronization.

---

## Using Mutex

A `sync.Mutex` protects shared data.

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	counter := 0
	var mu sync.Mutex
	var wg sync.WaitGroup

	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()

			mu.Lock()
			counter++
			mu.Unlock()
		}()
	}

	wg.Wait()
	fmt.Println(counter)
}
```

This safely prints `1000`.

---

## Go concurrency philosophy

A common Go saying is:

> Do not communicate by sharing memory; instead, share memory by communicating.

This means Go often prefers using channels to pass data between goroutines instead of using locks everywhere.

However, both channels and mutexes are valid tools. Use the one that fits the problem.

---

## Common goroutine mistakes

### 1. Forgetting that goroutines may not finish

```go
func main() {
	go doWork()
}
```

When `main` returns, the program exits.

Use `WaitGroup`, channels, or another synchronization mechanism.

---

### 2. Loop variable capture mistakes

In older versions of Go, this was a common bug:

```go
for i := 0; i < 5; i++ {
	go func() {
		fmt.Println(i)
	}()
}
```

It might print `5` five times.

Safer version:

```go
for i := 0; i < 5; i++ {
	i := i
	go func() {
		fmt.Println(i)
	}()
}
```

Or pass it as an argument:

```go
for i := 0; i < 5; i++ {
	go func(n int) {
		fmt.Println(n)
	}(i)
}
```

Note: In Go 1.22 and later, loop variable semantics changed, making the original loop safer, but many codebases still use the explicit pattern.

---

### 3. Deadlocks

A deadlock happens when goroutines are waiting forever for each other.

Example:

```go
package main

func main() {
	ch := make(chan int)
	ch <- 1
}
```

This deadlocks because no goroutine is receiving from the channel.

Fix:

```go
package main

import "fmt"

func main() {
	ch := make(chan int)

	go func() {
		ch <- 1
	}()

	fmt.Println(<-ch)
}
```

---

### 4. Data races

Run your program with the race detector:

```bash
go run -race main.go
```

or:

```bash
go test -race ./...
```

This helps detect unsafe concurrent access.

---

## Goroutine lifecycle

A goroutine starts when you use `go`.

```go
go f()
```

It finishes when the function returns.

```go
go func() {
	// work
}()
```

There is no built-in way to forcibly kill a goroutine. Instead, you usually use cancellation patterns, often with `context`.

---

## Using context for cancellation

The `context` package is commonly used to cancel goroutines.

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func worker(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			fmt.Println("Worker stopped")
			return
		default:
			fmt.Println("Working...")
			time.Sleep(500 * time.Millisecond)
		}
	}
}

func main() {
	ctx, cancel := context.WithCancel(context.Background())

	go worker(ctx)

	time.Sleep(2 * time.Second)
	cancel()

	time.Sleep(1 * time.Second)
}
```

When `cancel()` is called, the context’s `Done` channel is closed, allowing goroutines to stop.

---

## Worker pool example

A worker pool limits how many goroutines run at once.

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func worker(id int, jobs <-chan int, results chan<- int, wg *sync.WaitGroup) {
	defer wg.Done()

	for job := range jobs {
		fmt.Println("Worker", id, "processing job", job)
		time.Sleep(1 * time.Second)
		results <- job * 2
	}
}

func main() {
	jobs := make(chan int, 10)
	results := make(chan int, 10)

	var wg sync.WaitGroup

	for w := 1; w <= 3; w++ {
		wg.Add(1)
		go worker(w, jobs, results, &wg)
	}

	for j := 1; j <= 5; j++ {
		jobs <- j
	}
	close(jobs)

	wg.Wait()
	close(results)

	for result := range results {
		fmt.Println("Result:", result)
	}
}
```

This runs 3 workers that process 5 jobs concurrently.

---

## When to use goroutines

Goroutines are useful for:

- Handling many network connections
- Running background tasks
- Processing jobs concurrently
- Building APIs that handle many requests
- Performing I/O-bound work
- Running independent computations in parallel

---

## When not to overuse goroutines

Goroutines are cheap, but not free.

Avoid creating huge numbers of goroutines if:

- They all compete for the same lock
- They create too much scheduling overhead
- The work is tiny
- The program becomes harder to reason about

Concurrency should solve a problem, not make the code confusing.

---

## Key packages for concurrency

Important packages:

```go
sync
sync/atomic
context
time
```

Common types:

```go
sync.WaitGroup
sync.Mutex
sync.RWMutex
sync.Once
sync.Pool
atomic.Int64
context.Context
```

---

## Simple mental model

A goroutine is:

```text
a function that runs concurrently with other code
```

You start one with:

```go
go functionName()
```

You coordinate goroutines using:

- channels
- `sync.WaitGroup`
- `sync.Mutex`
- `context`

---

## Summary

Goroutines are Go’s lightweight way to run functions concurrently.

Basic syntax:

```go
go doSomething()
```

Important points:

- Goroutines are lighter than OS threads.
- The Go runtime schedules them.
- Use channels to communicate safely.
- Use `WaitGroup` to wait for completion.
- Use mutexes to protect shared state.
- Use `context` for cancellation.
- Use `-race` to detect concurrency bugs.