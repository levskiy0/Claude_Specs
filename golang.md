# Go Style Guide

This style guide adapts core programming principles to idiomatic Go, based on official Go documentation, Effective Go, and community best practices.

## Core Principles

### 1. Store Repeating Values as Constants

Use typed constants with iota for enumerations and group related constants together.

```go
// Good: Typed constants with iota
type Environment int

const (
    Test Environment = iota
    Development
    Production
)

// Good: String constants with String() method
type Status string

const (
    StatusActive   Status = "active"
    StatusInactive Status = "inactive"
    StatusPending  Status = "pending"
)

func (s Status) String() string {
    return string(s)
}

// Bad: Magic strings scattered in code
if mode == "test" { // Don't do this
    // ...
}
```

### 2. Reuse Code - Don't Duplicate

Leverage interfaces, embedding, and generics (Go 1.18+) for code reusability.

```go
// Good: Interface-based abstraction
type Storage interface {
    Save(key string, data []byte) error
    Load(key string) ([]byte, error)
}

// Good: Embedding for composition
type User struct {
    Name  string
    Email string
}

type AdminUser struct {
    User // Embedded
    Permissions []string
}

// Good: Generics for type-safe reusability
func Max[T constraints.Ordered](a, b T) T {
    if a > b {
        return a
    }
    return b
}
```

### 3. Don't Write Monolithic Code

Follow Go's package-oriented design with small, focused packages.

```go
// Good: Organized package structure
myapp/
├── cmd/
│   └── api/
│       └── main.go
├── internal/
│   ├── user/
│   │   ├── service.go
│   │   └── repository.go
│   └── auth/
│       └── middleware.go
└── pkg/
    └── validator/
        └── validator.go

// Good: Single responsibility
package user

type Service struct {
    repo Repository
}

func (s *Service) CreateUser(ctx context.Context, email string) (*User, error) {
    // Focused on user creation logic
}
```

### 4. Don't Make Things Up

Always verify with official Go documentation and use established patterns.

```go
// Good: Check documentation at pkg.go.dev
// Use go doc command:
// go doc fmt.Printf
// go doc strings.Contains

// Good: Acknowledge when unsure
func ProcessData(data interface{}) error {
    // TODO: Research proper type assertion patterns
    // I don't know the best approach for this use case
    return errors.New("not implemented: need to research type assertions")
}
```

### 5. Don't Invent Non-existent Methods

Verify method existence before use and handle errors appropriately.

```go
// Good: Check if method exists
import "reflect"

func CallMethodIfExists(obj interface{}, methodName string) {
    v := reflect.ValueOf(obj)
    method := v.MethodByName(methodName)
    
    if method.IsValid() && method.Type().NumIn() == 0 {
        method.Call(nil)
    }
}

// Good: Use go doc to verify
// $ go doc strings.Contains
// $ go list -m all  # List all dependencies
```

## Go-Specific Best Practices

### Error Handling

```go
// Good: Wrap errors with context
func processFile(filename string) error {
    file, err := os.Open(filename)
    if err != nil {
        return fmt.Errorf("cannot open file %q: %w", filename, err)
    }
    defer file.Close()
    
    // Process file...
    return nil
}

// Good: Custom error types
type ValidationError struct {
    Field string
    Value interface{}
    Msg   string
}

func (e ValidationError) Error() string {
    return fmt.Sprintf("validation failed for %s: %s", e.Field, e.Msg)
}
```

### Naming Conventions

- **Exported names**: Start with capital letter (MixedCaps)
- **Unexported names**: Start with lowercase (mixedCaps)
- **Packages**: Lowercase, single word, no underscores
- **Interfaces**: Often end with -er suffix (Reader, Writer)
- **Constants**: Use MixedCaps, not ALL_CAPS

### Testing

```go
// Good: Table-driven tests
func TestAdd(t *testing.T) {
    tests := []struct {
        name string
        a, b int
        want int
    }{
        {"positive", 2, 3, 5},
        {"negative", -2, -3, -5},
        {"mixed", 2, -3, -1},
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            if got := Add(tt.a, tt.b); got != tt.want {
                t.Errorf("Add() = %v, want %v", got, tt.want)
            }
        })
    }
}
```

### Concurrency Patterns

```go
// Good: Explicit goroutine lifecycle
func worker(ctx context.Context, jobs <-chan Job, results chan<- Result) {
    for {
        select {
        case job, ok := <-jobs:
            if !ok {
                return // Channel closed, exit gracefully
            }
            result := processJob(job)
            results <- result
        case <-ctx.Done():
            return // Context cancelled
        }
    }
}
```

### Configuration Management

```go
// Good: Structured configuration
type Config struct {
    Environment string
    Database    DatabaseConfig
    Server      ServerConfig
}

type DatabaseConfig struct {
    Host     string
    Port     int
    User     string
    Password string
}

// Good: Environment-based configuration
func LoadConfig() (*Config, error) {
    env := os.Getenv("GO_ENV")
    if env == "" {
        env = "development"
    }
    
    configFile := fmt.Sprintf("config.%s.json", env)
    return loadConfigFromFile(configFile)
}
```

## Tools and Verification

- **gofmt**: Mandatory code formatting
- **go vet**: Static analysis
- **golangci-lint**: Comprehensive linting
- **go doc**: Documentation lookup
- **pkg.go.dev**: Online package documentation
- **gopls**: Language server for IDE support

## References

- [Effective Go](https://go.dev/doc/effective_go)
- [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments)
- [Go Doc Comments](https://tip.golang.org/doc/comment)
- [Google Go Style Guide](https://google.github.io/styleguide/go/)
- [Uber Go Style Guide](https://github.com/uber-go/guide)