# Cloudflared Embedded API Guide

## Overview

This fork provides two ways to embed cloudflared tunnels:

1. **Go module API** (`package cloudflared`) — import `cmd/cloudflared` directly from Go code
2. **CGO/DLL exports** (`//go:build cgo`) — C-callable shared library for Python, C, etc.

Both share the same codebase in `cmd/cloudflared/`.

---

## Branch & Remote Structure

| Remote     | URL                                         | Role                                                |
|------------|---------------------------------------------|-----------------------------------------------------|
| `origin`   | git@github.com:gt4o4/cloudflared.git        | This fork (push here)                               |
| `upstream` | https://github.com/cloudflare/cloudflared   | Official source; its release tags are the sync points |

The fork's GitHub parent is QudsLab/Cloudflared, which is why GitHub's compare
view and `gh api compare` fail against cloudflare/cloudflared; nothing is taken
from that parent.

| Branch  | Purpose                                                                                     |
|---------|---------------------------------------------------------------------------------------------|
| `main`  | Upstream's newest release tag + two commits: the `unix+ssh:` ingress scheme and the inits/service tweaks. Pinned by the nix-configs fleet (`common/overlays/cloudflared.nix`); rebased onto each release |
| `cflib` | The embedding API on top of `main` (the default branch): `package cloudflared` in `cmd/cloudflared/`, the CGO exports (`lib_bin_exports.go`), the Python wrapper, `updates/` and the DLL workflow. Rebased onto `main` after each release; nothing in the fleet consumes it |

Old heads stay reachable as `pin-<date>` (`main`) and `pin-<date>-cflib` tags.

## Syncing Procedure

Upstream is taken by rebase, never by merge: `main` onto the release tag, then
`cflib` onto `main`. Each step force-pushes with `--force-with-lease` after
tagging the old head.

```bash
git clone git@github.com:gt4o4/cloudflared.git && cd cloudflared
git remote add upstream https://github.com/cloudflare/cloudflared.git
git fetch upstream --tags
git checkout main              # the clone starts on cflib, the default branch

# 1. main onto the newest release tag (two commits replay)
OLD_MAIN=$(git rev-parse main)
git tag -m "pin before <date> rebase onto <tag>" pin-<date> "$OLD_MAIN"
git rebase <tag>                                # e.g. 2026.10.3
grep -n 'unix+ssh' ingress/ingress.go           # the patch survived
gofmt -e -l $(git diff --name-only <tag> | grep '\.go$') && go build ./...
git push origin pin-<date>
git push --force-with-lease=main:"$OLD_MAIN" origin main

# 2. cflib onto the new main: replay ONLY cflib's own commits
OLD_CFLIB=$(git rev-parse cflib)
git tag -m "pin before <date> rebase of cflib onto main" pin-<date>-cflib "$OLD_CFLIB"
git rebase --onto main "$OLD_MAIN" cflib -X theirs
#   --onto ... $OLD_MAIN skips the old copies of main's two commits (a plain
#   `git rebase main` replays them too and conflicts on them); -X theirs keeps
#   cflib's hunk wherever both sides touched a file.

# 3. Review the files both sides touched: -X theirs is wrong for code
comm -12 <(git diff --name-only "$OLD_MAIN" main | sort) \
         <(git diff --name-only "$OLD_MAIN" "$OLD_CFLIB" | sort)
#   README.md, AGENTS.md, .github/workflows/check.yaml: cflib's own rewrites,
#     keep ours (the README then lacks upstream's newest sections; port by hand
#     what matters).
#   cmd/cloudflared/tunnel/quick_tunnel.go: keep UPSTREAM's startup code and
#     fire OnURLReady(quickTunnelURL) just before cliutil.LogTable(...).
#   .gitignore: upstream's lines (/vendor) plus ours.
#   Any cmd/cloudflared/*.go: keep `package cloudflared`,
#     `runApp(..., args []string)`, `wireOnURLReady()` and main.go's API functions.

# 4. Re-sync the mirror that replace.py applies (pairs: updates/json/file_mapping.json)
for f in main.go lib_bin_exports.go generic_service.go linux_service.go \
         macos_service.go windows_service.go; do
  cp "cmd/cloudflared/$f" "updates/modified_files/$f"
done
cp cmd/cloudflared/tunnel/quick_tunnel.go updates/modified_files/quick_tunnel.go
python3 updates/replace.py . --verify
git add -A && git commit -m "Re-sync updates/modified_files with cmd/cloudflared after rebase onto main"

# 5. Verify (the branch's checks; not `go build ./...`: updates/modified_files holds two packages)
gofmt -e -l $(git diff --name-only main | grep '\.go$')
go vet ./cmd/cloudflared/...
go build ./cmd/cloudflared/...
git diff --stat main                            # cflib's own delta, ~45 files

# 6. Push
git push origin pin-<date>-cflib
git push --force-with-lease=cflib:"$OLD_CFLIB" origin cflib
```

Last done 2026-10-21: `main` onto 2026.10.3 (`fd503fea`), `cflib` `adfe4006` →
`2c7d180a` (its first sync since the August `main`; quick_tunnel.go and
.gitignore needed the hand fixes above).

---

## Go Module API

### Consumer usage

```go
import "github.com/cloudflare/cloudflared/cmd/cloudflared"

cloudflared.Init()
cloudflared.StartQuickTunnel(port)

// Poll for readiness
for !cloudflared.IsReady() {
    time.Sleep(time.Second)
}
url := cloudflared.GetURL()

// When done
cloudflared.Stop()
```

### Functions

| Function                    | Description                                    |
|-----------------------------|------------------------------------------------|
| `Init()`                    | Reset state, create shutdown channel            |
| `Stop()`                    | Close shutdown channel, signal exit             |
| `SetURL(url string)`        | Record tunnel URL (called by OnURLReady)        |
| `GetURL() string`           | Return tunnel URL (empty until ready)           |
| `IsReady() bool`            | Whether tunnel URL is known                     |
| `Run(args []string)`        | Run cloudflared with args (blocks)              |
| `StartQuickTunnel(port int)`| Launch quick tunnel in goroutine                |
| `RunNamed(token string)`    | Launch named tunnel in goroutine                |

---

## CGO/DLL API

Clean API to get tunnel status and URL directly without parsing output or logs.

## Quick Start

```python
from t import load_dll, init_tunnel, start_tunnel, get_tunnel_url

# Load DLL (silent mode by default)
load_dll("path/to/cloudflared.dll", silent=True, debug=False)
init_tunnel()

# Start tunnel and wait for URL
if start_tunnel(port=5000, timeout=30, debug=False):
    url = get_tunnel_url()
    print(f"Tunnel URL: {url}")  # Only this prints!
```

## New DLL Functions

### `CloudflaredGetTunnelStatus()`
Returns the current tunnel status as an integer:
- `0` = Not started
- `1` = Starting (initializing connection)
- `2` = Ready (tunnel established with URL)

**C Signature:**
```c
int CloudflaredGetTunnelStatus();
```

**Python (ctypes):**
```python
lib.CloudflaredGetTunnelStatus.restype = ctypes.c_int
status = lib.CloudflaredGetTunnelStatus()
```

### `CloudflaredGetTunnelURL()`
Returns the tunnel URL as a C string (or NULL if not ready).

**C Signature:**
```c
char* CloudflaredGetTunnelURL();
```

**Python (ctypes):**
```python
lib.CloudflaredGetTunnelURL.restype = ctypes.c_char_p
c_url = lib.CloudflaredGetTunnelURL()
if c_url:
    url = c_url.decode('utf-8')
    lib.CloudflaredFreeString(c_url)  # Important: free memory!
```

## Usage Pattern

```python
import ctypes
import time

# Load DLL
lib = ctypes.CDLL("cloudflared.dll")

# Setup function signatures
lib.CloudflaredInit.restype = ctypes.c_int
lib.CloudflaredRun.argtypes = [ctypes.c_char_p]
lib.CloudflaredRun.restype = ctypes.c_int
lib.CloudflaredGetTunnelStatus.restype = ctypes.c_int
lib.CloudflaredGetTunnelURL.restype = ctypes.c_char_p
lib.CloudflaredFreeString.argtypes = [ctypes.c_char_p]

# Initialize
lib.CloudflaredInit()

# Start tunnel in background thread
import threading
def run():
    lib.CloudflaredRun(b"cloudflared tunnel --url http://localhost:5000")
threading.Thread(target=run, daemon=True).start()

# Poll until ready
while True:
    status = lib.CloudflaredGetTunnelStatus()
    
    if status == 2:  # Ready
        c_url = lib.CloudflaredGetTunnelURL()
        url = c_url.decode('utf-8')
        lib.CloudflaredFreeString(c_url)
        print(f"Tunnel ready: {url}")
        break
    elif status == 0:  # Failed
        print("Tunnel failed to start")
        break
    
    time.sleep(0.5)
```

## Benefits

✓ **No output parsing** - Direct API calls  
✓ **Thread-safe** - Uses mutex internally  
✓ **Clean interface** - Simple status codes  
✓ **Memory safe** - Proper string allocation/deallocation  
✓ **Real-time** - Poll anytime for current status  

## Modified Files

1. **lib_bin_exports.go**
   - Added `globalTunnelURL` and `globalTunnelReady` globals
   - Added `CloudflaredGetTunnelURL()` export
   - Added `CloudflaredGetTunnelStatus()` export
   - Added `CloudflaredSetTunnelURL()` internal function

2. **quick_tunnel.go**
   - Calls `CloudflaredSetTunnelURL()` when tunnel is established
   - Sets global state for DLL API access

## Memory Management

⚠️ **Important**: Always call `CloudflaredFreeString()` after getting the URL to avoid memory leaks:

```python
c_url = lib.CloudflaredGetTunnelURL()
if c_url:
    url = c_url.decode('utf-8')
    lib.CloudflaredFreeString(c_url)  # Free the C string!
```

## Example Class

See `python_example.py` for a complete `CloudflaredTunnel` class that wraps all functionality.
