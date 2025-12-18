---
title: Asynq slog Adapter
---

# ADR-06: Asynq Logging with slog Adapter

> 🇰🇷 [한국어 버전](/ko/adr/collector/06-asynq-slog-adapter.md)

| Date       | Author       | Repos     |
| ---------- | ------------ | --------- |
| 2024-12-18 | @KubrickCode | collector |

## Context

### The Problem: stderr vs stdout in Cloud Platforms

Asynq uses Go's standard `log` package by default, which outputs to stderr. Many cloud platforms and log aggregation services classify stderr output as ERROR level, regardless of the actual log severity.

**Impact:**

- INFO-level Asynq messages appear as errors in monitoring dashboards
- Alerts may be triggered by normal operational logs
- Difficult to distinguish actual errors from routine messages
- Log filtering becomes unreliable

### Structured Logging Requirements

Modern observability practices prefer structured (JSON) logging for:

- Machine-parseable log entries
- Consistent field extraction
- Correlation across services
- Efficient querying and filtering

The standard `log` package outputs unstructured text, incompatible with structured logging pipelines.

## Decision

**Implement a `SlogAdapter` that wraps `slog.Logger` to satisfy the `asynq.Logger` interface.**

Key design choices:

1. **Use slog**: Go 1.21+ standard library structured logger
2. **Stdout output**: JSON logs to stdout instead of stderr
3. **No os.Exit in Fatal**: Log with severity attribute, let the caller handle termination

## Options Considered

### Option A: SlogAdapter (Selected)

**Implementation:**

```go
type SlogAdapter struct {
    logger *slog.Logger
}

func (l *SlogAdapter) Info(args ...any) {
    l.logger.Info(fmt.Sprint(args...))
}

func (l *SlogAdapter) Fatal(args ...any) {
    l.logger.Error(fmt.Sprint(args...), "severity", "fatal")
}
```

**Pros:**

- Uses Go standard library (no external dependencies)
- JSON output to stdout for platform compatibility
- Consistent with application-wide logging strategy
- Fatal doesn't abruptly terminate; allows graceful handling

**Cons:**

- Requires adapter implementation and maintenance
- Slightly different Fatal semantics than expected

### Option B: Asynq Default Logger

- Uses Go's `log.Logger` with stderr output
- No code changes required
- Platform compatibility issues in cloud environments

### Option C: Third-Party Logging Library

- Use zap, zerolog, or logrus with Asynq adapter
- Additional dependency
- May conflict with existing logging choices

## Implementation Notes

### asynq.Logger Interface

```go
type Logger interface {
    Debug(args ...any)
    Info(args ...any)
    Warn(args ...any)
    Error(args ...any)
    Fatal(args ...any)
}
```

All methods accept variadic arguments, which the adapter joins with `fmt.Sprint`.

### Fatal Behavior

The adapter intentionally avoids calling `os.Exit`. Instead, it logs at Error level with a `"severity": "fatal"` attribute. This:

- Allows deferred cleanup to execute
- Enables graceful shutdown handling
- Provides fatal-level visibility in logs without abrupt termination

### Configuration

```go
srv := asynq.NewServer(
    redisOpt,
    asynq.Config{
        Logger: queue.NewSlogAdapter(slog.Default()),
        // other config...
    },
)
```

## Consequences

### Positive

**Platform Compatibility:**

- Logs correctly classified by severity in cloud platforms
- No false error alerts from normal operations
- Reliable log-level filtering

**Structured Logging:**

- JSON format enables machine parsing
- Consistent with application logging
- Easier debugging with structured fields

**Operational Clarity:**

- Clear separation of actual errors vs routine messages
- Improved monitoring dashboard accuracy
- Reduced alert fatigue

### Negative

**Maintenance Overhead:**

- Custom adapter code to maintain
- Must track asynq.Logger interface changes

**Semantic Difference:**

- Fatal doesn't terminate the process
- Callers expecting immediate exit may be surprised

## References

- [Go slog Package Documentation](https://pkg.go.dev/log/slog)
- [Asynq Logger Interface](https://pkg.go.dev/github.com/hibiken/asynq#Logger)
