# Building editcp on macOS

This fork adds a native macOS build. The upstream project only ships Linux binaries.

## Requirements

- macOS 12+ (tested on macOS 15 Sequoia)
- Homebrew
- Go 1.21+
- Qt 5 via Homebrew
- libusb via Homebrew
- Xcode Command Line Tools

```bash
brew install qt@5 libusb
xcode-select --install
```

You also need a checkout of therecipe/qt with the docs submodules and some patches
applied to it. Build and install `qtsetup` and `qtdeploy` from that checkout:

```bash
git clone https://github.com/therecipe/qt ~/go/src/github.com/therecipe/qt
cd ~/go/src/github.com/therecipe/qt
# Apply the go.mod and parser.go patches described in the commit history
go install ./cmd/qtsetup
go install ./cmd/qtdeploy
QT_DIR=/usr/local/opt/qt@5 QT_VERSION=$(qmake -query QT_VERSION) \
  QT_API=5.13.0 qtsetup generate desktop
```

## Build

```bash
git clone https://github.com/campbell-r-e/editcp
cd editcp
make darwin
```

The `.app` bundle lands at `deploy/darwin/editcp.app`.

## What was changed from upstream

### `radio.go`
Removed the `RadioExists()` pre-flight check that opens and immediately closes
the libusb context before `ReadRadio`. On macOS the second `libusb_init` call
fails with "libusb_device reference not released on last exit", causing a crash.
Also added a `return` after `FreeCodeplug()` so that `cp.Valid()` is never
called on a freed codeplug pointer.

### `vendor/github.com/dalefarnsworth-dmr/stdfu/stdfu_unix.go` (patch)
`gousb.NewContext()` panics when called from a Qt C++ callback on the main OS
thread because Qt's NSApplication event loop has already claimed the IOKit/
CFRunLoop resources that libusb needs. The fix runs USB initialisation on a
dedicated goroutine with `runtime.LockOSThread()` and wraps it in `recover()`
so a libusb init failure returns a proper error instead of crashing the process.
The patched file lives in `patches/` and is copied into `vendor/` by
`make darwin` after `go mod vendor` would otherwise overwrite it.

### `Makefile`
Added the `darwin` target which:
- Refreshes the macOS SDK symlink in `~/Library/FakeXcode.app`
- Runs `go mod tidy && go mod vendor`
- Copies Qt 5.13.0 doc index files from the therecipe/qt checkout into the
  vendor tree (required by the minimal binding generator; these are wiped by
  `go mod vendor`)
- Applies the stdfu patch
- Runs `qtdeploy build desktop` with all required env vars

### `~/go/src/github.com/therecipe/qt` (not in this repo)
- `go.mod`: added `go 1.12` directive; added require+replace for docs/5.12.0
  and docs/5.13.0 submodules; replaced stale 2019 `golang.org/x` mirrors that
  caused a segfault in `IoctlGetTermios` on modern macOS.
- `internal/binding/parser/parser.go`: replaced `GoListOptional("{{.Dir}}", ...)` 
  calls for the docs path with `GoQtPkgPath(...)` so the binding generator works
  correctly under `-mod=vendor`.
