---
title: go-log
---

`go-log` is the gomatic ecosystem's **CLI-agnostic structured-logging configuration layer** for Go. It is a thin layer over the standard library's [`log/slog`](https://pkg.go.dev/log/slog): textual [`Level`](https://pkg.go.dev/github.com/gomatic/go-log#Level) and [`Format`](https://pkg.go.dev/github.com/gomatic/go-log#Format) value types that bind cleanly from a consumer's flags, gathered into a [`LoggerConfig`](https://pkg.go.dev/github.com/gomatic/go-log#LoggerConfig) whose [`NewLogger`](https://pkg.go.dev/github.com/gomatic/go-log#LoggerConfig.NewLogger) method builds a `*slog.Logger` over any `io.Writer`. The package knows nothing about command-line frameworks — wiring these types to flags lives entirely in the consumer.

- **Source:** [`gomatic/go-log`](https://github.com/gomatic/go-log)
- **API reference:** [pkg.go.dev/github.com/gomatic/go-log](https://pkg.go.dev/github.com/gomatic/go-log)

## Install

```sh
go get github.com/gomatic/go-log
```

## Why a configuration layer

Every CLI needs to turn a `--log-level` and `--log-format` flag into a configured `*slog.Logger`, and every CLI tends to reinvent that plumbing slightly differently. `go-log` owns the **mechanism only**: it parses textual levels and formats into the right `slog` handler, with sensible defaults, and leaves flag declaration to the consumer. The `Level` and `Format` types are plain string newtypes, so binding them to a flag is just assigning a string.

## Usage

### Build a logger

```go
package main

import (
	"os"

	log "github.com/gomatic/go-log"
)

func main() {
	logger := log.LoggerConfig{Level: "info", Format: log.FormatJSON}.NewLogger(os.Stderr)
	logger.Info("ready", "addr", ":8080")
}
```

`LoggerConfig` is a value, not a pointer — copy it freely. `NewLogger` accepts any `io.Writer`, so the same config drives a logger to `os.Stderr`, a file, or a test buffer.

### Levels

`Level` accepts `debug`, `info`, `warn`, or `error`, and **defaults to `info`** when the value is empty or unrecognized:

```go
log.LoggerConfig{Level: "debug"}.NewLogger(os.Stderr) // debug and above
log.LoggerConfig{Level: ""}.NewLogger(os.Stderr)      // empty → info
log.LoggerConfig{Level: "bogus"}.NewLogger(os.Stderr) // invalid → info
```

### Formats

`Format` selects the encoding via the two exported constants, and **defaults to text** for any unknown value:

```go
log.LoggerConfig{Format: log.FormatText}.NewLogger(os.Stderr) // text handler
log.LoggerConfig{Format: log.FormatJSON}.NewLogger(os.Stderr) // json handler
log.LoggerConfig{Format: "yaml"}.NewLogger(os.Stderr)         // unknown → text
```

### Binding to flags

The point of the string newtypes is that a consumer binds them straight from its own flag layer — `go-log` never imports a CLI framework:

```go
cfg := log.LoggerConfig{
	Level:  log.Level(*levelFlag),   // e.g. "warn"
	Format: log.Format(*formatFlag), // e.g. "json"
}
logger := cfg.NewLogger(os.Stderr)
```

## Design

- **Two textual value types** — [`Level`](https://pkg.go.dev/github.com/gomatic/go-log#Level) and [`Format`](https://pkg.go.dev/github.com/gomatic/go-log#Format) are `string` newtypes, so they are safe to copy, compare, and assign directly from a flag's string value.
- **Forgiving defaults** — an empty or invalid `Level` resolves to `info`; an unknown `Format` resolves to the text handler. A misconfigured flag degrades to a working logger rather than failing.
- **Any writer** — `NewLogger` takes an `io.Writer`, decoupling the logger from `os.Stderr` and making it trivial to capture output in tests.
- **CLI-agnostic** — the package depends only on the standard library (`io`, `log/slog`) and knows nothing about command-line frameworks; flag binding lives in the consumer.

## Who uses it

Every gomatic Go CLI shares this logging setup rather than re-deriving it: [`renderizer`](https://github.com/gomatic/renderizer), [`template.cli`](https://github.com/gomatic/template.cli), and the other [`gomatic/go-*`](https://github.com/orgs/gomatic/repositories?q=go-) libraries.
