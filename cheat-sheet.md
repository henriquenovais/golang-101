# Golang Cheat Sheet
### For developers coming from TypeScript

---

## Table of Contents

1. [Module and Package System](#1-module-and-package-system)
2. [Variables and Types](#2-variables-and-types)
3. [Functions](#3-functions)
4. [Control Flow](#4-control-flow)
5. [Structs](#5-structs)
6. [Interfaces](#6-interfaces)
7. [Errors](#7-errors)
8. [Pointers](#8-pointers)
9. [Collections](#9-collections)
10. [Goroutines and Concurrency](#10-goroutines-and-concurrency)
11. [Channels](#11-channels)
12. [File I/O](#12-file-io)
13. [Testing](#13-testing)
14. [Common Standard Library Packages](#14-common-standard-library-packages)
15. [CLI Patterns](#15-cli-patterns)
16. [Database with database/sql](#16-database-with-databasesql)
17. [Useful Toolchain Commands](#17-useful-toolchain-commands)
18. [Quick Reference: TypeScript to Go](#18-quick-reference:-typescript-to-go)

---

## 1. Module and Package System

```bash
# Create a new module (like npm init)
go mod init github.com/yourname/projectname

# Add a dependency (like npm install)
go get gopkg.in/yaml.v3

# Remove unused dependencies, tidy go.sum
go mod tidy
```

```go
// Every file declares its package at the top
package main        // executable — must have a main() function
package config      // library package — name matches the directory

// Importing packages
import "fmt"
import "os"

// Multiple imports (preferred style)
import (
    "fmt"
    "os"
    "strings"

    "gopkg.in/yaml.v3"                               // third-party
    "github.com/yourname/project/internal/config"    // internal package
)

// Blank import — imports for side effects only (e.g. registering a DB driver)
import _ "modernc.org/sqlite"

// Alias an import
import migrationfs "github.com/yourname/project/internal/fs"
```

> **Key rule:** only identifiers that start with a capital letter are exported. `Task` is public. `task` is private to the package. There are no `export` keywords.

---

## 2. Variables and Types

```go
// Explicit declaration
var name string = "Alice"
var age  int    = 30

// Short declaration (most common inside functions)
name := "Alice"
age  := 30

// Multiple assignment
x, y := 10, 20
a, b := b, a    // swap — no temp variable needed

// Zero values (Go always initialises variables)
var i int     // 0
var f float64 // 0.0
var s string  // ""
var b bool    // false
var p *int    // nil

// Constants
const MaxRetries = 3
const Pi = 3.14159

// Typed constant
const Timeout time.Duration = 30 * time.Second

// iota — auto-incrementing constants (like an enum)
type Direction int
const (
    North Direction = iota  // 0
    East                    // 1
    South                   // 2
    West                    // 3
)
```

### Primitive Types

```go
// Boolean
var ok bool = true

// Integers
var i   int    // platform size (64-bit on modern systems)
var i8  int8   // -128 to 127
var i32 int32
var i64 int64
var u   uint   // unsigned

// Floats
var f32 float32
var f64 float64  // use this by default

// String
var s string = "hello"
s2 := `raw string literal
spans multiple lines`

// Rune (a single Unicode code point — like a char)
var r rune = 'A'

// Byte (alias for uint8)
var b byte = 65
```

### Type Conversion

```go
// No implicit conversion in Go — always explicit
i := 42
f := float64(i)
u := uint(f)
s := strconv.Itoa(i)       // int to string
n, err := strconv.Atoi(s)  // string to int (can fail)
```

---

## 3. Functions

```go
// Basic function
func greet(name string) string {
    return "Hello, " + name
}

// Multiple return values — very common in Go
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

// Calling a multi-return function
result, err := divide(10, 2)
if err != nil {
    fmt.Println("error:", err)
}

// Ignoring a return value with blank identifier
result, _ := divide(10, 2)

// Named return values (use sparingly — can hurt readability)
func minMax(nums []int) (min, max int) {
    min, max = nums[0], nums[0]
    for _, n := range nums {
        if n < min { min = n }
        if n > max { max = n }
    }
    return // "naked return" — returns named values
}

// Variadic function (like ...rest in TypeScript)
func sum(nums ...int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}
sum(1, 2, 3)
nums := []int{1, 2, 3}
sum(nums...)   // spread a slice into variadic

// First-class functions
add := func(a, b int) int { return a + b }
result := add(2, 3)

// Function as parameter
func apply(nums []int, fn func(int) int) []int {
    result := make([]int, len(nums))
    for i, n := range nums {
        result[i] = fn(n)
    }
    return result
}

// Closure — captures variables from outer scope
func counter() func() int {
    count := 0
    return func() int {
        count++
        return count
    }
}
c := counter()
c() // 1
c() // 2

// defer — runs when the surrounding function returns (like finally)
// Defers are LIFO (last in, first out)
func readFile(path string) error {
    f, err := os.Open(path)
    if err != nil {
        return err
    }
    defer f.Close()  // guaranteed to run even if we return early
    // ... read the file
    return nil
}
```

---

## 4. Control Flow

```go
// if — no parentheses required
if x > 10 {
    fmt.Println("big")
} else if x > 5 {
    fmt.Println("medium")
} else {
    fmt.Println("small")
}

// if with initialiser — very common for error handling
if err := doSomething(); err != nil {
    return err
}

// for — the only loop in Go (replaces while too)
for i := 0; i < 10; i++ {
    fmt.Println(i)
}

// while-style loop
for x < 100 {
    x *= 2
}

// infinite loop
for {
    // break to exit
    break
}

// range over a slice
fruits := []string{"apple", "banana", "cherry"}
for i, fruit := range fruits {
    fmt.Println(i, fruit)
}

// range — index only
for i := range fruits {
    fmt.Println(i)
}

// range — value only (blank identifier for index)
for _, fruit := range fruits {
    fmt.Println(fruit)
}

// range over a map
ages := map[string]int{"Alice": 30, "Bob": 25}
for name, age := range ages {
    fmt.Println(name, age)
}

// range over a string (yields runes, not bytes)
for i, ch := range "hello" {
    fmt.Printf("%d: %c\n", i, ch)
}

// switch — no fallthrough by default (unlike C/JS)
switch day {
case "Monday", "Tuesday":
    fmt.Println("early week")
case "Friday":
    fmt.Println("end of week")
default:
    fmt.Println("midweek")
}

// switch with no expression — like if/else if chain
switch {
case x < 0:
    fmt.Println("negative")
case x == 0:
    fmt.Println("zero")
default:
    fmt.Println("positive")
}

// switch on type — used with interfaces
func describe(i interface{}) {
    switch v := i.(type) {
    case int:
        fmt.Printf("int: %d\n", v)
    case string:
        fmt.Printf("string: %s\n", v)
    default:
        fmt.Printf("unknown: %T\n", v)
    }
}
```

---

## 5. Structs

```go
// Define a struct (no classes in Go)
type Task struct {
    Name    string
    Command string
    Deps    []string
}

// Create an instance — named fields (preferred)
t := Task{
    Name:    "test",
    Command: "go test ./...",
    Deps:    []string{},
}

// Create with positional fields (fragile — avoid)
t := Task{"test", "go test ./...", []string{}}

// Zero-value struct
var t Task   // all fields are zero values

// Access fields
fmt.Println(t.Name)
t.Command = "go test -v ./..."

// Struct with tags (used by JSON, YAML, DB libraries)
type User struct {
    ID    int    `json:"id"    db:"id"`
    Name  string `json:"name"  db:"name"`
    Email string `json:"email" db:"email"`
}

// Anonymous struct — useful for one-off shapes (e.g. test cases)
person := struct {
    Name string
    Age  int
}{
    Name: "Alice",
    Age:  30,
}

// Methods on structs — value receiver (does not modify original)
func (t Task) IsCompound() bool {
    return len(t.Deps) > 0
}

// Methods on structs — pointer receiver (can modify original, more common)
func (t *Task) AddDep(dep string) {
    t.Deps = append(t.Deps, dep)
}

// Calling methods
t.IsCompound()
t.AddDep("lint")   // Go auto-takes the address when needed

// Embedding — Go's form of composition (not inheritance)
type Animal struct {
    Name string
}
func (a Animal) Speak() string { return a.Name + " speaks" }

type Dog struct {
    Animal        // embedded — Dog gets Animal's fields and methods
    Breed string
}

d := Dog{Animal: Animal{Name: "Rex"}, Breed: "Labrador"}
d.Speak()   // promoted method — works directly on Dog
d.Name      // promoted field
```

---

## 6. Interfaces

```go
// Define an interface
type Executor interface {
    Run(command string, stdout io.Writer, stderr io.Writer) error
}

// Any type that has a matching Run method satisfies this interface.
// No "implements" keyword needed.
type ShellExecutor struct{}

func (e *ShellExecutor) Run(command string, stdout io.Writer, stderr io.Writer) error {
    cmd := exec.Command("sh", "-c", command)
    cmd.Stdout = stdout
    cmd.Stderr = stderr
    return cmd.Run()
}

// Using the interface as a parameter type
func runTask(e Executor, command string) error {
    return e.Run(command, os.Stdout, os.Stderr)
}

// Pass any satisfying type
runTask(&ShellExecutor{}, "go test ./...")

// The empty interface — accepts any value (like TypeScript's unknown)
var anything interface{} = 42
anything = "now a string"

// In modern Go (1.18+), use any as an alias for interface{}
var val any = true

// Type assertion — extract the concrete value
var i interface{} = "hello"
s, ok := i.(string)   // safe assertion — ok is false if wrong type
if ok {
    fmt.Println(s)
}
s := i.(string)        // unsafe — panics if wrong type

// Composing interfaces
type Reader interface {
    Read(p []byte) (n int, err error)
}
type Writer interface {
    Write(p []byte) (n int, err error)
}
type ReadWriter interface {
    Reader    // embed both
    Writer
}

// Checking if a value implements an interface at compile time
// This line causes a compile error if ShellExecutor does not satisfy Executor
var _ Executor = (*ShellExecutor)(nil)
```

---

## 7. Errors

```go
// The error type is a built-in interface:
// type error interface { Error() string }

// Return nil for success, error for failure
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

// Always check errors immediately
result, err := divide(10, 0)
if err != nil {
    fmt.Println("failed:", err)
    return
}

// Wrapping errors with context — use %w to allow unwrapping
func loadConfig(path string) error {
    data, err := os.ReadFile(path)
    if err != nil {
        return fmt.Errorf("loadConfig: reading file %s: %w", path, err)
    }
    _ = data
    return nil
}

// Unwrapping — check if an error is or wraps a specific error
if errors.Is(err, os.ErrNotExist) {
    fmt.Println("file not found")
}

// Unwrapping into a concrete type
var pathErr *os.PathError
if errors.As(err, &pathErr) {
    fmt.Println("path:", pathErr.Path)
}

// Custom error type
type ValidationError struct {
    Field   string
    Message string
}
func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation error on %s: %s", e.Field, e.Message)
}

// Sentinel errors — predefined error values to compare against
var ErrNotFound = errors.New("not found")
var ErrTimeout  = errors.New("timeout")

func findTask(name string) error {
    return fmt.Errorf("findTask: %w", ErrNotFound)
}

// Check against sentinel
if errors.Is(err, ErrNotFound) { ... }

// panic and recover — use only for truly unrecoverable situations
// NOT for regular control flow (unlike exceptions in TypeScript)
func mustPositive(n int) int {
    if n <= 0 {
        panic("n must be positive")
    }
    return n
}

// recover — catch a panic (usually in library code, not application code)
func safeDiv(a, b int) (result int, err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("recovered from panic: %v", r)
        }
    }()
    return a / b, nil
}
```

---

## 8. Pointers

```go
// A pointer holds the memory address of a value
// Go has pointers but no pointer arithmetic

var x int = 42
p := &x         // & takes the address of x — p is *int
fmt.Println(*p) // * dereferences — prints 42
*p = 100        // modifies x through the pointer
fmt.Println(x)  // 100

// new() — allocates zero-value and returns a pointer
p := new(int)   // *int pointing to 0
*p = 5

// Why use pointer receivers on methods?
// 1. To modify the struct's fields
// 2. To avoid copying a large struct on every call

type Config struct{ Debug bool }

func (c *Config) Enable() { c.Debug = true }   // modifies the original
func (c Config)  IsOn() bool { return c.Debug } // reads only — value receiver is fine

// Nil pointer — the zero value of any pointer type
var p *int      // nil
if p != nil {
    fmt.Println(*p)
}

// Pointer to struct — common pattern
type Task struct{ Name string }
t := &Task{Name: "build"}  // t is *Task
t.Name = "test"            // Go auto-dereferences — no need for (*t).Name

// When to use pointers vs values:
// Use pointer receiver if the method modifies state
// Use pointer receiver if the struct is large
// Use value receiver for small read-only structs
// Be consistent — if any method uses a pointer receiver, use it for all
```

---

## 9. Collections

### Slices

```go
// Slice — a dynamic, resizable view over an array (use this, not arrays)
var s []int                    // nil slice (length 0)
s  := []int{1, 2, 3}          // slice literal
s  := make([]int, 3)          // [0, 0, 0] — length 3, capacity 3
s  := make([]int, 0, 10)      // empty slice, pre-allocated capacity of 10

// Length and capacity
len(s)   // number of elements
cap(s)   // underlying array capacity

// Append — returns a new slice (may allocate new underlying array)
s = append(s, 4)
s = append(s, 5, 6, 7)
s = append(s, other...)   // append another slice

// Slicing — [low:high] (low inclusive, high exclusive)
s := []int{0, 1, 2, 3, 4}
s[1:3]   // [1, 2]
s[:3]    // [0, 1, 2]
s[2:]    // [2, 3, 4]
s[:]     // full copy reference

// Copy into a new slice (independent from original)
dst := make([]int, len(src))
copy(dst, src)

// Check if empty
if len(s) == 0 { ... }

// Iterate
for i, v := range s {
    fmt.Println(i, v)
}

// 2D slice
matrix := [][]int{
    {1, 2, 3},
    {4, 5, 6},
}
matrix[1][2]  // 6

// Delete element at index i (order-preserving)
s = append(s[:i], s[i+1:]...)

// Filter (no built-in filter — write it yourself)
filtered := s[:0]
for _, v := range s {
    if v > 2 {
        filtered = append(filtered, v)
    }
}
```

### Maps

```go
// Map — key/value store (like an object or Map in TypeScript)
var m map[string]int            // nil map — reading is safe, writing panics
m  := map[string]int{}          // empty map — safe to read and write
m  := make(map[string]int)      // same as above
m  := map[string]int{           // map literal
    "alice": 30,
    "bob":   25,
}

// Get
age := m["alice"]      // returns 0 if key is missing — no error
age, ok := m["alice"]  // ok is false if key is missing — use this form
if ok {
    fmt.Println(age)
}

// Set
m["charlie"] = 35

// Delete
delete(m, "bob")

// Check existence
if _, ok := m["alice"]; ok {
    fmt.Println("found")
}

// Iterate (order is randomised — never rely on map order)
for key, value := range m {
    fmt.Println(key, value)
}

// Map of slices — common pattern
graph := map[string][]string{
    "ci":   {"lint", "test", "build"},
    "lint": {},
}
graph["ci"] = append(graph["ci"], "format")

// Use struct as map value
type User struct{ Name string; Age int }
users := map[string]User{
    "u1": {Name: "Alice", Age: 30},
}
```

### Arrays

```go
// Arrays are fixed-size — rarely used directly; prefer slices
var a [3]int           // [0, 0, 0]
a := [3]int{1, 2, 3}
a := [...]int{1, 2, 3} // compiler infers size

// Arrays are value types — assignment copies the entire array
b := a    // b is a full copy, not a reference
```

---

## 10. Goroutines and Concurrency

```go
// Goroutine — lightweight thread managed by the Go runtime
// Start one with the go keyword
go func() {
    fmt.Println("running in background")
}()

// Pass a value explicitly to avoid closure bugs
for _, task := range tasks {
    go func(t Task) {
        t.Run()
    }(task)    // task is passed by value — safe
}

// sync.WaitGroup — wait for a group of goroutines to finish (like Promise.all)
var wg sync.WaitGroup

for _, task := range tasks {
    wg.Add(1)                // increment counter before starting goroutine
    go func(t Task) {
        defer wg.Done()      // decrement counter when goroutine exits
        t.Run()
    }(task)
}

wg.Wait()    // blocks until counter reaches 0

// sync.Mutex — protect shared data from concurrent writes
type SafeCounter struct {
    mu    sync.Mutex
    count int
}

func (c *SafeCounter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}

func (c *SafeCounter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.count
}

// sync.RWMutex — multiple concurrent readers, exclusive writer
type Cache struct {
    mu   sync.RWMutex
    data map[string]string
}

func (c *Cache) Get(key string) string {
    c.mu.RLock()         // multiple goroutines can hold RLock simultaneously
    defer c.mu.RUnlock()
    return c.data[key]
}

func (c *Cache) Set(key, value string) {
    c.mu.Lock()          // exclusive — no readers or writers allowed
    defer c.mu.Unlock()
    c.data[key] = value
}

// sync.Once — run something exactly once (useful for lazy initialisation)
var once sync.Once
once.Do(func() {
    fmt.Println("runs exactly one time")
})
```

---

## 11. Channels

```go
// Channel — typed conduit for communication between goroutines
ch := make(chan int)         // unbuffered — sender blocks until receiver is ready
ch := make(chan int, 10)     // buffered — sender blocks only when buffer is full

// Send and receive
ch <- 42         // send (blocks if unbuffered and no receiver)
val := <-ch      // receive (blocks until a value is sent)

// Close a channel — signals no more values will be sent
close(ch)

// Receive from a closed channel returns zero value immediately
val, ok := <-ch   // ok is false when channel is closed and empty

// Range over a channel — exits when channel is closed
for val := range ch {
    fmt.Println(val)
}

// Common pattern: fan-out errors from goroutines
func runAll(tasks []Task) error {
    errs := make(chan error, len(tasks))    // buffered — goroutines never block
    var wg sync.WaitGroup

    for _, t := range tasks {
        wg.Add(1)
        go func(task Task) {
            defer wg.Done()
            if err := task.Run(); err != nil {
                errs <- err
            }
        }(t)
    }

    wg.Wait()
    close(errs)

    for err := range errs {
        if err != nil {
            return err    // return first error encountered
        }
    }
    return nil
}

// select — wait on multiple channel operations (like switch for channels)
select {
case msg := <-ch1:
    fmt.Println("from ch1:", msg)
case msg := <-ch2:
    fmt.Println("from ch2:", msg)
case <-time.After(1 * time.Second):
    fmt.Println("timed out")
default:
    fmt.Println("no channel ready")  // non-blocking
}

// Done channel pattern — signal cancellation
done := make(chan struct{})

go func() {
    for {
        select {
        case <-done:
            fmt.Println("shutting down")
            return
        default:
            doWork()
        }
    }
}()

close(done)   // broadcast: all goroutines listening on done will unblock

// context — the standard way to handle cancellation and timeouts
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

ctx, cancel := context.WithCancel(context.Background())
defer cancel()

// Pass context as the first argument to functions that do I/O
func fetchData(ctx context.Context, url string) error {
    req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
    _, err := http.DefaultClient.Do(req)
    return err
}
```

---

## 12. File I/O

```go
// Read entire file into memory
data, err := os.ReadFile("config.yaml")
if err != nil {
    return fmt.Errorf("reading file: %w", err)
}
content := string(data)

// Write entire file
err := os.WriteFile("output.txt", []byte("hello"), 0644)

// File permissions (Unix octal):
// 0644 = owner can read/write, group and others can read
// 0755 = owner can read/write/execute, group and others can read/execute

// Open a file for reading
f, err := os.Open("input.txt")    // read-only
if err != nil { return err }
defer f.Close()

// Open with flags (create, write, append, etc.)
f, err := os.OpenFile("log.txt", os.O_CREATE|os.O_APPEND|os.O_WRONLY, 0644)
if err != nil { return err }
defer f.Close()

// Buffered reader — efficient for line-by-line reading
scanner := bufio.NewScanner(f)
for scanner.Scan() {
    line := scanner.Text()
    fmt.Println(line)
}
if err := scanner.Err(); err != nil {
    return err
}

// Buffered writer
writer := bufio.NewWriter(f)
fmt.Fprintln(writer, "line 1")
fmt.Fprintln(writer, "line 2")
writer.Flush()    // don't forget to flush!

// Read directory entries
entries, err := os.ReadDir("migrations")
if err != nil { return err }
for _, entry := range entries {
    if entry.IsDir() { continue }
    fmt.Println(entry.Name())
}

// Path operations
import "path/filepath"

filepath.Join("internal", "config", "config.go")  // internal/config/config.go
filepath.Base("/path/to/file.go")                  // file.go
filepath.Dir("/path/to/file.go")                   // /path/to
filepath.Ext("file.go")                            // .go
filepath.Abs("relative/path")                      // absolute path, error

// Check if file exists
_, err := os.Stat("file.txt")
if os.IsNotExist(err) {
    fmt.Println("file does not exist")
}

// Create directory (and all parents)
err := os.MkdirAll("a/b/c", 0755)

// Temp file — useful in tests
f, err := os.CreateTemp("", "prefix-*.yaml")
defer os.Remove(f.Name())   // clean up

// Temp directory
dir, err := os.MkdirTemp("", "testdir-*")
defer os.RemoveAll(dir)
```

---

## 13. Testing

```go
// File name must end in _test.go
// Package can be package foo (white-box) or package foo_test (black-box)

package config_test

import "testing"

// Basic test — function name must start with Test
func TestAdd(t *testing.T) {
    got  := add(2, 3)
    want := 5
    if got != want {
        t.Errorf("add(2, 3) = %d, want %d", got, want)
    }
}

// t.Fatal — stops the test immediately (like t.Error + return)
func TestLoad(t *testing.T) {
    cfg, err := Load("testdata/valid.yaml")
    if err != nil {
        t.Fatalf("unexpected error: %v", err)   // stops here if err != nil
    }
    if cfg == nil {
        t.Error("expected non-nil config")
    }
}

// Table-driven tests — idiomatic Go (like describe/it in Jest)
func TestDivide(t *testing.T) {
    tests := []struct {
        name    string
        a, b    float64
        want    float64
        wantErr bool
    }{
        {name: "normal division",  a: 10, b: 2,  want: 5},
        {name: "divide by zero",   a: 10, b: 0,  wantErr: true},
        {name: "negative divisor", a: 10, b: -2, want: -5},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := divide(tt.a, tt.b)

            if tt.wantErr && err == nil {
                t.Error("expected error, got nil")
            }
            if !tt.wantErr && err != nil {
                t.Errorf("unexpected error: %v", err)
            }
            if got != tt.want {
                t.Errorf("got %f, want %f", got, tt.want)
            }
        })
    }
}

// t.Helper — marks the function as a test helper
// Error lines point to the caller, not inside the helper
func assertEqual(t *testing.T, got, want int) {
    t.Helper()
    if got != want {
        t.Errorf("got %d, want %d", got, want)
    }
}

// Subtests and setup/teardown
func TestSuite(t *testing.T) {
    // Setup
    db := setupTestDB(t)

    t.Run("Insert", func(t *testing.T) { ... })
    t.Run("Select", func(t *testing.T) { ... })
    t.Run("Delete", func(t *testing.T) { ... })

    // Teardown (or use t.Cleanup)
    db.Close()
}

// t.Cleanup — registers a function to run when the test ends
func TestWithCleanup(t *testing.T) {
    dir := t.TempDir()                // auto-cleaned up after test
    f, _ := os.CreateTemp(dir, "*.yaml")
    t.Cleanup(func() { os.Remove(f.Name()) })
}

// testdata/ — convention for test fixture files
// Go will not compile files inside testdata/
// Access with relative path: "testdata/input.yaml"

// Benchmark test
func BenchmarkAdd(b *testing.B) {
    for i := 0; i < b.N; i++ {
        add(2, 3)
    }
}

// Run only benchmarks
// go test -bench=. ./...
// go test -bench=BenchmarkAdd -benchmem ./...
```

```bash
# Run all tests
go test ./...

# Verbose — shows each test name and PASS/FAIL
go test -v ./...

# Run a specific test
go test -v -run TestDivide ./internal/...

# Run with race detector (always use this)
go test -race ./...

# Coverage report
go test -cover ./...
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out   # open in browser

# Short mode — skip slow tests (use t.Skip in slow tests)
go test -short ./...
```

---

## 14. Common Standard Library Packages

### fmt

```go
// Print
fmt.Println("hello", "world")       // adds spaces, newline
fmt.Printf("name: %s, age: %d\n", name, age)
fmt.Fprintf(os.Stderr, "error: %v\n", err)   // write to any io.Writer

// Format verbs
%v    // default format
%+v   // struct with field names
%#v   // Go syntax representation
%T    // type of the value
%d    // integer
%f    // float (%.2f = 2 decimal places)
%s    // string
%q    // quoted string
%b    // binary
%x    // hexadecimal
%p    // pointer
%w    // wrap error (only in fmt.Errorf)

// Sprintf — format to a string
s := fmt.Sprintf("user %s is %d years old", name, age)

// Sscanf — parse a string
var name string
var age int
fmt.Sscanf("Alice 30", "%s %d", &name, &age)
```

### strings

```go
import "strings"

strings.Contains("hello world", "world")    // true
strings.HasPrefix("hello", "he")            // true
strings.HasSuffix("file.go", ".go")         // true
strings.Index("hello", "ll")               // 2 (-1 if not found)
strings.Count("hello", "l")               // 2
strings.Replace("foo bar foo", "foo", "baz", 1)  // "baz bar foo"
strings.ReplaceAll("foo bar foo", "foo", "baz")   // "baz bar baz"
strings.ToLower("HELLO")                   // "hello"
strings.ToUpper("hello")                   // "HELLO"
strings.TrimSpace("  hello  ")             // "hello"
strings.Trim("--hello--", "-")             // "hello"
strings.TrimPrefix("001_name.up.sql", "001_")
strings.TrimSuffix("file.go", ".go")
strings.Split("a,b,c", ",")               // ["a", "b", "c"]
strings.Join([]string{"a", "b"}, ", ")    // "a, b"
strings.Fields("  foo   bar  ")           // ["foo", "bar"] — split on whitespace
strings.Builder  // efficient string concatenation
```

### strconv

```go
import "strconv"

strconv.Itoa(42)              // int to string: "42"
strconv.Atoi("42")            // string to int: 42, nil
strconv.ParseFloat("3.14", 64)
strconv.ParseBool("true")
strconv.FormatFloat(3.14159, 'f', 2, 64)  // "3.14"
strconv.FormatInt(255, 16)    // "ff" (base 16)
```

### os

```go
import "os"

os.Args             // []string — command line arguments (Args[0] is the binary)
os.Exit(1)          // exit with code
os.Getenv("HOME")   // read environment variable
os.Setenv("KEY", "value")
os.Stderr           // io.Writer
os.Stdout           // io.Writer
os.Stdin            // io.Reader
os.ReadFile(path)
os.WriteFile(path, data, perm)
os.Open(path)       // read-only
os.Create(path)     // create or truncate
os.Remove(path)
os.RemoveAll(path)
os.MkdirAll(path, perm)
os.Stat(path)       // file info
os.IsNotExist(err)  // check if error is "not found"
```

### time

```go
import "time"

time.Now()
time.Now().Format("2006-01-02 15:04:05")   // reference time is Mon Jan 2 15:04:05 MST 2006
time.Now().Format("20060102150405")         // compact timestamp
time.Now().Unix()    // Unix timestamp (seconds)
time.Now().UnixMilli()

time.Sleep(2 * time.Second)
time.Since(start)    // duration elapsed

d := 5 * time.Second
d := 100 * time.Millisecond

time.After(1 * time.Second)   // returns a channel that receives after duration

t, err := time.Parse("2006-01-02", "2024-03-15")

// Note: Go's time format uses this specific reference time:
// Mon Jan 2 15:04:05 MST 2006
// (not Y-m-d H:i:s like most languages)
```

### io

```go
import "io"

io.Writer     // interface: Write(p []byte) (n int, err error)
io.Reader     // interface: Read(p []byte) (n int, err error)
io.ReadAll(r) // read everything from a Reader into []byte

// Discard — like /dev/null, useful in tests
io.Discard

// Multi-writer — write to multiple writers at once
w := io.MultiWriter(os.Stdout, logFile)
fmt.Fprintln(w, "written to both")

// Copy — stream from Reader to Writer
n, err := io.Copy(dst, src)
```

### encoding/json

```go
import "encoding/json"

type User struct {
    Name  string `json:"name"`
    Email string `json:"email"`
    Age   int    `json:"age,omitempty"`   // omit if zero
    pass  string                           // unexported — never marshalled
}

// Marshal — struct to JSON bytes
data, err := json.Marshal(user)
data, err := json.MarshalIndent(user, "", "  ")   // pretty-print

// Unmarshal — JSON bytes to struct
var user User
err := json.Unmarshal(data, &user)

// Encode/Decode — stream-based (preferred for HTTP)
encoder := json.NewEncoder(w)
encoder.Encode(user)

decoder := json.NewDecoder(r)
decoder.Decode(&user)
```

### sync

```go
import "sync"

sync.WaitGroup   // wait for goroutines — see section 10
sync.Mutex       // exclusive lock — see section 10
sync.RWMutex     // read/write lock — see section 10
sync.Once        // run exactly once — see section 10
sync.Map         // concurrent-safe map (use sparingly — usually a Mutex + map is clearer)
```

---

## 15. CLI Patterns

```go
// Reading arguments
// os.Args[0] = binary name
// os.Args[1] = first argument
if len(os.Args) < 2 {
    fmt.Fprintln(os.Stderr, "Usage: mytool <command>")
    os.Exit(1)
}
command := os.Args[1]

// Subcommand dispatch
switch command {
case "run":
    if len(os.Args) < 3 {
        fmt.Fprintln(os.Stderr, "Usage: mytool run <name>")
        os.Exit(1)
    }
    runTask(os.Args[2])
case "list":
    listTasks()
default:
    fmt.Fprintf(os.Stderr, "unknown command: %s\n", command)
    os.Exit(1)
}

// flag package — for named flags like --verbose or --config=path
import "flag"

verbose := flag.Bool("verbose", false, "enable verbose output")
config  := flag.String("config", "tasks.yaml", "path to config file")
port    := flag.Int("port", 8080, "port to listen on")

flag.Parse()   // must call this before using flag values

fmt.Println(*verbose)   // dereference — flag returns pointers
fmt.Println(*config)
fmt.Println(*port)

// Remaining args after flags
args := flag.Args()   // []string

// Exit codes
os.Exit(0)   // success
os.Exit(1)   // general error (use this by default)
os.Exit(2)   // misuse of command (wrong arguments)

// Write to stderr for errors, stdout for output
fmt.Fprintln(os.Stderr, "error: something went wrong")
fmt.Fprintln(os.Stdout, "result: done")

// Graceful shutdown — listen for OS signals
sigChan := make(chan os.Signal, 1)
signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)

go func() {
    sig := <-sigChan
    fmt.Printf("received signal %s, shutting down\n", sig)
    // cleanup...
    os.Exit(0)
}()
```

---

## 16. Database with database/sql

```go
import (
    "database/sql"
    _ "modernc.org/sqlite"   // blank import registers the driver
)

// Open a connection (does not actually connect yet)
db, err := sql.Open("sqlite", "myapp.db")
if err != nil { return err }
defer db.Close()

// In-memory database — great for tests
db, err := sql.Open("sqlite", ":memory:")

// Verify connection
if err := db.Ping(); err != nil { return err }

// Execute — for INSERT, UPDATE, DELETE, CREATE TABLE
result, err := db.Exec(`
    CREATE TABLE IF NOT EXISTS users (
        id   INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        age  INTEGER
    )
`)

result, err := db.Exec(
    "INSERT INTO users (name, age) VALUES (?, ?)",
    "Alice", 30,
)
id, _ := result.LastInsertId()
rows, _ := result.RowsAffected()

// Query — for SELECT returning multiple rows
rows, err := db.Query("SELECT id, name, age FROM users WHERE age > ?", 18)
if err != nil { return err }
defer rows.Close()   // always close rows

for rows.Next() {
    var id   int
    var name string
    var age  int
    if err := rows.Scan(&id, &name, &age); err != nil {
        return err
    }
    fmt.Println(id, name, age)
}
if err := rows.Err(); err != nil { return err }   // check for iteration errors

// QueryRow — for SELECT returning a single row
var name string
err := db.QueryRow("SELECT name FROM users WHERE id = ?", 1).Scan(&name)
if err == sql.ErrNoRows {
    fmt.Println("not found")
} else if err != nil {
    return err
}

// Transactions
tx, err := db.Begin()
if err != nil { return err }
defer tx.Rollback()   // no-op if Commit has been called

_, err = tx.Exec("INSERT INTO users (name) VALUES (?)", "Bob")
if err != nil { return err }

if err := tx.Commit(); err != nil { return err }

// Prepared statements — more efficient for repeated queries
stmt, err := db.Prepare("INSERT INTO users (name, age) VALUES (?, ?)")
if err != nil { return err }
defer stmt.Close()

stmt.Exec("Alice", 30)
stmt.Exec("Bob", 25)
```

---

## 17. Useful Toolchain Commands

```bash
# Build
go build ./...                    # build all packages (check for errors)
go build -o bin/myapp ./cmd/app   # build specific binary to path

# Run without building a file
go run ./cmd/app/main.go

# Test
go test ./...                     # all tests
go test -v ./...                  # verbose
go test -run TestName ./...       # specific test
go test -race ./...               # race detector
go test -cover ./...              # coverage summary
go test -coverprofile=c.out ./... # coverage to file
go tool cover -html=c.out         # open coverage in browser
go test -bench=. ./...            # run benchmarks
go test -short ./...              # skip slow tests

# Format — always run this; CI will reject unformatted code
go fmt ./...

# Vet — static analysis, catches real bugs
go vet ./...

# Dependencies
go get package@version            # add or update dependency
go get package@latest             # update to latest
go mod tidy                       # remove unused, add missing
go mod download                   # download all deps to cache
go list -m all                    # list all dependencies

# Documentation
go doc fmt                        # package-level docs
go doc fmt.Println                # specific function
go doc -all strings               # all exported symbols

# Install a binary tool globally
go install github.com/some/tool@latest
# Binary lands in $GOPATH/bin (~/.go/bin) — make sure it is in your PATH

# Upgrade a specific dependency
go get github.com/some/package@v2.0.0

# Show module graph
go mod graph

# Check for vulnerabilities
go install golang.org/x/vuln/cmd/govulncheck@latest
govulncheck ./...

# golangci-lint — the standard multi-linter
# Install: https://golangci-lint.run/usage/install/
golangci-lint run ./...
```

---

## 18. Quick Reference: TypeScript to Go

| Concept | TypeScript | Go |
|---|---|---|
| Variable | `const x = 5` | `x := 5` |
| Typed variable | `const x: number = 5` | `var x int = 5` |
| Function | `function f(a: string): number` | `func f(a string) int` |
| Arrow function | `const f = (a: string) => a.length` | `f := func(a string) int { return len(a) }` |
| Optional return | `string \| undefined` | `(string, error)` — use the second value |
| Array | `number[]` | `[]int` |
| Object | `{ name: string }` | `struct { Name string }` |
| Dictionary | `Record<string, number>` | `map[string]int` |
| Null check | `if (x !== null)` | `if x != nil` |
| Async function | `async function f()` | Regular function + goroutine |
| Await | `await f()` | Call `f()` and handle returned error |
| Try/catch | `try { } catch (e) { }` | `result, err := f(); if err != nil { }` |
| Interface | `interface I { m(): void }` | `type I interface { M() }` |
| Implements | `class A implements I` | Implicit — just define the matching method |
| Null/undefined | `null`, `undefined` | `nil` |
| Any type | `any` / `unknown` | `any` / `interface{}` |
| Spread | `[...a, ...b]` | `append(a, b...)` |
| Destructuring | `const { name } = user` | No destructuring — use `user.Name` |
| Optional chaining | `user?.address?.city` | Explicit nil checks |
| Ternary | `x > 0 ? "pos" : "neg"` | No ternary — use if/else |
| Template literal | `` `hello ${name}` `` | `fmt.Sprintf("hello %s", name)` |
| Enum | `enum Direction { North }` | `const ( North Direction = iota )` |
| Generic | `function f<T>(x: T): T` | `func f[T any](x T) T` (Go 1.18+) |
| Module export | `export const x` | Capital letter: `var X` |
| Module import | `import { x } from './y'` | `import "github.com/you/project/y"` |

---

*Keep this file open while you code. The patterns repeat constantly — after a week they will feel natural.*
