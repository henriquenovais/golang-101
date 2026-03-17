# Go Learning Projects: CLI Task Runner & DB Migration Tool

> A complete guide for experienced TypeScript developers learning Go through practical projects.

---

## Table of Contents

1. [Before You Start](#before-you-start)
2. [Project 1: CLI Task Runner](#project-1-cli-task-runner)
3. [Project 2: DB Migration Tool](#project-2-db-migration-tool)
4. [TypeScript → Go Cheat Sheet](#typescript--go-cheat-sheet)

---

## Before You Start

### System Setup on Pop!_OS

Pop!_OS does not ship with Go. Follow these steps on your practice notebook to get a clean, properly configured environment.

#### 1. Install Go via the official tarball

Do not use `apt` to install Go — the version in Ubuntu/Pop!_OS repositories is often significantly out of date. Use the official release instead:

```bash
# Remove any old Go installation if present
sudo rm -rf /usr/local/go

# Download the latest stable release (check https://go.dev/dl for the current version)
wget https://go.dev/dl/go1.22.4.linux-amd64.tar.gz

# Extract to /usr/local
sudo tar -C /usr/local -xzf go1.22.4.linux-amd64.tar.gz

# Clean up the tarball
rm go1.22.4.linux-amd64.tar.gz
```

#### 2. Configure your PATH

Pop!_OS uses `~/.bashrc` for terminal sessions. Add Go's bin directory and your personal Go workspace bin:

```bash
echo '' >> ~/.bashrc
echo '# Go' >> ~/.bashrc
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> ~/.bashrc

# Apply immediately in the current terminal
source ~/.bashrc
```

If you use `zsh` (which Pop!_OS does not set by default, but you may have configured it), replace `~/.bashrc` with `~/.zshrc`.

#### 3. Verify the installation

```bash
go version
# Expected: go version go1.22.4 linux/amd64

go env GOPATH
# Expected: /home/youruser/go
```

#### 4. Install VS Code Go extension (recommended)

If you use VS Code, install the official Go extension. It handles formatting, linting, and test running inside the editor. First, install the underlying tools:

```bash
# Install gopls (the Go language server) and the debugger
go install golang.org/x/tools/gopls@latest
go install github.com/go-delve/delve/cmd/dlv@latest
```

Then in VS Code: open the Command Palette (`Ctrl+Shift+P`) → `Go: Install/Update Tools` → select all → OK.

#### 5. Install git (if not already present)

```bash
sudo apt update && sudo apt install -y git
```

---

### Initialize a Go Module

Go uses modules instead of `package.json`. Each project starts with:

```bash
mkdir my-project && cd my-project
go mod init github.com/yourname/my-project
```

This creates a `go.mod` file — the Go equivalent of `package.json`. The module path (`github.com/yourname/my-project`) is used for imports within your project. It does not need to match a real GitHub URL while you are learning, but it is good practice to use one that would.

---

### Key Mindset Shifts from TypeScript

| TypeScript | Go |
|---|---|
| Errors are thrown and caught | Errors are returned as values |
| `async/await` for concurrency | Goroutines and channels |
| `interface` with `implements` | Interfaces are implicit (structural) |
| `class` with inheritance | `struct` with composition |
| `npm install` | `go get` |
| `node_modules/` | Module cache (`~/go/pkg/mod`) |
| `tsc` compiles to JS | `go build` compiles to a native binary |
| `jest` for testing | `go test` built into the toolchain |
| `.eslintrc` + `prettier` | `go vet` + `go fmt` (built-in, non-negotiable) |

---

---

# Project 1: CLI Task Runner

## What You Are Building

A command-line tool that reads a `tasks.yaml` config file and executes named shell commands. Think of it as your own version of `npm run` or `make`.

```bash
./runner run test          # runs a single task
./runner run ci            # runs a task with dependencies
./runner list              # lists all available tasks
```

---

## The Config File

Users define their tasks in a `tasks.yaml` at the root of their project:

```yaml
tasks:
  lint: "golangci-lint run ./..."
  test: "go test ./..."
  build: "go build -o bin/app ./cmd/app"
  clean: "rm -rf bin/"
  ci:
    deps: [lint, test, build]
```

- Simple tasks map a name directly to a shell command string.
- Compound tasks (like `ci`) declare dependencies and no command of their own.
- The runner resolves dependencies and executes them in the correct order.

---

## Project Structure

```
cli-task-runner/
├── cmd/
│   └── runner/
│       └── main.go              # Entry point — wires everything together
├── internal/
│   ├── task/
│   │   ├── task.go              # Domain: Task struct and Executor interface
│   │   ├── graph.go             # Domain: cycle detection logic
│   │   └── task_test.go
│   ├── runner/
│   │   ├── runner.go            # Use case: resolve deps, execute tasks
│   │   └── runner_test.go
│   ├── config/
│   │   ├── config.go            # Adapter: parse tasks.yaml into domain types
│   │   └── config_test.go
│   └── shell/
│       ├── shell.go             # Adapter: real OS command executor
│       └── shell_test.go
├── go.mod
├── go.sum
└── tasks.yaml                   # Example config for manual testing
```

> **Why `internal/`?** Go treats the `internal` directory specially — packages inside it cannot be imported by code outside the module. This enforces your Clean Architecture boundaries at the compiler level, for free. There is no need for barrel files or explicit exports.

> **Why `cmd/`?** A Go project can have multiple entry points (multiple `main` packages). The `cmd/` directory is the convention for housing them. Each subdirectory under `cmd/` compiles to a separate binary.

---

## Phase 1 — Domain Model and Config Parsing

**Goal:** Define what a `Task` is and be able to load tasks from a `tasks.yaml` file into domain objects.

**Go concepts you will practice:** structs, struct tags, slices, maps, error wrapping with `fmt.Errorf`, `os.ReadFile`, table-driven tests.

**Dependency to install:**
```bash
go get gopkg.in/yaml.v3
```

---

### Requirements

#### REQ-1.1 — Task domain model

> **Given** the project domain,
> **When** I define the core data structure,
> **Then** a `Task` struct must exist in `internal/task/task.go` with the fields: `Name` (string), `Command` (string), and `Deps` (slice of strings).

---

#### REQ-1.2 — Load a simple task from YAML

> **Given** a `tasks.yaml` file containing a task with only a command string (e.g. `test: "go test ./..."`),
> **When** `config.Load` is called with the path to that file,
> **Then** it must return a slice containing one `Task` with the correct `Name` and `Command` fields, and no error.

---

#### REQ-1.3 — Load a compound task with dependencies

> **Given** a `tasks.yaml` file containing a task with a `deps` list and no command (e.g. `ci: deps: [lint, test]`),
> **When** `config.Load` is called with the path to that file,
> **Then** it must return a `Task` with the correct `Name`, an empty `Command`, and a `Deps` slice matching the declared dependencies, and no error.

---

#### REQ-1.4 — Load a mixed config with multiple tasks

> **Given** a `tasks.yaml` file containing both simple and compound tasks,
> **When** `config.Load` is called,
> **Then** it must return a slice with all tasks present, preserving each task's name, command, and deps accurately.

---

#### REQ-1.5 — Return an error for a missing file

> **Given** a file path that does not exist on disk,
> **When** `config.Load` is called with that path,
> **Then** it must return a non-nil error and a nil slice.

---

#### REQ-1.6 — Return an error for malformed YAML

> **Given** a file containing invalid YAML syntax,
> **When** `config.Load` is called with that path,
> **Then** it must return a non-nil error describing the parse failure and a nil slice.

---

## Phase 2 — Command Execution with Dependency Injection

**Goal:** Execute a task's shell command by delegating to an `Executor` interface, making the runner fully testable without real shell processes.

**Go concepts you will practice:** interfaces (implicit satisfaction), pointer receivers, `io.Writer` injection, mock structs in tests.

---

### Requirements

#### REQ-2.1 — Executor interface

> **Given** the need to run shell commands and keep the runner testable,
> **When** I define the execution contract,
> **Then** an `Executor` interface must exist in `internal/task/task.go` with a single method: `Run(command string, stdout io.Writer, stderr io.Writer) error`.

---

#### REQ-2.2 — Run a known task

> **Given** a `Runner` initialised with a list of tasks and a mock `Executor`,
> **When** `Run` is called with the name of a task that has a command,
> **Then** the mock `Executor` must receive exactly that command string, and `Run` must return nil.

---

#### REQ-2.3 — Return an error for an unknown task

> **Given** a `Runner` initialised with a list of tasks,
> **When** `Run` is called with a task name that does not exist in the list,
> **Then** `Run` must return a non-nil error and the mock `Executor` must not be called.

---

#### REQ-2.4 — Propagate executor errors

> **Given** a `Runner` with a mock `Executor` configured to return an error for a specific command,
> **When** `Run` is called with the task that maps to that command,
> **Then** `Run` must return a non-nil error wrapping the executor's error.

---

#### REQ-2.5 — Run dependencies before the task

> **Given** a compound task `ci` with `deps: [lint, test]`, and simple tasks `lint` and `test` each with their own commands,
> **When** `Run` is called with `"ci"`,
> **Then** the mock `Executor` must be called first with `lint`'s command, then with `test`'s command, in that order, and `Run` must return nil.

---

#### REQ-2.6 — Stop execution if a dependency fails

> **Given** a compound task `ci` with `deps: [lint, test]`, where the mock `Executor` returns an error for `lint`'s command,
> **When** `Run` is called with `"ci"`,
> **Then** `Run` must return a non-nil error and must not execute `test`'s command.

---

#### REQ-2.7 — Write output to the injected writer

> **Given** a `Runner` initialised with a `bytes.Buffer` as its output writer,
> **When** a task is successfully executed,
> **Then** the buffer must contain a log line indicating which task was run.

---

## Phase 3 — Cycle Detection

**Goal:** Detect circular dependencies in the task graph before any execution begins, preventing infinite loops.

**Go concepts you will practice:** recursive functions, maps as sets, depth-first search, returning wrapped errors.

---

### Requirements

#### REQ-3.1 — No error for a valid linear graph

> **Given** a set of tasks where `ci` depends on `lint` and `test`, and `lint` and `test` have no dependencies,
> **When** cycle detection is run,
> **Then** it must return nil.

---

#### REQ-3.2 — Detect a direct self-reference

> **Given** a task `build` that lists itself in its own `deps`,
> **When** cycle detection is run,
> **Then** it must return a non-nil error mentioning `"build"`.

---

#### REQ-3.3 — Detect a two-node cycle

> **Given** task `a` depends on `b`, and task `b` depends on `a`,
> **When** cycle detection is run,
> **Then** it must return a non-nil error.

---

#### REQ-3.4 — Detect a deep transitive cycle

> **Given** a chain where `a → b → c → a`,
> **When** cycle detection is run,
> **Then** it must return a non-nil error.

---

#### REQ-3.5 — Cycle detection runs before execution

> **Given** a `Runner` initialised with a task graph containing a cycle,
> **When** `Run` is called,
> **Then** it must return a non-nil error immediately and the `Executor` must not be called.

---

## Phase 4 — Parallel Execution

**Goal:** Run tasks with no dependencies between them concurrently using goroutines and `sync.WaitGroup`.

**Go concepts you will practice:** goroutines (`go` keyword), `sync.WaitGroup`, buffered channels for error collection, the race detector (`go test -race`).

---

### Requirements

#### REQ-4.1 — Run independent tasks concurrently

> **Given** a `Runner` and a list of task names with no dependencies between them,
> **When** `RunParallel` is called with those names,
> **Then** all tasks must be executed and `RunParallel` must return nil.

---

#### REQ-4.2 — Return an error if any parallel task fails

> **Given** a list of three independent tasks where one is configured to fail,
> **When** `RunParallel` is called,
> **Then** `RunParallel` must return a non-nil error after all goroutines complete.

---

#### REQ-4.3 — Execute all tasks even if one fails

> **Given** a list of three independent tasks where one fails,
> **When** `RunParallel` is called,
> **Then** the remaining two tasks must still be executed (no early exit), and all three `Executor` calls must be recorded.

---

#### REQ-4.4 — No data races

> **Given** any call to `RunParallel` with multiple tasks,
> **When** tests are run with `go test -race ./...`,
> **Then** the race detector must report zero data races.

---

## Phase 5 — CLI Entry Point

**Goal:** Wire all components together into a working binary that a user can build and run from their terminal.

**Go concepts you will practice:** `os.Args`, `os.Exit`, `fmt.Fprintf(os.Stderr, ...)`, the `main` package, building a binary with `go build`.

---

### Requirements

#### REQ-5.1 — Run a task by name

> **Given** a valid `tasks.yaml` in the current directory,
> **When** the user runs `./runner run <taskname>`,
> **Then** the binary must execute the named task and exit with code `0`.

---

#### REQ-5.2 — Exit with a non-zero code on task failure

> **Given** a task whose shell command exits with a non-zero status,
> **When** the user runs `./runner run <taskname>`,
> **Then** the binary must print an error to stderr and exit with a non-zero code.

---

#### REQ-5.3 — Exit with a non-zero code for an unknown task

> **Given** a `tasks.yaml` that does not contain a task named `"ghost"`,
> **When** the user runs `./runner run ghost`,
> **Then** the binary must print an error to stderr and exit with a non-zero code.

---

#### REQ-5.4 — List all available tasks

> **Given** a valid `tasks.yaml`,
> **When** the user runs `./runner list`,
> **Then** the binary must print each task name to stdout and exit with code `0`.

---

#### REQ-5.5 — Print usage for missing or invalid arguments

> **Given** the binary is invoked with no arguments or an unrecognised subcommand,
> **When** it starts,
> **Then** it must print a usage message to stderr and exit with a non-zero code.

---

---

# Project 2: DB Migration Tool

## What You Are Building

A CLI tool that manages database schema changes through versioned SQL files — a simplified version of `golang-migrate` or Flyway.

```bash
./migrate up                        # apply all pending migrations
./migrate down                      # roll back the last applied migration
./migrate status                    # show which migrations have run
./migrate create add_users_table    # scaffold a new migration file pair
```

---

## How Migrations Work

Each migration is a pair of `.sql` files in a `migrations/` directory:

```
migrations/
├── 001_create_users.up.sql
├── 001_create_users.down.sql
├── 002_add_email_index.up.sql
└── 002_add_email_index.down.sql
```

- `.up.sql` — the change to apply (`CREATE TABLE`, `ALTER TABLE`, etc.)
- `.down.sql` — how to reverse it (`DROP TABLE`, etc.)

The tool tracks which migrations have been applied in a `schema_migrations` table inside the database itself:

```sql
CREATE TABLE IF NOT EXISTS schema_migrations (
    version    TEXT PRIMARY KEY,
    applied_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

**On databases:** you will use SQLite for everything. SQLite is a file-based database — no server, no Docker, no installation needed. For tests, you will use an in-memory SQLite instance (`:memory:`) that is created and destroyed per test run. Swapping to PostgreSQL later only requires changing the driver import and connection string — your migration logic stays identical.

---

## Project Structure

```
db-migration-tool/
├── cmd/
│   └── migrate/
│       └── main.go              # Entry point — wires everything together
├── internal/
│   ├── migration/
│   │   ├── migration.go         # Domain: Migration struct
│   │   └── migration_test.go
│   ├── store/
│   │   ├── store.go             # Interface: MigrationStore contract
│   │   ├── sqlite.go            # Adapter: SQLite implementation
│   │   └── sqlite_test.go       # Tests against :memory: SQLite
│   ├── fs/
│   │   ├── loader.go            # Adapter: read migration files from disk
│   │   ├── creator.go           # Adapter: scaffold new migration file pairs
│   │   └── fs_test.go
│   └── migrator/
│       ├── migrator.go          # Use case: up, down, status logic
│       └── migrator_test.go     # Tests using a mock store
├── migrations/                  # Your SQL files live here
├── go.mod
└── go.sum
```

> **Why is `store.go` an interface and `sqlite.go` a separate file?** The interface defines the contract your use case (`migrator`) depends on. The SQLite file is an adapter that satisfies that contract. Your `migrator` package imports the interface, never the SQLite adapter directly. This means you can test the migrator with a mock store and only test the SQLite adapter against a real (in-memory) database. This is the Dependency Inversion Principle expressed in idiomatic Go.

---

## Phase 1 — Domain Model

**Goal:** Define what a migration is as a domain concept, independent of any database or file system concern.

**Go concepts you will practice:** structs, pointer fields (for nullable values), value receiver methods.

---

### Requirements

#### REQ-1.1 — Migration domain struct

> **Given** the project domain,
> **When** I define the core data structure,
> **Then** a `Migration` struct must exist in `internal/migration/migration.go` with the fields: `Version` (string), `Name` (string), `UpSQL` (string), `DownSQL` (string), and `AppliedAt` (pointer to `time.Time`, nil when not yet applied).

---

#### REQ-1.2 — IsApplied returns true for applied migrations

> **Given** a `Migration` with a non-nil `AppliedAt`,
> **When** `IsApplied()` is called,
> **Then** it must return `true`.

---

#### REQ-1.3 — IsApplied returns false for pending migrations

> **Given** a `Migration` with a nil `AppliedAt`,
> **When** `IsApplied()` is called,
> **Then** it must return `false`.

---

## Phase 2 — Reading Migration Files from Disk

**Goal:** Scan a directory for `.up.sql` / `.down.sql` file pairs and parse them into domain `Migration` objects, sorted by version ascending.

**Go concepts you will practice:** `os.ReadDir`, `filepath.Join`, `strings` package, `sort.Strings`, returning structured errors.

---

### Requirements

#### REQ-2.1 — Load a single migration pair

> **Given** a directory containing `001_create_users.up.sql` and `001_create_users.down.sql`,
> **When** `fs.Load` is called with that directory path,
> **Then** it must return a slice with one `Migration` where `Version` is `"001"`, `Name` is `"create_users"`, `UpSQL` matches the file content, and `DownSQL` matches the file content.

---

#### REQ-2.2 — Load multiple pairs sorted by version

> **Given** a directory containing migration pairs for versions `"003"`, `"001"`, and `"002"` in any file system order,
> **When** `fs.Load` is called,
> **Then** it must return a slice of three migrations sorted ascending by version: `001`, `002`, `003`.

---

#### REQ-2.3 — Ignore non-migration files

> **Given** a directory containing valid migration pairs alongside unrelated files (e.g. `README.md`, `.DS_Store`),
> **When** `fs.Load` is called,
> **Then** only valid migration pairs must be returned and unrelated files must be silently ignored.

---

#### REQ-2.4 — Return an error for a missing directory

> **Given** a directory path that does not exist,
> **When** `fs.Load` is called with that path,
> **Then** it must return a non-nil error and a nil slice.

---

#### REQ-2.5 — Scaffold a new migration file pair

> **Given** a target directory and a migration name (e.g. `"add_users_table"`),
> **When** `fs.Create` is called,
> **Then** two new files must be created in the directory: one ending in `.up.sql` and one ending in `.down.sql`, both prefixed with a timestamp-based version string, and both containing a placeholder comment.

---

#### REQ-2.6 — Created migration files have unique versions

> **Given** `fs.Create` is called twice in sequence,
> **When** both calls succeed,
> **Then** the two resulting file pairs must have different version prefixes so they do not collide.

---

## Phase 3 — The Store Interface and SQLite Adapter

**Goal:** Define the persistence contract as an interface, then implement it against a real SQLite database. Test the adapter using an in-memory database.

**Go concepts you will practice:** interfaces, blank imports (`_ "modernc.org/sqlite"`), `database/sql`, `rows.Close()` with `defer`, `*sql.DB`, in-memory SQLite for tests.

**Dependency to install:**
```bash
go get modernc.org/sqlite
```

---

### Requirements

#### REQ-3.1 — Store interface contract

> **Given** the need to decouple the migrator from any specific database,
> **When** I define the persistence contract,
> **Then** a `Store` interface must exist in `internal/store/store.go` with the methods: `Init() error`, `Applied() (map[string]bool, error)`, `Record(version string) error`, `Remove(version string) error`, and `Exec(sql string) error`.

---

#### REQ-3.2 — Init creates the tracking table

> **Given** a fresh SQLite in-memory database,
> **When** `Init()` is called,
> **Then** a `schema_migrations` table must exist in the database and `Init` must return nil.

---

#### REQ-3.3 — Init is idempotent

> **Given** a database where `Init()` has already been called once,
> **When** `Init()` is called again,
> **Then** it must return nil without error (uses `CREATE TABLE IF NOT EXISTS`).

---

#### REQ-3.4 — Applied returns empty map when no migrations recorded

> **Given** a freshly initialised store with no recorded migrations,
> **When** `Applied()` is called,
> **Then** it must return an empty map and nil error.

---

#### REQ-3.5 — Record persists a migration version

> **Given** a freshly initialised store,
> **When** `Record("001")` is called,
> **Then** a subsequent call to `Applied()` must return a map containing `"001": true`.

---

#### REQ-3.6 — Remove deletes a recorded migration

> **Given** a store where version `"001"` has been recorded,
> **When** `Remove("001")` is called,
> **Then** a subsequent call to `Applied()` must return a map that does not contain `"001"`.

---

#### REQ-3.7 — Exec runs arbitrary SQL against the database

> **Given** a freshly initialised store,
> **When** `Exec` is called with a valid `CREATE TABLE` statement,
> **Then** the table must exist in the database and `Exec` must return nil.

---

#### REQ-3.8 — Exec returns an error for invalid SQL

> **Given** a store with a live database connection,
> **When** `Exec` is called with malformed SQL,
> **Then** it must return a non-nil error.

---

## Phase 4 — The Migrator Use Case

**Goal:** Implement `Up`, `Down`, and `Status` by composing the `Store` interface and the loaded `Migration` slice. Tests use a mock store — no database involved.

**Go concepts you will practice:** interface-driven use cases, mock structs satisfying interfaces, `io.Writer` injection for output, iterating slices in reverse.

---

### Requirements

#### REQ-4.1 — Up applies all pending migrations in order

> **Given** a migrator with two migrations (`001`, `002`) and a mock store where neither is applied,
> **When** `Up()` is called,
> **Then** the store's `Exec` must be called twice (once per migration's `UpSQL`) in ascending version order, both versions must be recorded, and `Up` must return nil.

---

#### REQ-4.2 — Up skips already applied migrations

> **Given** a migrator with two migrations where `001` is already recorded as applied,
> **When** `Up()` is called,
> **Then** the store's `Exec` must be called only once (for `002`), and `Up` must return nil.

---

#### REQ-4.3 — Up returns an error if Exec fails

> **Given** a mock store whose `Exec` returns an error,
> **When** `Up()` is called,
> **Then** `Up` must return a non-nil error and must not call `Record` for the failed migration.

---

#### REQ-4.4 — Up stops on first failure

> **Given** two pending migrations where the first migration's `Exec` fails,
> **When** `Up()` is called,
> **Then** `Up` must return an error and must not attempt to execute the second migration.

---

#### REQ-4.5 — Down rolls back the most recently applied migration

> **Given** a migrator with migrations `001` and `002` where both are applied,
> **When** `Down()` is called,
> **Then** the store's `Exec` must be called with `002`'s `DownSQL`, version `"002"` must be removed from the store, and `Down` must return nil.

---

#### REQ-4.6 — Down only rolls back one migration per call

> **Given** three applied migrations (`001`, `002`, `003`),
> **When** `Down()` is called,
> **Then** only `003`'s `DownSQL` must be executed, and only `"003"` must be removed. Migrations `001` and `002` must remain applied.

---

#### REQ-4.7 — Down does nothing when no migrations are applied

> **Given** a migrator with migrations where none are applied,
> **When** `Down()` is called,
> **Then** the store's `Exec` must not be called and `Down` must return nil.

---

#### REQ-4.8 — Status prints applied and pending migrations

> **Given** two migrations where `001` is applied and `002` is pending, and a `bytes.Buffer` as the output writer,
> **When** `Status()` is called,
> **Then** the buffer must contain a line for `001` labelled `"applied"` and a line for `002` labelled `"pending"`.

---

## Phase 5 — The `create` Subcommand

**Goal:** Allow users to scaffold new migration file pairs from the CLI without writing the filenames manually.

**Go concepts you will practice:** `time.Now().Format(...)`, `os.WriteFile`, `filepath.Join`, handling subcommand arguments from `os.Args`.

---

### Requirements

#### REQ-5.1 — Create scaffolds an up and a down file

> **Given** an existing `migrations/` directory and the name `"add_users_table"`,
> **When** `./migrate create add_users_table` is run,
> **Then** two new files must appear in the `migrations/` directory — one `.up.sql` and one `.down.sql` — both with matching version prefixes.

---

#### REQ-5.2 — Created files contain placeholder comments

> **Given** a newly scaffolded migration pair,
> **When** their contents are read,
> **Then** both files must contain at least one SQL comment line indicating where the migration SQL should be written.

---

#### REQ-5.3 — Exit with error if name argument is missing

> **Given** the user runs `./migrate create` with no name argument,
> **When** the binary starts,
> **Then** it must print a usage message to stderr and exit with a non-zero code.

---

## Phase 6 — CLI Entry Point

**Goal:** Wire all components together into a working binary. Each subcommand (`up`, `down`, `status`, `create`) maps to the appropriate use case.

**Go concepts you will practice:** `switch` on strings, `os.Args`, `os.Exit`, separating concerns between `main` (wiring) and `internal` packages (logic).

---

### Requirements

#### REQ-6.1 — `up` applies all pending migrations

> **Given** a `migrations/` directory with pending migrations and a `migrations.db` file (created if absent),
> **When** the user runs `./migrate up`,
> **Then** all pending migrations must be applied and the binary must exit with code `0`.

---

#### REQ-6.2 — `down` rolls back the last migration

> **Given** at least one applied migration,
> **When** the user runs `./migrate down`,
> **Then** the most recently applied migration must be rolled back and the binary must exit with code `0`.

---

#### REQ-6.3 — `status` prints migration state

> **Given** a mix of applied and pending migrations,
> **When** the user runs `./migrate status`,
> **Then** each migration must be listed with its status (`applied` or `pending`) and the binary must exit with code `0`.

---

#### REQ-6.4 — `create` scaffolds a new migration pair

> **Given** a `migrations/` directory,
> **When** the user runs `./migrate create <name>`,
> **Then** a new `.up.sql` and `.down.sql` pair must be created in `migrations/` and the binary must print the created file paths to stdout.

---

#### REQ-6.5 — Non-zero exit on any failure

> **Given** any subcommand that encounters an error (bad SQL, missing directory, etc.),
> **When** the error occurs,
> **Then** the binary must print a descriptive error to stderr and exit with a non-zero code.

---

#### REQ-6.6 — Print usage for unknown or missing subcommands

> **Given** the binary is invoked with no arguments or an unrecognised subcommand,
> **When** it starts,
> **Then** it must print a list of available subcommands to stderr and exit with a non-zero code.

---

---

# TypeScript → Go Cheat Sheet

## Error Handling

```typescript
// TypeScript
try {
  const data = await readFile("config.yaml");
} catch (err) {
  console.error(err);
}
```

```go
// Go
data, err := os.ReadFile("config.yaml")
if err != nil {
    fmt.Fprintf(os.Stderr, "error: %v\n", err)
    return
}
```

> In Go, errors are values returned from functions. You handle them where they occur, not at a distance via try/catch. The `%w` verb in `fmt.Errorf("context: %w", err)` wraps an error so callers can inspect it later with `errors.Is` or `errors.As`.

---

## Interfaces

```typescript
// TypeScript — must declare implements
interface Runner {
  run(cmd: string): Promise<void>;
}
class ShellRunner implements Runner { ... }
```

```go
// Go — implicit. No implements keyword.
type Runner interface {
    Run(cmd string) error
}
// ShellRunner satisfies Runner automatically if it has a matching Run method.
type ShellRunner struct{}
func (s *ShellRunner) Run(cmd string) error { ... }
```

---

## Concurrency

```typescript
// TypeScript
await Promise.all([task1(), task2(), task3()]);
```

```go
// Go
var wg sync.WaitGroup
errs := make(chan error, len(tasks))

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
```

---

## Dependency Injection and Mocking

```typescript
// TypeScript
const runner = new Runner({ executor: mockExecutor });
```

```go
// Go — pass the interface, inject any implementation
r := runner.New(tasks, mockExecutor, &bytes.Buffer{})
```

---

## Running Tests

```bash
# Run all tests
go test ./...

# Run with verbose output (shows each test name)
go test -v ./...

# Run a specific test by name
go test -v -run TestMigrator_Up ./internal/migrator/...

# Run with the race detector — always do this before shipping
go test -race ./...

# Run tests and show coverage
go test -cover ./...
```

---

## Useful Daily Commands

```bash
go build ./...          # compile everything, check for errors
go test ./...           # run all tests
go vet ./...            # static analysis — catches common mistakes
go fmt ./...            # format all code (non-negotiable in Go, run this constantly)
go mod tidy             # clean up go.mod and go.sum after adding or removing deps
go doc <package>        # read documentation for any package inline
```

---

*Good luck. The moment Go's error handling and concurrency model click will feel like a genuine shift — not just new syntax, but a different way of thinking about reliability.*
