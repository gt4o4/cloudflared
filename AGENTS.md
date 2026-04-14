# Cloudflared

Cloudflare's command-line tool and networking daemon written in Go.
Production-grade tunneling and network connectivity services used by millions of
developers and organizations worldwide.

## Essential Commands

### Build & Test (Always run before commits)

```bash
# Full development check (run before any commit)
make test lint

# Build for current platform
make cloudflared

# Run all unit tests with coverage
make test
make cover

# Run specific test
go test -run TestFunctionName ./path/to/package

# Run tests with race detection
go test -race ./...
```

### Platform-Specific Builds

```bash
# Linux
TARGET_OS=linux TARGET_ARCH=amd64 make cloudflared

# Windows
TARGET_OS=windows TARGET_ARCH=amd64 make cloudflared

# macOS ARM64
TARGET_OS=darwin TARGET_ARCH=arm64 make cloudflared

# FIPS compliant build
FIPS=true make cloudflared
```

### Code Quality & Formatting

```bash
# Run linter (38+ enabled linters)
make lint

# Auto-fix formatting
make fmt
gofmt -w .
goimports -w .

# Security scanning
make vet

# Component tests (Python integration tests)
cd component-tests && python -m pytest test_file.py::test_function_name
```

## Project Knowledge

### Package Structure

- Use meaningful package names that reflect functionality
- Package names should be lowercase, single words when possible
- Avoid generic names like `util`, `common`, `helper`

### Function and Method Guidelines

```go
// Good: Clear purpose, proper error handling
func (c *Connection) HandleRequest(ctx context.Context, req *http.Request) error {
    if req == nil {
        return errors.New("request cannot be nil")
    }
    // Implementation...
    return nil
}
```

### Error Handling

- Always handle errors explicitly, never ignore them
- Use `fmt.Errorf` for error wrapping
- Create meaningful error messages with context
- Use error variables for common errors

```go
// Good error handling patterns
if err != nil {
    return fmt.Errorf("failed to process connection: %w", err)
}
```

### Logging Standards

- Use `github.com/rs/zerolog` for structured logging
- Include relevant context fields
- Use appropriate log levels (Debug, Info, Warn, Error)

```go
logger.Info().
    Str("tunnelID", tunnel.ID).
    Int("connIndex", connIndex).
    Msg("Connection established")
```

### Testing Patterns

- Use `github.com/stretchr/testify` for assertions
- Test files end with `_test.go`
- Use table-driven tests for multiple scenarios
- Always use `t.Parallel()` for parallel-safe tests
- Use meaningful test names that describe behavior

```go
func TestMetricsListenerCreation(t *testing.T) {
    t.Parallel()
    // Test implementation
    assert.Equal(t, expected, actual)
    require.NoError(t, err)
}
```

### Constants and Variables

```go
const (
    MaxGracePeriod       = time.Minute * 3
    MaxConcurrentStreams = math.MaxUint32
    LogFieldConnIndex    = "connIndex"
)

var (
    // Group related variables
    switchingProtocolText = fmt.Sprintf("%d %s", http.StatusSwitchingProtocols, http.StatusText(http.StatusSwitchingProtocols))
    flushableContentTypes = []string{sseContentType, grpcContentType, sseJsonContentType}
)
```

### Type Definitions

- Define interfaces close to their usage
- Keep interfaces small and focused
- Use descriptive names for complex types

```go
type TunnelConnection interface {
    Serve(ctx context.Context) error
}

type TunnelProperties struct {
    Credentials    Credentials
    QuickTunnelUrl string
}
```

## Key Architectural Patterns

### Context Usage

- Always accept `context.Context` as first parameter for long-running operations
- Respect context cancellation in loops and blocking operations
- Pass context through call chains

### Concurrency

- Use channels for goroutine communication
- Protect shared state with mutexes
- Prefer `sync.RWMutex` for read-heavy workloads

### Configuration

- Use structured configuration with validation
- Support both file-based and CLI flag configuration
- Provide sensible defaults

### Metrics and Observability

- Instrument code with Prometheus metrics
- Use OpenTelemetry for distributed tracing
- Include structured logging with relevant context

## Boundaries

### ✅ Always Do

- Run `make test lint` before any commit
- Handle all errors explicitly with proper context
- Use `github.com/rs/zerolog` for all logging
- Add `t.Parallel()` to all parallel-safe tests
- Follow the import grouping conventions
- Use meaningful variable and function names
- Include context.Context for long-running operations
- Close resources in defer statements

### ⚠️ Ask First Before

- Adding new dependencies to go.mod
- Modifying CI/CD configuration files
- Changing build system or Makefile
- Modifying component test infrastructure
- Adding new linter rules or changing golangci-lint config
- Making breaking changes to public APIs
- Changing logging levels or structured logging fields

### 🚫 Never Do

- Ignore errors without explicit handling (`_ = err`)
- Use generic package names (`util`, `helper`, `common`)
- Commit code that fails `make test lint`
- Use `fmt.Print*` instead of structured logging
- Modify vendor dependencies directly
- Commit secrets, credentials, or sensitive data
- Use deprecated or unsafe Go patterns
- Skip testing for new functionality
- Remove existing tests unless they're genuinely invalid

## Fork Structure & Merging

### Remotes

| Remote     | URL                                          | Role                              |
|------------|----------------------------------------------|-----------------------------------|
| `origin`   | https://github.com/cloudflare/cloudflared    | Official cloudflare source        |
| `upstream` | https://github.com/QudsLab/Cloudflared       | QudsLab fork (prebuilt binaries)  |
| `gt4o4`    | https://github.com/gt4o4/cloudflared         | Our working fork                  |

### Branches

| Branch  | Purpose                                                       |
|---------|---------------------------------------------------------------|
| `main`  | CGO shared-library exports (`lib_bin_exports.go`) + CI tweaks |
| `cflib` | Go module API: `package cloudflared` in `cmd/cloudflared/`    |

### Key Modifications (vs official cloudflare/cloudflared)

All `cmd/cloudflared/*.go` files use `package cloudflared` (not `package main`).
This makes the package importable as a Go module:

```go
import "github.com/cloudflare/cloudflared/cmd/cloudflared"

cloudflared.Init()
cloudflared.StartQuickTunnel(8080)
```

Changes from upstream:
- `main.go`: `main()` refactored to `initApp()` + `runAppWithArgs()`, API functions added (Init, Stop, Run, StartQuickTunnel, RunNamed, GetURL, IsReady)
- `tunnel/quick_tunnel.go`: `var OnURLReady func(string)` callback replaces CGO/cflib state
- `lib_bin_exports.go` (`//go:build cgo`): CGO exports wire `tunnel.OnURLReady` via `wireOnURLReady()`
- Service files (generic, linux, macos, windows): `runApp()` accepts `args []string` instead of using `os.Args`
- `updates/modified_files/`: kept in sync with `cmd/cloudflared/` for the replace.py workflow

### Merging Upstream Changes

```bash
git fetch --all
git checkout cflib

# Merge official cloudflare first (smaller, less conflict)
git merge origin/master -m "Merge origin/master (official cloudflare/cloudflared) into cflib"

# Then QudsLab fork
git merge upstream/main -m "Merge upstream/main (QudsLab/Cloudflared) into cflib"

# Verify
go vet ./cmd/cloudflared/...
go build ./cmd/cloudflared/...

git push gt4o4 cflib
```

When resolving conflicts, keep our side for:
- `package cloudflared` declarations (not `package main`)
- `var OnURLReady func(string)` callback in `quick_tunnel.go`
- `runApp(..., args []string)` signatures in service files
- `wireOnURLReady()` calls in `lib_bin_exports.go`
- API functions (Init/Stop/Run/etc.) in `main.go`

After resolving, sync `updates/modified_files/` to match `cmd/cloudflared/`.

## Dependencies Management

- Use Go modules (`go.mod`) exclusively
- Vendor dependencies for reproducible builds
- Keep dependencies up-to-date and secure
- Prefer standard library when possible
- Cloudflared uses a fork of quic-go always check release notes before bumping
  this dependency.

## Security Considerations

- FIPS compliance support available
- Vulnerability scanning integrated in CI
- Credential handling follows security best practices
- Network security with TLS/QUIC protocols
- Regular security audits and updates
- Post quantum encryption

## Common Patterns to Follow

1. **Graceful shutdown**: Always implement proper cleanup
2. **Resource management**: Close resources in defer statements
3. **Error propagation**: Wrap errors with meaningful context
4. **Configuration validation**: Validate inputs early
5. **Logging consistency**: Use structured logging throughout
6. **Testing coverage**: Aim for comprehensive test coverage
7. **Documentation**: Comment exported functions and types

Remember: This is a mission-critical networking tool used in production by many
organizations. Code quality, security, and reliability are paramount.
