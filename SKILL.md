---
name: termux
description: Hard-learned, non-obvious knowledge for running agents and dev tooling on Termux/Android. Load this skill when working on an Android/Termux device — builds, paths, processes, permissions, or anything that mysteriously fails.
user-invocable: false
---

# Termux: Hard-Learned Hints

Non-obvious facts only. If it's in the docs or works like Linux, it's not here.

## Paths & Filesystem

- `/tmp` does not exist and is not writable — use `$HOME/tmp` or `$TMPDIR`.
- There is no `/bin`, `/usr`, `/etc` — everything lives under `$PREFIX` (`/data/data/com.termux/files/usr`). Absolute paths from Linux convention silently fail.
- Shebangs like `#!/usr/bin/env` break — use `$PREFIX/bin/env` or run `termux-fix-shebang` on installed scripts.
- npm-installed binaries may lack the exec bit and/or carry CRLF shebangs (`env: 'node\r'`) — check both.
- Shared storage (`~/storage/*`) is FUSE: **no symlinks**, and permissions behave differently from `$HOME`.
- Other apps' private data is unreachable (Android sandbox), no Bluetooth access. Device features (camera, SMS, GPS, TTS, clipboard, sensors…) only via the `termux-api` CLI + Termux:API add-on app.
- Android's DUMP permission is not granted — anything reading system state via `dumpsys` returns nothing useful.

## Native Builds

- node-gyp / node-pty builds fail without `python`, `clang`, `make` installed first.
- `Undefined variable android_ndk_path in binding.gyp` → override `GYP_DEFINES` with a dummy value rather than installing the NDK.
- After `npm install`, re-check exec bits and shebang endings on bin scripts — updates can undo fixes.

## Processes & Lifecycle

- Background processes die with the spawning session unless detached (`setsid` + `nohup`); children receive SIGTERM when a session breaks.
- Android freezes/kills background apps under memory pressure — use a wake lock for anything long-running.
- Boot-time autostart requires the Termux:Boot add-on, opened once after install; even then it depends on matching app sources (see below).
- UNIX sockets in `$HOME` may fail with `EACCES` on Android — treat socket-based control interfaces as best-effort.
- RAM and swap are small and swap fills silently — "random crashes" are often OOM kills. Check memory pressure before blaming code.

## OAuth & Credentials

- `.netrc` is whitespace-delimited — passwords with spaces break it (quotes don't help); store secrets in a dedicated 600-mode file instead.
- OAuth callbacks to `localhost` work on-device: the phone's own browser can hit a local callback server, so headless-style flows run without a second machine.

## Debugging Pitfalls

- `pkill -f` / `pgrep -f` patterns match your own `bash -c` command line — bracket a character (`[-]`) or kill by exact pid.
- `getent`, `nslookup`, and similar core Linux tools are absent; `curl` and `ping` work fine.
- When a process "is running but not responding", first check for orphaned processes still holding the port before restarting.
- `rclone config create` can hang indefinitely on Termux — writing `rclone.conf` directly (with a JSON token blob) is equivalent and reliable.
- A healthy pidfile can point at a dead pid while an orphan holds the resource — trust the port/socket, not the pidfile.

## Add-ons & App Sources

- Termux add-ons (API, Boot, Widget) must come from the **same source** as the Termux app (signing keys differ between F-Droid, GitHub, and Play Store) — mixing breaks add-ons.
- Some add-ons (notably Termux:API) are not on the Play Store at all; F-Droid/GitHub is the complete source.
- Since 2026, Android developer verification blocks installing unverified APKs on new devices; the escape hatch ("advanced flow" in developer settings) carries a 24-hour wait. Plan sideload-dependent setups accordingly.
