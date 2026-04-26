# macOS Build Notes

This document covers everything required to build editcp natively on macOS,
including every bug hit along the way and the exact fixes applied.  The
upstream project ships only Linux binaries and its build instructions assume
Linux with Docker.  None of that works on macOS without substantial surgery.

---

## Quick start — just run the app

Download `editcp-1.0.31-macos.dmg` from the
[Releases](https://github.com/campbell-r-e/editcp/releases) page, open it,
drag **editcp.app** to Applications.

First launch: right-click → Open to bypass Gatekeeper (the app is unsigned).

To read/write a radio over USB you also need libusb:
```bash
brew install libusb
```

Put the MD-380 in DFU mode (hold PTT, press Power) before clicking
**Radio → Read codeplug from radio**.

---

## Building from source

### System requirements

- macOS 12 or later (tested on macOS 15 Sequoia, Intel x86_64)
- Xcode Command Line Tools: `xcode-select --install`
- [Homebrew](https://brew.sh)
- Go 1.21+: `brew install go`
- Qt 5 via Homebrew: `brew install qt@5`
- libusb via Homebrew: `brew install libusb`

### Step 1 — patch and install therecipe/qt tools

`qtdeploy` and `qtsetup` are build tools from the (now-archived)
`therecipe/qt` Go-Qt binding library.  They must be built from source because
the archived module has several incompatibilities with modern Go and macOS.

```bash
git clone https://github.com/therecipe/qt ~/go/src/github.com/therecipe/qt
cd ~/go/src/github.com/therecipe/qt
```

Apply the following patches by hand (the repo is archived so there is no
upstream to send them to).

#### 1a. `go.mod` — fix module graph for modern Go and macOS

The original `go.mod` is missing a `go` directive and references ancient
`golang.org/x` mirrors (2019 vintage) that cause a segfault in
`IoctlGetTermios` on modern macOS kernels.  It also doesn't declare the two
Qt-docs submodules that the binding generator needs.

Replace the whole file with:

```
module github.com/therecipe/qt

go 1.12

require (
    github.com/therecipe/qt/internal/binding/files/docs/5.12.0 v0.0.0-00010101000000-000000000000
    github.com/therecipe/qt/internal/binding/files/docs/5.13.0 v0.0.0-00010101000000-000000000000
    golang.org/x/crypto v0.37.0
    golang.org/x/sys v0.32.0
    golang.org/x/tools v0.32.0
)

replace (
    github.com/therecipe/qt/internal/binding/files/docs/5.12.0 => ./internal/binding/files/docs/5.12.0
    github.com/therecipe/qt/internal/binding/files/docs/5.13.0 => ./internal/binding/files/docs/5.13.0
)
```

Then run `go mod tidy` to regenerate `go.sum`.

**Why the old versions segfault:** the 2019 `golang.org/x/sys` version calls
`IoctlGetTermios` with a struct layout that changed in a later kernel ABI
revision.  On macOS 15 this manifests as a SIGSEGV at startup.

#### 1b. `internal/binding/parser/parser.go` — fix docs path under `-mod=vendor`

The binding generator locates Qt XML doc index files using `go list` to find
the `docs/5.x.x` sub-module directory.  When the consumer project builds with
`-mod=vendor` (which Go enforces when a `vendor/` directory is present), `go
list` refuses to read modules outside the vendor tree and returns an empty
path.  This causes every Qt class/module to load as empty, producing `EOF`
errors during `qtsetup generate` and ultimately an app that silently drops all
Qt bindings.

Find every call site in `parser.go` that looks like:

```go
utils.LoadOptional(filepath.Join(strings.TrimSpace(utils.GoListOptional(
    "{{.Dir}}",
    "github.com/therecipe/qt/internal/binding/files/docs/"+utils.QT_API(utils.QT_VERSION()),
    "-find", "get doc dir")),
    fmt.Sprintf("qt%v.index", strings.ToLower(m))))
```

and replace each one with:

```go
utils.LoadOptional(filepath.Join(
    utils.GoQtPkgPath("internal/binding/files/docs/"+utils.QT_API(utils.QT_VERSION())),
    fmt.Sprintf("qt%v.index", strings.ToLower(m))))
```

`GoQtPkgPath` resolves the path relative to the therecipe/qt source tree,
which is always correct regardless of build mode.  There are five call sites
(WebKit, Felgo, Homebrew/MXE/NIX, PKG_CONFIG, and the default case) — all
must be changed.

#### 1c. Build and install the tools

```bash
cd ~/go/src/github.com/therecipe/qt
go install ./cmd/qtsetup ./cmd/qtdeploy
```

#### 1d. Create a fake Xcode directory

`qtdeploy` requires `XCODE_DIR` to point to an `Xcode.app`-shaped directory
containing the macOS SDK.  Full Xcode is not required; the Command Line Tools
SDK is enough.  Create a permanent stub that survives reboots:

```bash
SDK_PATH=$(xcrun --show-sdk-path)
SDK_VER=$(xcrun --show-sdk-version)
FAKE="$HOME/Library/FakeXcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs"
mkdir -p "$FAKE"
ln -s "$SDK_PATH" "$FAKE/MacOSX${SDK_VER}.sdk"
```

The `make darwin` target refreshes this symlink automatically on each build.

#### 1e. Run `qtsetup generate`

```bash
QT_DIR=/usr/local/opt/qt@5 \
QT_VERSION=$(/usr/local/opt/qt@5/bin/qmake -query QT_VERSION) \
QT_API=5.13.0 \
XCODE_DIR="$HOME/Library/FakeXcode.app" \
qtsetup generate desktop
```

**`QT_API=5.13.0` is required.**  Without it the generator defaults to the
full Qt version string (e.g. `5.15.18`) and tries to load
`docs/5.15.18/qt*.index` files that don't exist, producing `EOF` on every
module and generating an empty binding set.  The docs submodules only ship
`5.12.0` and `5.13.0`; `5.13.0` is the closest match for Qt 5.15.

### Step 2 — build editcp

```bash
git clone https://github.com/campbell-r-e/editcp
cd editcp
make darwin
```

The `.app` bundle lands at `deploy/darwin/editcp.app`.

`make open` launches it immediately.  `make install-macos` copies it to
`/Applications`.

---

## Changes made to editcp for macOS

### `radio.go` — remove double libusb init

**Symptom:** clicking *Radio → Read codeplug from radio* crashed with:

```
panic: runtime error: invalid memory address or nil pointer dereference
github.com/dalefarnsworth-dmr/codeplug.(*Field).listNames(...)
```

**Root cause (found via `LIBUSB_DEBUG=4`):**

The menu callback calls `codeplug.RadioExists()` as a pre-flight check,
which internally calls `dfu.New()` → `stdfu.New()` → `gousb.NewContext()` →
`libusb_init()`.  It opens the device, verifies it exists, then immediately
calls `dfu.Close()` → `libusb_exit()`.

libusb on macOS leaves a dangling device reference at exit (a known gousb
v2.1.0 issue where `OpenDeviceWithVIDPID` increments the device ref count via
`libusb_get_device_list` but the corresponding unref is not called before
`libusb_exit`).  This sets an internal libusb flag.

1.6 seconds later the same callback calls `cp.ReadRadio()`, which calls
`dfu.New()` again.  `libusb_init()` sees the leftover ref and refuses:

```
libusb: error [darwin_first_time_init] libusb_device reference not released
on last exit. will not continue
```

`gousb.NewContext()` panics.  Our `recover()` catches it and returns an
error.  `ReadRadio` propagates the error.  `radio.go` then calls
`edt.FreeCodeplug()` — and then immediately calls `cp.Valid()` on the now-
freed codeplug pointer.  Nil pointer dereference, crash.

**Fix:**

1. Remove the `codeplug.RadioExists()` pre-check entirely.  The radio check
   is redundant — `ReadRadio` returns a clear error if the device is absent.
   Eliminating it removes the first `libusb_init`/`libusb_exit` cycle, so
   there is no leftover reference when `ReadRadio` initialises libusb.

2. Add `return` after `edt.FreeCodeplug()` so `cp.Valid()` is never reached
   when `ReadRadio` returns an error.

### `patches/github.com/dalefarnsworth-dmr/stdfu/stdfu_unix.go` — libusb on Qt main thread

**Symptom (earlier crash before the RadioExists fix):**

```
panic: libusb: unknown error [code -99]

goroutine 1 [running, locked to thread]:
github.com/google/gousb.newContextWithImpl(...)
    .../gousb@v2.1.0+incompatible/usb.go:146
github.com/dalefarnsworth-dmr/stdfu.New()
    .../stdfu_unix.go:54
```

**Root cause:**

`gousb.NewContext()` calls `libusb_init()`.  On macOS, libusb's darwin
backend initialises IOKit notification infrastructure and attaches a source to
the calling thread's CFRunLoop.  When called from goroutine 1 (the Go runtime
main goroutine), that goroutine is pinned to the process's main OS thread.

By the time the user clicks *Read codeplug from radio*, Qt's `QApplication`
has long since called `NSApplicationMain`, which sets up Cocoa's event loop
on that same OS thread and owns its CFRunLoop.  libusb attempts to register
its IOKit notification source on a run loop that Qt already controls; the
operation fails and libusb returns `LIBUSB_ERROR_OTHER` (-99).  `gousb`
converts that return value to a `panic`.

A standalone Go program calling the same code works fine because the main
thread's run loop is not owned by anything else.

**Fix:**

Move all libusb/gousb initialisation to a dedicated goroutine that is locked
to a *new* OS thread (not the Qt main thread) using `runtime.LockOSThread()`.
Wrap the initialisation in `recover()` to convert any panic into a returned
error.  The goroutine communicates back through a channel.

```go
func New() (*StDfu, error) {
    type result struct {
        stDfu *StDfu
        err   error
    }
    ch := make(chan result, 1)

    go func() {
        runtime.LockOSThread()
        defer runtime.UnlockOSThread()

        var r result
        func() {
            defer func() {
                if rec := recover(); rec != nil {
                    r.err = fmt.Errorf("USB initialization failed: %v", rec)
                }
            }()

            ctx := gousb.NewContext()
            stDfu := &StDfu{ctx: ctx}
            // ... open device, claim interface ...
            r.stDfu = stDfu
        }()
        ch <- r
    }()

    r := <-ch
    return r.stDfu, r.err
}
```

The patched file lives at `patches/github.com/dalefarnsworth-dmr/stdfu/stdfu_unix.go`
and is copied into `vendor/` by `make darwin` after `go mod vendor` (which
would otherwise restore the original file).

### `Makefile` — `darwin` target

The `darwin` target encodes all required environment variables so the build
is fully reproducible with a single `make darwin`:

```makefile
QT5_DIR    = /usr/local/opt/qt@5
FAKE_XCODE = $(HOME)/Library/FakeXcode.app
GOBIN      = $(HOME)/go/bin

darwin: setup-macos-sdk FORCE
    go mod tidy
    go mod vendor
    # Qt doc indexes are wiped by go mod vendor; restore them from the
    # therecipe/qt source checkout.
    @mkdir -p vendor/github.com/therecipe/qt/internal/binding/files/docs
    @cp -r $(GOPATH)/src/github.com/therecipe/qt/internal/binding/files/docs/5.13.0 \
        vendor/github.com/therecipe/qt/internal/binding/files/docs/
    # Apply the stdfu macOS patch (also wiped by go mod vendor).
    @cp patches/github.com/dalefarnsworth-dmr/stdfu/stdfu_unix.go \
        vendor/github.com/dalefarnsworth-dmr/stdfu/stdfu_unix.go
    PATH="$(QT5_DIR)/bin:$(GOBIN):$$PATH" \
    QT_DIR="$(QT5_DIR)" \
    QT_VERSION=$$($(QT5_DIR)/bin/qmake -query QT_VERSION) \
    QT_API=5.13.0 \
    XCODE_DIR="$(FAKE_XCODE)" \
    QT_NOT_CACHED=true \
    qtdeploy build desktop
```

`QT_NOT_CACHED=true` forces regeneration of the minimal Qt bindings on every
build.  Without it, a stale Go build cache can serve a previous (pre-patch)
version of the generated binding code, producing `undefined: NewQByteArrayFromPointer`
and similar link errors.

---

## Errors encountered and resolved during the port

This section is a complete log of every non-obvious problem hit during the
port, in the order they appeared.

### E1 — `GO111MODULE=off` not supported

```
go: GO111MODULE=off is no longer supported
```

The therecipe/qt build scripts assume GOPATH mode.  Go 1.21+ removed
`GO111MODULE=off`.  Resolution: run all therecipe/qt tool builds in modules
mode from within the cloned repo directory.

### E2 — `go.sum` mismatch / missing module

The docs submodules (`docs/5.12.0`, `docs/5.13.0`) are not referenced in the
original `go.mod` so `go mod tidy` deleted their entries from `go.sum`.
Resolution: add explicit `require` + `replace` directives for both submodules
(see patch 1a above).

### E3 — segfault at startup (`IoctlGetTermios`)

```
SIGSEGV: segmentation violation
runtime.throw2(...)
golang.org/x/sys/unix.IoctlGetTermios(...)
```

The `golang.org/x/sys` version pinned in `go.mod` (a 2019 pre-release) used a
`Termios` struct layout that diverged from the macOS 15 kernel ABI.
Resolution: replace all stale `golang.org/x` pins with current releases
(sys v0.32.0, tools v0.32.0, crypto v0.37.0).

### E4 — `EOF` during `qtsetup generate` (Core, Widgets, …)

```
Error parsing module Core: EOF
Error parsing module Widgets: EOF
```

The binding generator opened `docs/5.15.18/qtcore.index` — which doesn't
exist because `QT_VERSION` returned `5.15.18` and the generator appended it
directly to the docs path.  Resolution: set `QT_API=5.13.0` explicitly so
the generator uses the `5.13.0` doc index files that are actually present in
the repository.

### E5 — `undefined: NewQByteArrayFromPointer` (link error)

```
vendor/github.com/therecipe/qt/core/core-minimal.go:XXX:
    undefined: NewQByteArrayFromPointer
```

The Go build cache served a pre-generation version of the minimal binding
files.  Qt's `qtdeploy` had generated the correct files but `go build` cached
the old empty stubs.  Resolution: set `QT_NOT_CACHED=true` to force
regeneration; alternatively `go clean -cache`.

### E6 — `failed to find XCODE_DIR`

```
qtdeploy: failed to find XCODE_DIR
```

`qtdeploy` looks for a `.app` bundle with a specific internal directory
structure for the macOS SDK.  Full Xcode is not installed; only the Command
Line Tools are present.  Resolution: create `~/Library/FakeXcode.app` (a
permanent location, not `/tmp` which doesn't survive reboots) containing a
symlink to the CLT SDK (see step 1d above).

### E7 — `EOF` during `qtsetup generate` under `-mod=vendor`

After adding the vendor directory and enabling `-mod=vendor` (which Go 1.14+
does automatically when `vendor/` is present), `qtsetup generate` produced
`EOF` on every module again.  This was different from E4.

Root cause: the parser called `go list -find github.com/therecipe/qt/internal/binding/files/docs/5.13.0`
to locate the docs directory.  In vendor mode, `go list` restricts itself to
the consumer's vendor tree and finds nothing (the docs submodule lives inside
the therecipe/qt checkout, not inside editcp's vendor).  The generator
received an empty path and opened `/qt5.13.0.index` (non-existent), getting
`EOF`.

Resolution: patch `internal/binding/parser/parser.go` to use `GoQtPkgPath`
instead of `GoListOptional` for the docs path (see patch 1b above).

### E8 — `panic: libusb: unknown error [code -99]` (first crash)

Covered in detail under **Changes made — stdfu patch** above.  Short version:
libusb IOKit init fails when called from Qt's main OS thread.  Fixed by
running `gousb.NewContext()` in a goroutine with `runtime.LockOSThread()`.

### E9 — `panic: runtime error: invalid memory address or nil pointer dereference` (second crash)

Covered in detail under **Changes made — radio.go** above.  Short version:
`RadioExists()` opened and closed libusb leaving a dangling device reference;
the subsequent `ReadRadio()` call failed to re-init libusb; the error path
then called `cp.Valid()` on a freed codeplug.  Fixed by removing
`RadioExists()` and adding `return` after `FreeCodeplug()`.

---

## Runtime usage

### Reading a codeplug from an MD-380

1. Connect the radio via USB.
2. Put the radio in DFU mode: hold **PTT**, then press **Power**.
   The screen will show the bootloader version and the device will enumerate
   as `0x0483:0xdf11` (STMicroelectronics DFU).
3. Open editcp.
4. **Radio → Read codeplug from radio**.
5. A progress dialog shows the read proceeding (~620 blocks × 1 KB).
6. Save as `.rdt`.

### Writing to an OpenGD77 radio (e.g. Baofeng DM-1701)

editcp does not support OpenGD77 natively.  Use the
[OpenGD77 CPS](https://github.com/rogerclarkmelbourne/OpenGD77) to import the
`.rdt` file (or export from editcp to the text/CSV format and re-import).

---

## File inventory

| Path | Description |
|------|-------------|
| `Makefile` | Build system; `make darwin` is the macOS target |
| `radio.go` | Modified: removed `RadioExists()` pre-check, added `return` after `FreeCodeplug()` |
| `patches/github.com/dalefarnsworth-dmr/stdfu/stdfu_unix.go` | Patched stdfu: libusb init on dedicated goroutine |
| `MACOS_BUILD.md` | This file |
| `deploy/darwin/editcp.app` | Built app bundle (not committed, produced by `make darwin`) |
| `deploy/editcp-1.0.31-macos.dmg` | Distributable DMG (not committed, attached to GitHub Release) |

Files **not** in this repo but required to build:

| Location | Description |
|----------|-------------|
| `~/go/src/github.com/therecipe/qt/` | Patched therecipe/qt checkout |
| `~/Library/FakeXcode.app/` | Fake Xcode SDK stub |
