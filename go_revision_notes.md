# Go (Golang) — Complete Revision Notes
> Bootcamp reference covering all 12 phases. Declarations, internals, interview topics, memory tricks.

---

## Table of Contents
1. [Program Structure](#1-program-structure)
2. [Variables, Constants, Declarations](#2-variables-constants-declarations)
3. [Data Types & Type System](#3-data-types--type-system)
4. [Control Flow](#4-control-flow)
5. [Arrays, Slices, Maps](#5-arrays-slices-maps)
6. [Functions](#6-functions)
7. [Pointers](#7-pointers)
8. [Structs, Methods, Interfaces](#8-structs-methods-interfaces)
9. [Goroutines, Channels, Mutex, WaitGroups](#9-goroutines-channels-mutex-waitgroups)
10. [File Operations](#10-file-operations)
11. [HTTP Client](#11-http-client)
12. [REST API Server](#12-rest-api-server)

---

## 1. Program Structure

### Rules — Memorize These
- Every `.go` file starts with `package <name>`
- Entry point is always `package main` + `func main()`
- Unused imports = **compile error** (not a warning)
- Unused variables = **compile error**
- Opening brace `{` must be on the **same line** — never a new line
- No semicolons — the compiler inserts them at line endings

```go
package main        // mandatory first line

import "fmt"        // single import
import (            // grouped import — preferred
    "fmt"
    "os"
    "net/http"
)

func main() {       // brace SAME LINE — mandatory
    fmt.Println("Hello, Go")
}
```

### fmt — Format Verbs Cheat Sheet
| Verb | Meaning | Example Output |
|------|---------|----------------|
| `%d` | decimal integer | `71` |
| `%f` | float | `3.140000` |
| `%s` | string | `hello` |
| `%q` | quoted string | `"hello"` |
| `%v` | default format | anything |
| `%T` | type of variable | `int`, `string` |
| `%b` | binary | `1000111` |
| `%c` | character | `G` |
| `%p` | pointer address | `0xc000014080` |
| `%t` | boolean | `true` |

### Behind the Scenes
- Go compiles to a **single static binary** — no runtime dependencies
- The Go runtime is embedded in the binary — handles garbage collection, goroutine scheduling
- `go build` compiles; `go run` compiles + runs in one step

---

## 2. Variables, Constants, Declarations

### All Declaration Types

```go
// 1. var with explicit type
var name string
var age int = 25

// 2. var with type inference
var city = "Bengaluru"

// 3. Short declaration — INSIDE FUNCTIONS ONLY
name := "Gopher"
age := 25

// 4. Multiple assignment
x, y := 3, 7
a, b, c := 1, "hello", true

// 5. Package-level var block
var (
    AppVersion string = "1.0.0"
    MaxRetries int    = 5
    Debug      bool   = false
)

// 6. Constants
const Pi = 3.14159
const (
    MaxSize = 100
    AppName = "GoApp"
)

// 7. iota — auto-incrementing in const block
const (
    Sunday = iota   // 0
    Monday          // 1
    Tuesday         // 2
    Wednesday       // 3
)

// 8. Blank identifier — discard value
_, err := someFunction()
value, _ := divide(10, 2)
```

### Declaration Rules — Critical Table
| Style | Where | Keyword | Type |
|-------|-------|---------|------|
| `var x int` | anywhere | `var` | variables |
| `var x = 10` | anywhere | `var` | variables |
| `x := 10` | inside functions ONLY | none | variables |
| `const X = 10` | anywhere | `const` | constants |
| `_` | anywhere | none | discard |

### Zero Values — Every Type Has One
| Type | Zero Value |
|------|-----------|
| `int`, `float64` | `0` |
| `string` | `""` |
| `bool` | `false` |
| `pointer` | `nil` |
| `slice` | `nil` |
| `map` | `nil` |
| `interface` | `nil` |

### iota Behaviour — Interview Favourite
```go
const (
    A = iota     // 0
    B            // 1 — repeats last expression with iota incremented
    C = 10       // 10 — explicit override
    D            // 10 — repeats last expression (10), NOT iota
    E = iota     // 4 — iota was counting all along
)
```
> iota counts **line position in the const block**, not number of iota usages.

### Interview Topics
- Why does `:=` not work at package level? — It's syntactic sugar for `var` + assignment inside a function scope. Package-level declarations need `var`.
- What happens if you declare a variable and never use it? — **Compile error**.
- What is the zero value of a pointer? — `nil`. Dereferencing it causes runtime panic.
- Can constants be typed? — Yes: `const Pi float64 = 3.14`

---

## 3. Data Types & Type System

### Numeric Types
| Type | Size | Notes |
|------|------|-------|
| `int` | 32 or 64-bit (platform) | default integer |
| `int8/16/32/64` | explicit | use when size matters |
| `uint` | unsigned, platform | no negatives |
| `float32` | 32-bit | less precision |
| `float64` | 64-bit | **default float** |
| `byte` | alias for `uint8` | raw data, ASCII |
| `rune` | alias for `int32` | Unicode code point |
| `complex64/128` | complex numbers | rare |

### Strings — Key Facts
```go
s := "Gopher"

len(s)           // 6 — counts BYTES not characters
s[0]             // 71 — byte value of 'G' (uint8)
string(s[0])     // "G" — convert byte to string
s[0] = 'H'      // ❌ COMPILE ERROR — strings are immutable

// Multiline string
text := `line one
line two
line three`

// Range over string gives RUNES not bytes
for i, r := range "Go🔥" {
    fmt.Printf("%d: %c\n", i, r)
}
// 0: G
// 1: o
// 2: 🔥  ← index 2, but 🔥 takes 4 bytes
```

### Type Conversion — Always Explicit
```go
var x int = 42
var f float64 = float64(x)   // ✅
var y int = f                 // ❌ compile error

// String conversions
n := 65
string(n)                     // "A" — unicode code point, NOT "65"
fmt.Sprintf("%d", n)          // "65" — what you usually want
strconv.Itoa(n)               // "65" — int to string

// []byte and string
b := []byte("hello")          // string → byte slice
s := string(b)                // byte slice → string
```

### byte vs rune — Interview Essential
| | byte | rune |
|--|------|------|
| Underlying type | `uint8` | `int32` |
| Size | 8 bits | 32 bits |
| Represents | single ASCII byte | Unicode code point |
| Literal | `'A'` stored as 65 | `'🔥'` stored as 128293 |

### Behind the Scenes — Strings
- Go strings are **immutable byte slices** under the hood
- String header = pointer to byte array + length (2 words)
- `len(s)` returns number of **bytes**, not characters
- For correct character counting with unicode: `len([]rune(s))`

### Interview Topics
- Why does `string(65)` return `"A"` not `"65"`? — It converts the integer as a Unicode code point.
- What is the difference between `byte` and `rune`? — `byte = uint8` (ASCII), `rune = int32` (Unicode).
- Are strings mutable in Go? — No. `s[0] = 'x'` is a compile error.
- What does `len()` count on a string? — Bytes, not characters.

---

## 4. Control Flow

### if — All Forms
```go
// Basic
if x > 5 {
    fmt.Println("big")
} else if x == 5 {
    fmt.Println("five")
} else {
    fmt.Println("small")
}

// With init statement — very idiomatic
if val := getValue(); val > 0 {
    fmt.Println(val)
}
// val is NOT accessible here — scoped to if block
```

### for — The ONLY Loop in Go (6 Forms)
```go
// 1. Classic C-style
for i := 0; i < 5; i++ {
    fmt.Println(i)
}

// 2. While-style (condition only)
n := 10
for n > 0 {
    n--
}

// 3. Infinite loop
for {
    if done { break }
}

// 4. Range over slice — index and value
for i, v := range []int{10, 20, 30} {
    fmt.Println(i, v)
}

// 5. Range — discard index
for _, v := range slice {
    fmt.Println(v)
}

// 6. Range over string — gives (byte index, rune)
for i, r := range "Go🔥" {
    fmt.Printf("%d: %c\n", i, r)
}

// 7. Range over map
for k, v := range myMap {
    fmt.Println(k, v)
}

// 8. Range over channel — blocks until close
for v := range ch {
    fmt.Println(v)
}
```

### switch — All 4 Forms
```go
// 1. Classic — no break needed
switch x {
case 1:
    fmt.Println("one")
case 2, 3:              // multiple values in one case
    fmt.Println("two or three")
default:
    fmt.Println("other")
}

// 2. No condition — acts like if-else chain
switch {
case x > 10:
    fmt.Println("big")
case x > 0:
    fmt.Println("positive")
default:
    fmt.Println("non-positive")
}

// 3. With init statement
switch y := getValue(); {
case y > 0:
    fmt.Println("positive")
default:
    fmt.Println("non-positive")
}

// 4. Explicit fallthrough — opt-in only
switch x {
case 1:
    fmt.Println("one")
    fallthrough         // forces next case to execute
case 2:
    fmt.Println("two") // runs even if x == 1
}

// 5. Type switch
func describe(v any) {
    switch val := v.(type) {
    case int:
        fmt.Println("int:", val)
    case string:
        fmt.Println("string:", val)
    default:
        fmt.Println("unknown")
    }
}
```

### Go vs C/Java switch — Interview Essential
| Language | Default behaviour | Override with |
|---------|------------------|---------------|
| C, Java | Falls through automatically | `break` |
| Go | Breaks automatically | `fallthrough` |

### Interview Topics
- Go has no `while` keyword — how do you write a while loop? Use `for condition { }`.
- Can you `break` out of nested loops? Yes — use **labeled break**: `outer: for { for { break outer } }`.
- What does `range` return on a string? — `(byte index, rune value)` — index is byte position, not character position.
- Is `fallthrough` checked at compile time? — No, it's runtime. But it must be the last statement in a case.

---

## 5. Arrays, Slices, Maps

### Arrays — Fixed Size
```go
// Declarations
var arr [3]int                      // [0 0 0]
arr2 := [3]string{"Go", "Rust", "C"}
arr3 := [...]int{1, 2, 3, 4}       // compiler counts — size = 4

// Key fact: size is part of the type
// [3]int and [4]int are DIFFERENT TYPES — cannot assign or compare
```

### Slices — All Declaration Forms
```go
// 1. Slice literal
s := []int{1, 2, 3}

// 2. make(type, length, capacity)
s2 := make([]int, 3)      // len=3, cap=3
s3 := make([]int, 3, 10)  // len=3, cap=10

// 3. From array
arr := [5]int{1, 2, 3, 4, 5}
s4 := arr[1:4]             // [2 3 4] — shares memory with arr

// 4. Nil slice — zero value
var s5 []int               // nil, len=0, cap=0
s5 == nil                  // true
len(s5)                    // 0 — safe

// 5. Empty slice — NOT nil
s6 := []int{}              // not nil, len=0
s6 == nil                  // false

// 6. Append
s = append(s, 4)           // single value
s = append(s, 5, 6, 7)    // multiple values
s = append(s, other...)    // spread another slice
```

### Slice Operations
```go
s := []int{1, 2, 3, 4, 5}

s[1:3]   // [2 3]   — index 1 inclusive, 3 exclusive
s[:2]    // [1 2]   — from start
s[2:]    // [3 4 5] — to end
s[:]     // [1 2 3 4 5] — full copy reference

len(s)   // 5 — current elements
cap(s)   // capacity of underlying array

// Copy — independent, no shared memory
dst := make([]int, len(s))
copy(dst, s)
```

### Slice — Behind the Scenes
A slice is a **3-word struct**:
```
[pointer to array | length | capacity]
```
- `append` returns a **new slice header**. When capacity is exceeded, Go allocates a **new array** (typically 2x) and copies all elements.
- This is why you MUST write `s = append(s, val)` — the returned slice may point to a different array.
- Two slices sharing the same underlying array — modifying one modifies the other.

```go
a := []int{1, 2, 3, 4, 5}
b := a[1:3]     // b shares a's array
b[0] = 99       // modifies a[1] too!
fmt.Println(a)  // [1 99 3 4 5]

// Fix — use copy
b = make([]int, 2)
copy(b, a[1:3]) // now independent
```

### Maps — All Declaration Forms
```go
// 1. Map literal
m := map[string]int{
    "alice": 90,
    "bob":   85,
}

// 2. make
m2 := make(map[string]int)

// 3. Nil map — zero value — CANNOT WRITE TO IT
var m3 map[string]int   // nil
m3["key"] = 1           // ❌ PANIC: assignment to nil map

// Writing
m["charlie"] = 95

// Reading — always returns zero value if key missing, never panics
val := m["unknown"]     // 0 — no error

// Safe read — check existence
val, ok := m["alice"]
if ok {
    fmt.Println("found:", val)
}

// Delete
delete(m, "bob")

// Range — order NOT guaranteed
for k, v := range m {
    fmt.Println(k, v)
}
```

### Maps — Behind the Scenes
- Implemented as a **hash table**
- Zero value is `nil` — a nil map has no hash table allocated
- Reading from nil map is safe (returns zero value)
- Writing to nil map **panics** — no hash table to write to
- Map iteration order is **intentionally randomized** each run (by design, to prevent code depending on order)

### Interview Topics — Slices
- What happens when `append` exceeds capacity? — New array allocated (2x), all data copied, new slice returned.
- Why must you reassign `s = append(s, val)`? — append returns a new slice header; ignoring it loses the result.
- What is the difference between `nil` slice and empty slice `[]int{}`? — Both have len=0, but nil slice == nil; empty slice != nil. Both are safe to `append` to.
- How do you safely copy a slice? — `copy(dst, src)` — creates independent memory.

### Interview Topics — Maps
- What is the zero value of a map? — `nil`.
- Reading from nil map — safe, returns zero value. Writing to nil map — panics.
- Is map iteration order guaranteed? — No, intentionally random.
- Are maps thread-safe? — No. Concurrent read/write without sync causes a **fatal error** (not just a race condition).

---

## 6. Functions

### All Function Forms
```go
// 1. Basic
func add(a, b int) int {
    return a + b
}

// 2. Multiple return values — idiomatic Go
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("cannot divide by zero")
    }
    return a / b, nil
}

// 3. Named return values
func minMax(nums []int) (min, max int) {
    min, max = nums[0], nums[0]
    for _, v := range nums {
        if v < min { min = v }
        if v > max { max = v }
    }
    return   // naked return — returns named values
}

// 4. Variadic — variable number of args
func sum(nums ...int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}
sum(1, 2, 3)          // direct call
sum(slice...)         // spread a slice

// 5. Function as value
multiply := func(a, b int) int { return a * b }
fmt.Println(multiply(3, 4))

// 6. Function as parameter
func apply(a, b int, op func(int, int) int) int {
    return op(a, b)
}

// 7. Immediately invoked
result := func(x int) int { return x * 2 }(5)
```

### defer — Execution Order
```go
func main() {
    defer fmt.Println("one")    // declared first
    defer fmt.Println("two")
    defer fmt.Println("three")  // declared last
}
// Output:
// three  ← LIFO
// two
// one
```

### defer — Behind the Scenes
- `defer` pushes the function call onto a **defer stack** (LIFO)
- Arguments are **evaluated immediately** when defer is called, not when it runs
- Runs when the **surrounding function returns** — whether normally or via panic
- Primary use: cleanup (close files, unlock mutex, close DB connections)

```go
// defer evaluates arguments immediately
x := 10
defer fmt.Println(x)   // prints 10, not 20
x = 20
```

### Error Convention — Critical
```go
// error is ALWAYS the last return value
func doWork() (string, error) {
    if somethingWrong {
        return "", fmt.Errorf("describe what went wrong: %w", err) // %w wraps error
    }
    return "result", nil
}

// Caller ALWAYS checks err immediately
result, err := doWork()
if err != nil {
    // handle — never ignore
    return
}
// use result here
```

### Interview Topics
- What is a naked return? — Returning without values when named return variables are declared. Avoid in long functions — hurts readability.
- When do deferred functions run? — When the surrounding function returns (normal or panic).
- What order do multiple defers execute? — LIFO — last declared, first executed.
- Can you defer a method call? — Yes: `defer file.Close()`.

---

## 7. Pointers

### All Pointer Operations
```go
x := 42
p := &x          // & = "address of" → p is *int
fmt.Println(p)   // 0xc000014080 — memory address
fmt.Println(*p)  // 42 — dereference: value at address
*p = 100         // modify x through pointer
fmt.Println(x)   // 100

// new — allocates zeroed memory, returns pointer
p2 := new(int)   // *int pointing to zero value
*p2 = 99

// Nil pointer — zero value of any pointer
var p3 *int      // nil
fmt.Println(*p3) // ❌ PANIC: nil pointer dereference — runtime, not compile time

// Safe check
if p3 != nil {
    fmt.Println(*p3)
}
```

### Pass by Value vs Pointer
```go
// Go is ALWAYS pass by value
// Without pointer — function works on a copy
func double(n int) {
    n = n * 2    // modifies local copy only
}

// With pointer — pass address as value → indirect mutation
func double(n *int) {
    *n = *n * 2  // modifies value at address
}

x := 5
double(&x)
fmt.Println(x)   // 10
```

### Pointer Receivers with Structs
```go
type User struct { Name string; Age int }

u := User{Name: "Ada"}
p := &u
p.Name = "Grace"    // Go auto-dereferences — (*p).Name = "Grace"
fmt.Println(u.Name) // "Grace"
```

### Behind the Scenes — Pointers
- Go has **garbage collection** — no manual `free()`
- Pointers allow sharing memory without copying large structs
- Stack vs heap: small, short-lived variables → stack; escaped variables (pointers returned from functions) → heap
- **Escape analysis**: Go compiler decides automatically whether a variable goes on stack or heap

### Interview Topics
- What is the zero value of a pointer? — `nil`.
- Is nil pointer dereference a compile error or runtime error? — **Runtime panic**.
- What does `&` do? What does `*` do? — `&` takes address; `*` dereferences (gets value at address).
- Go is pass-by-value — so how do you mutate something in a function? — Pass a pointer to it.
- Difference between `new(T)` and `&T{}`? — Both return `*T`. `new` zero-initializes; `&T{}` allows field initialization.

---

## 8. Structs, Methods, Interfaces

### Struct Declarations
```go
// Basic struct
type Person struct {
    Name  string
    Age   int
    Email string
}

// Initialization forms
p1 := Person{Name: "Ada", Age: 30, Email: "ada@go.dev"}  // named — preferred
p2 := Person{"Bob", 25, "bob@go.dev"}                     // positional — fragile, avoid
var p3 Person                                               // zero value struct
p4 := &Person{Name: "Grace"}                              // pointer to struct

// Anonymous struct
point := struct{ X, Y int }{X: 1, Y: 2}

// Embedded struct
type Employee struct {
    Person              // embedded — inherits fields and methods
    Company string
}
e := Employee{Person: Person{Name: "Ada"}, Company: "Go Inc"}
e.Name    // promoted — accessible directly
```

### Methods — Value vs Pointer Receiver
```go
type Rectangle struct {
    Width, Height float64
}

// Value receiver — works on a COPY, cannot modify original
func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}

// Pointer receiver — works on ORIGINAL, can modify
func (r *Rectangle) Scale(factor float64) {
    r.Width *= factor
    r.Height *= factor
}

r := Rectangle{Width: 5, Height: 3}
r.Scale(2)              // Go auto-takes address: (&r).Scale(2)
fmt.Println(r.Area())   // 60
```

### Receiver Rules — Interview Essential
| Use value receiver when | Use pointer receiver when |
|------------------------|--------------------------|
| Not modifying struct | Modifying struct |
| Struct is small | Struct is large (avoid copy) |
| — | Consistency: if any method uses pointer receiver, all should |

### Interfaces
```go
// Define interface
type Shape interface {
    Area() float64
    Perimeter() float64
}

// Implicit implementation — no "implements" keyword
type Circle struct{ Radius float64 }

func (c Circle) Area() float64      { return 3.14159 * c.Radius * c.Radius }
func (c Circle) Perimeter() float64 { return 2 * 3.14159 * c.Radius }

// Circle now satisfies Shape automatically
var s Shape = Circle{Radius: 5}

// Empty interface — accepts any type
var anything any = 42
anything = "now a string"

// Type assertion — safe form
val, ok := anything.(string)
if ok {
    fmt.Println(val)
}

// Type switch
func describe(v any) {
    switch val := v.(type) {
    case int:    fmt.Println("int:", val)
    case string: fmt.Println("string:", val)
    default:     fmt.Println("unknown")
    }
}
```

### Behind the Scenes — Interfaces
An interface value is a **2-word struct**:
```
[type pointer | data pointer]
```
- When you assign a concrete type to an interface, Go stores a pointer to the type's method table (itab) and a pointer to the data.
- A nil interface `var s Shape` — both pointers are nil.
- A non-nil interface holding a nil pointer — the interface itself is NOT nil. Classic Go gotcha:

```go
var p *Circle = nil
var s Shape = p      // s is NOT nil — it has type info
s == nil             // false — interface has type, even though value is nil
```

### Interview Topics
- How does Go know if a type implements an interface? — If the type has all the methods (matching name, params, return types) — it implements it. No explicit declaration needed.
- Can you assign a concrete type to an interface? — Yes, if it implements all the interface methods.
- What is an empty interface `any`? — An interface with no methods — satisfied by every type.
- Can a value-receiver method satisfy an interface when using a pointer? — Yes. Can a pointer-receiver method satisfy an interface when using a value? — **No**, not always. This is the addressability rule.

---

## 9. Goroutines, Channels, Mutex, WaitGroups

### Why Concurrency in Go?
Go was designed for concurrent systems (servers, pipelines, microservices). Its concurrency model is built into the language — not a library add-on.

### Goroutines
```go
// Launch with go keyword
go someFunction()
go func() {
    fmt.Println("I am a goroutine")
}()

// ⚠️ If main() exits — ALL goroutines are KILLED immediately
// They don't finish — they die
```

### Goroutines — Behind the Scenes
- Goroutines are **NOT OS threads** — they are multiplexed onto OS threads by the Go scheduler (M:N model)
- Initial stack: **~2KB** (OS thread stack: ~1MB) — you can run millions of goroutines
- Go scheduler uses **cooperative + preemptive scheduling** (since Go 1.14)
- GOMAXPROCS controls how many OS threads run goroutines in parallel (default = number of CPUs)

### sync.WaitGroup — Why Required
```go
// Without WaitGroup — main exits before goroutines finish
import "sync"

var wg sync.WaitGroup

for i := 0; i < 5; i++ {
    wg.Add(1)             // increment counter BEFORE launching goroutine
    go func(n int) {
        defer wg.Done()   // FIRST LINE — decrement on exit, panic-safe
        fmt.Println(n)
    }(i)                  // pass i as argument — avoid closure trap
}

wg.Wait()                 // block until counter reaches 0
```

### Closure Trap — Interview Essential
```go
// ❌ WRONG — all goroutines share the same i
for i := 0; i < 3; i++ {
    go func() {
        fmt.Println(i)    // might print 3, 3, 3
    }()
}

// ✅ CORRECT — each goroutine gets its own copy
for i := 0; i < 3; i++ {
    go func(n int) {
        fmt.Println(n)    // prints 0, 1, 2 in some order
    }(i)
}
```

### Channels — All Forms
```go
// Unbuffered — sender blocks until receiver is ready (synchronized)
ch := make(chan int)
go func() { ch <- 42 }()   // send — blocks until someone receives
val := <-ch                 // receive — blocks until value arrives

// Buffered — sender only blocks when buffer is full
ch2 := make(chan string, 3)
ch2 <- "one"    // doesn't block — buffer has space
ch2 <- "two"
ch2 <- "three"
// ch2 <- "four" // would block — buffer full

// Directional channel types (used in function signatures)
func producer(ch chan<- int) { ch <- 1 }   // send-only
func consumer(ch <-chan int) { <-ch }       // receive-only

// Close channel — signal no more values
close(ch)

// Range over channel — blocks until closed
for v := range ch {
    fmt.Println(v)
}

// Check if channel is closed
val, ok := <-ch
if !ok {
    fmt.Println("channel closed")
}
```

### Channels — Behind the Scenes
```
Unbuffered channel:
Sender goroutine → [rendezvous point] → Receiver goroutine
Both must be present — synchronizes goroutines

Buffered channel:
Sender goroutine → [slot1|slot2|slot3] → Receiver goroutine
Sender only blocks when all slots full
Receiver only blocks when all slots empty
```

### select — Wait on Multiple Channels
```go
select {
case msg := <-ch1:
    fmt.Println("from ch1:", msg)
case msg := <-ch2:
    fmt.Println("from ch2:", msg)
case ch3 <- data:
    fmt.Println("sent to ch3")
default:
    fmt.Println("nothing ready")  // makes select non-blocking
}
```

### Mutex — Why Required
Without mutex, concurrent goroutines reading and writing shared data cause **race conditions** — one goroutine's write overwrites another's. Result is unpredictable, non-reproducible bugs.

```go
import "sync"

type Counter struct {
    mu    sync.Mutex
    value int
}

func (c *Counter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()   // ALWAYS defer — guarantees unlock even on panic
    c.value++
}

func (c *Counter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.value
}
```

### sync.RWMutex — For Read-Heavy Workloads
```go
var mu sync.RWMutex

// Multiple readers can hold lock simultaneously
mu.RLock()
defer mu.RUnlock()
// read data...

// Only one writer, no readers
mu.Lock()
defer mu.Unlock()
// write data...
```

### Race Condition — Behind the Scenes
```
Without mutex:
Goroutine 1: reads value = 5
Goroutine 2: reads value = 5
Goroutine 1: writes value = 6
Goroutine 2: writes value = 6  ← lost Goroutine 1's increment
Result: 6 instead of 7
```
Go has a built-in race detector: `go run -race main.go`

### Interview Topics — Goroutines
- What happens when main() exits with running goroutines? — They are **killed immediately**, not completed.
- Goroutines vs OS threads? — Goroutines are much lighter (~2KB), multiplexed onto threads by Go scheduler. Millions can run simultaneously.
- What is the closure trap? — All goroutines in a loop share the same loop variable. Fix by passing it as an argument.

### Interview Topics — Channels
- Unbuffered vs buffered channel? — Unbuffered: sender blocks until receiver ready (synchronous). Buffered: sender blocks only when buffer full (async up to capacity).
- What happens if you send to a closed channel? — **Panic**.
- What happens if you receive from a closed channel? — Returns zero value and `false` for the ok flag.
- Why use directional channels `chan<-` and `<-chan`? — Compile-time safety — prevents accidentally sending or receiving on the wrong end.

### Interview Topics — Mutex & WaitGroup
- Why do you need Mutex with goroutines? — Goroutines share memory. Without mutex, concurrent writes cause race conditions.
- Where should `defer wg.Done()` be placed? — **First line** in the goroutine function — panic safety.
- Where should `wg.Add(1)` be called? — **Before** launching the goroutine — calling it inside the goroutine risks a race with `wg.Wait()`.
- Mutex vs RWMutex? — Mutex: one goroutine at a time (read or write). RWMutex: multiple concurrent readers OR one writer.

---

## 10. File Operations

### All File Operation Patterns
```go
import (
    "bufio"
    "os"
)

// Write entire file at once (creates or overwrites)
err := os.WriteFile("file.txt", []byte("content"), 0644)

// Read entire file at once
data, err := os.ReadFile("file.txt")
fmt.Println(string(data))

// Create (= OpenFile with O_CREATE|O_TRUNC|O_WRONLY)
f, err := os.Create("output.txt")
defer f.Close()
f.WriteString("line\n")

// Open for reading only (= OpenFile with O_RDONLY)
f, err := os.Open("file.txt")
defer f.Close()

// OpenFile — full control
f, err := os.OpenFile("file.txt", os.O_APPEND|os.O_WRONLY, 0644)

// Explicit close (when you need to reopen same file)
f.Close()   // NOT defer — defer runs at function exit, too late

// Read line by line
scanner := bufio.NewScanner(f)
lineNum := 1
for scanner.Scan() {
    fmt.Printf("%d: %s\n", lineNum, scanner.Text())
    lineNum++
}

// Check if file exists
_, err = os.Stat("file.txt")
if os.IsNotExist(err) {
    fmt.Println("does not exist")
}
```

### File Flags
| Flag | Meaning |
|------|---------|
| `os.O_RDONLY` | Read only |
| `os.O_WRONLY` | Write only |
| `os.O_RDWR` | Read and write |
| `os.O_CREATE` | Create if not exists |
| `os.O_APPEND` | Append to end |
| `os.O_TRUNC` | Truncate on open |

### File Permissions (Unix)
`0644` → owner: read+write, group: read, others: read
`0755` → owner: read+write+execute, group+others: read+execute

### Why `defer f.Close()` — Behind the Scenes
- OS has a **limit on open file descriptors** (typically 1024 per process)
- Each open file consumes one descriptor
- Not closing = **resource leak** — eventually program crashes with "too many open files"
- `defer` guarantees close even when function returns early or panics

### Interview Topics
- `os.Open` vs `os.Create`? — `os.Open` = read-only, fails if not exists. `os.Create` = creates or truncates, write mode.
- Why use `bufio.Scanner` instead of reading whole file? — Memory efficiency — reads line by line without loading entire file.
- Why explicit `f.Close()` instead of `defer` when reopening the same file? — `defer` runs at function exit. If you need to read a file you just wrote, you must close and reopen before the function returns.
- What does `O_APPEND|O_WRONLY` do? — `O_WRONLY` gives write permission; `O_APPEND` positions writes at end of file.

---

## 11. HTTP Client

### All HTTP Call Patterns
```go
import (
    "bytes"
    "encoding/json"
    "io"
    "net/http"
)

// GET — simplest
resp, err := http.Get("https://api.example.com/users")
if err != nil {
    fmt.Println("Error:", err)
}
defer resp.Body.Close()   // ALWAYS — immediately after error check

body, err := io.ReadAll(resp.Body)
fmt.Println(string(body), resp.StatusCode)

// POST with JSON body
type Payload struct {
    Name string `json:"name"`
}
data, _ := json.Marshal(Payload{Name: "Ada"})
resp, err = http.Post(url, "application/json", bytes.NewBuffer(data))
defer resp.Body.Close()

// Custom request — headers, auth, method control
req, err := http.NewRequest("GET", url, nil)
req.Header.Set("Authorization", "Bearer token")
req.Header.Set("Accept", "application/json")

client := &http.Client{Timeout: 10 * time.Second}
resp, err = client.Do(req)
defer resp.Body.Close()

// Decode JSON response directly
type User struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
}
var user User
json.NewDecoder(resp.Body).Decode(&user)
```

### JSON Struct Tags
```go
type User struct {
    Name  string `json:"name"`            // maps to "name" key
    Email string `json:"email,omitempty"` // omit if empty string
    Age   int    `json:"-"`              // always ignore this field
    Phone string `json:"phone_number"`   // rename in JSON
}
```

### json.Marshal vs json.NewDecoder
| | `json.Marshal` | `json.NewDecoder().Decode()` |
|--|---------------|------------------------------|
| Direction | Go struct → JSON bytes | JSON → Go struct |
| Use when | Sending data out | Reading response body |
| Counterpart | Sending with `http.Post` | Receiving from `resp.Body` |
| Alternative | — | `json.Unmarshal([]byte, &v)` — needs full body in memory first |

### Why `defer resp.Body.Close()` — Behind the Scenes
`resp.Body` is an open network connection. Not closing it:
- Prevents connection from being reused (HTTP keep-alive)
- Leaks from the connection pool
- Under load: connection pool exhausted → all requests hang

### Interview Topics
- `http.Get` vs `http.NewRequest + client.Do`? — `http.Get` is a shortcut with no control. `http.NewRequest` lets you set headers, auth, timeouts, method.
- Why set `Content-Type: application/json` on POST? — Tells the server how to interpret the body.
- What does `json:"name"` do? — Maps the struct field to a specific JSON key name (case-sensitive matching).
- Why use `json.NewDecoder(resp.Body).Decode()` instead of reading body then `json.Unmarshal`? — Decoder streams directly from body, more memory efficient. `Unmarshal` loads entire body into memory first.

---

## 12. REST API Server

### All Server Patterns
```go
import (
    "encoding/json"
    "net/http"
)

// Handler function signature — mandatory
func handler(w http.ResponseWriter, r *http.Request) {
    // w = write response into this
    // r = everything about incoming request
}

// Register routes — BEFORE ListenAndServe
http.HandleFunc("/users", usersHandler)
http.HandleFunc("/users/search", searchHandler)

// Start server — BLOCKS FOREVER — must be last
err := http.ListenAndServe(":8080", nil)

// Helper — write JSON response
func writeJSON(w http.ResponseWriter, status int, data any) {
    w.Header().Set("Content-Type", "application/json")  // must be before WriteHeader
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}
```

### Complete Handler Pattern
```go
func usersHandler(w http.ResponseWriter, r *http.Request) {
    switch r.Method {
    case http.MethodGet:
        // return all users
        writeJSON(w, http.StatusOK, users)

    case http.MethodPost:
        var u User
        err := json.NewDecoder(r.Body).Decode(&u)
        if err != nil {
            http.Error(w, "bad request", http.StatusBadRequest)
            return   // ✅ MUST return — http.Error doesn't stop execution
        }
        // save user
        writeJSON(w, http.StatusCreated, u)

    default:
        writeJSON(w, http.StatusMethodNotAllowed, nil)
    }
}

// Query parameters
func searchHandler(w http.ResponseWriter, r *http.Request) {
    name := r.URL.Query().Get("name")   // /search?name=ada
    token := r.Header.Get("Authorization")
}
```

### HTTP Status Codes — Common Ones
| Code | Constant | Meaning |
|------|----------|---------|
| 200 | `http.StatusOK` | Success |
| 201 | `http.StatusCreated` | Resource created |
| 400 | `http.StatusBadRequest` | Client error |
| 401 | `http.StatusUnauthorized` | Not authenticated |
| 403 | `http.StatusForbidden` | Not authorized |
| 404 | `http.StatusNotFound` | Not found |
| 405 | `http.StatusMethodNotAllowed` | Wrong method |
| 500 | `http.StatusInternalServerError` | Server error |

### Why Order Matters — Behind the Scenes
HTTP response has a strict order:
```
1. Headers (Content-Type, etc.)
2. Status code (WriteHeader)
3. Body
```
Once `WriteHeader` is called, headers are **flushed and locked**. Setting headers after `WriteHeader` is silently ignored. This is why `Content-Type` must come before `WriteHeader`.

```go
// ❌ WRONG — Content-Type set after WriteHeader, ignored
w.WriteHeader(http.StatusOK)
w.Header().Set("Content-Type", "application/json")

// ✅ CORRECT
w.Header().Set("Content-Type", "application/json")
w.WriteHeader(http.StatusOK)
json.NewEncoder(w).Encode(data)
```

### Why `return` After `http.Error`
```go
http.Error(w, "bad request", http.StatusBadRequest)
// WITHOUT return:
json.NewEncoder(w).Encode(data)   // writes MORE to response — corrupted
// "superfluous response.WriteHeader call" warning in logs
```
`http.Error` writes to the response but does **not** stop function execution. `return` is mandatory.

### Interview Topics
- What are the two parameters of every handler? — `http.ResponseWriter` (write response) and `*http.Request` (read request).
- Why register routes before `ListenAndServe`? — `ListenAndServe` blocks forever. Code after it never runs.
- Why must headers be set before `WriteHeader`? — Once WriteHeader is called, headers are flushed to the network and cannot be modified.
- Why `return` after `http.Error`? — `http.Error` doesn't stop execution. Without `return`, handler continues and double-writes the response.
- How do you protect shared data in an HTTP server? — `sync.Mutex` — concurrent requests hit the same memory.
- How do you read query params? — `r.URL.Query().Get("key")`.
- How do you read request headers? — `r.Header.Get("Authorization")`.

---

## Master Cheat Sheet — Things to Memorize

### Golden Rules
```
1. package main + import = mandatory in every runnable file
2. Unused imports = compile error
3. Unused variables = compile error
4. := only inside functions
5. defer = LIFO, runs at function return
6. error always last return value, always check != nil
7. return after http.Error()
8. defer resp.Body.Close() immediately after http call
9. Register routes before ListenAndServe
10. s = append(s, val) — always reassign
11. Writing to nil map = panic
12. defer wg.Done() = first line in goroutine
13. wg.Add(1) before launching goroutine
14. main() exit kills all goroutines
15. Headers before WriteHeader in HTTP handlers
```

### Quick Reference — Zero Values
```
int, float  → 0
string      → ""
bool        → false
pointer     → nil
slice       → nil
map         → nil
interface   → nil
struct      → all fields zero
```

### Quick Reference — When to Use What
```
Value receiver   → reading struct, no modification
Pointer receiver → modifying struct, large struct
Mutex            → protecting shared memory from concurrent goroutines
WaitGroup        → waiting for goroutines to finish
Channel          → communicating between goroutines
Buffered channel → decouple sender/receiver timing
defer            → cleanup (file close, mutex unlock, connection close)
```
