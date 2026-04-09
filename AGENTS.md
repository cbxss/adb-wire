# adb-wire — AGENTS.md

Async ADB wire protocol client over TCP. Talks directly to the ADB server using the wire protocol — no spawning `adb` subprocesses. Published on [crates.io](https://crates.io/crates/adb-wire).

## Structure

| File | Role |
|------|------|
| `src/lib.rs` | `AdbWire` client struct, host commands, install, forward/reverse, connect/disconnect, wait_for_device/boot |
| `src/wire.rs` | Hex-length framed protocol: `send()`, `read_okay()`, `read_hex_len()` |
| `src/shell.rs` | Shell v2 protocol: `ShellOutput` (stdout/stderr/exit), `ShellStream` (AsyncRead) |
| `src/sync.rs` | Sync protocol: stat (v1 + v2), pull, push, directory listing (LIST/DENT) |
| `src/error.rs` | `Error` enum (Io, Adb, Protocol) |

## Public API

```rust
let adb = AdbWire::new("emulator-5554")
    .server(addr)           // default: 127.0.0.1:5037
    .connect_timeout(dur)   // default: 2s
    .read_timeout(dur);     // default: 30s

// Shell
let result = adb.shell("getprop ro.build.version.sdk").await?;
let stream = adb.shell_stream("logcat").await?;
adb.shell_detach("input tap 500 500").await?;

// Files
adb.push_file("local.txt", "/sdcard/remote.txt").await?;
adb.pull_file("/sdcard/remote.txt", "local.txt").await?;
let stat = adb.stat("/sdcard/file.txt").await?;  // STAT2 with v1 fallback
let entries = adb.list_dir("/sdcard").await?;

// Install
adb.install("app.apk").await?;

// Port forwarding
adb.forward("tcp:8080", "tcp:8080").await?;
adb.reverse("tcp:3000", "tcp:3000").await?;

// Device lifecycle
adb.wait_for_device().await?;
adb.wait_for_boot().await?;

// Host commands (no device needed)
let devices = list_devices(DEFAULT_SERVER).await?;
connect_device(DEFAULT_SERVER, "192.168.1.100:5555").await?;
disconnect_device(DEFAULT_SERVER, "192.168.1.100:5555").await?;
```

## Protocol Details

- **Wire protocol**: `[4-char hex length][payload]` framing over TCP to ADB server (port 5037)
- **Shell v2**: 5-byte packet header `[id:u8][len:u32le]`, stream IDs 0=stdin, 1=stdout, 2=stderr, 3=exit
- **Sync protocol**: 8-byte headers `[tag:4][len:u32le]`, tags: STAT/STA2/RECV/SEND/DATA/DONE/LIST/DENT
- **STAT2**: Extended stat with u64 size/mtime, dev/ino/nlink/uid/gid/atime/ctime. Falls back to v1.

## Dependencies

Only `tokio` (net, rt, io-util, time, fs, macros). Zero other dependencies.

## Testing

```bash
cargo test                          # unit tests (28)
cargo test -- --ignored             # integration tests (13, need real device)
ADB_SERIAL=device-id cargo test -- --ignored  # specific device
```
