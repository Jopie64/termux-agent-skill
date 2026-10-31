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
- The `~/storage/*` shortcuts are themselves symlinks (to `/storage/emulated/0/...`): `ls` works but `find` doesn't follow symlink path args — it silently returns 0 results. Use `find -L ~/storage/...` (or the real `/storage/emulated/0` path). Symptom: a find that works on `~/storage/dcim/Camera` returns nothing on `~/storage/dcim`.
- Android 17 blocks hardlinks for app uids (`ln` / `fs.linkSync` → EACCES, even between plain files in `$HOME`; rename works). Not fixable via rish/Shizuku (shell-uid blocked too, SELinux enforcing) and no reflinks on f2fs. Consequence: pnpm silently falls back to copying — store + node_modules double the space. Use npm instead.
- Creating a directory under a **non-existent path inside `/proc`** hangs forever (`fs.mkdirSync('/proc/nonexistent/x', {recursive:true})` never returns). Never use a made-up `/proc/...` path as an "unwritable location" fixture in tests — put a regular file in the way instead.
- Termux runs with `umask 0077`, so a file-mode assertion that passes here (e.g. a `0644` file looking fine) can fail on CI. Force `umask 0022` when testing permission-sensitive code, and set modes explicitly instead of inheriting them.
- Other apps' private data is unreachable (Android sandbox), no Bluetooth access. Device features (camera, SMS, GPS, TTS, clipboard, sensors…) only via the `termux-api` CLI + Termux:API add-on app.
- Android's DUMP permission is not granted — anything reading system state via `dumpsys` returns nothing useful.

## Native Builds

- node-gyp / node-pty builds fail without `python`, `clang`, `make` installed first.
- `Undefined variable android_ndk_path in binding.gyp` → override `GYP_DEFINES` with a dummy value rather than installing the NDK.
- After `npm install`, re-check exec bits and shebang endings on bin scripts — updates can undo fixes.
- npm 11 blocks postinstall scripts by default (`allow-scripts`) — e.g. esbuild, `@parcel/watcher` platform binaries won't be downloaded. Builds may still work via optional platform-binary deps; always verify a build after install.
- TypeScript 7 publishes no Android arm64 build, so `tsc`/typecheck fails on-device. Either run it on a Linux host, or install an older compiler over the existing one with `--no-save` (keeps `package.json` and the lockfile upstream-conform) and repeat that after every `npm install`.

## Processes & Lifecycle

- Background processes die with the spawning session unless detached (`setsid` + `nohup`); children receive SIGTERM when a session breaks.
- Android freezes/kills background apps under memory pressure — use a wake lock for anything long-running.
- Boot-time autostart requires the Termux:Boot add-on, opened once after install; even then it depends on matching app sources (see below).
- tmux `attach` starts no new shell — `.bashrc`-based autostart never fires; hook `set-hook -g session-created` / `client-attached` to `run-shell -b '<service> status || <service> start'` instead.
- UNIX sockets in `$HOME` may fail with `EACCES` on Android — treat socket-based control interfaces as best-effort.
- RAM and swap are small and swap fills silently — "random crashes" are often OOM kills. Check memory pressure before blaming code.

## OAuth & Credentials

- `.netrc` is whitespace-delimited — passwords with spaces break it (quotes don't help); store secrets in a dedicated 600-mode file instead.
- OAuth callbacks to `localhost` work on-device: the phone's own browser can hit a local callback server, so headless-style flows run without a second machine.

## Networking

- **Android 17 LAN block: no longer the case on Johan's phone since 2026-10-31.** Termux reaches the LAN directly (ping/nc/ssh to 192.168.1.x all fine, Shizuku off); the `ACCESS_LOCAL_NETWORK` appop seems to have taken effect at reboot. Gateway is `192.168.1.254` (`.1` does not exist). Old workaround kept only as fallback: loopback tunnel via `rish` (Shizuku shell-uid) + `toybox nc -L` (`ssh nas-tunnel`). If a LAN connection ever silently times out while ICMP works, re-check this block first — and note a VPN tun (e.g. Tailscale) always worked, since it reaches LAN on the device's behalf.

## Debugging Pitfalls

- `pkill -f` / `pgrep -f` patterns match your own `bash -c` command line — bracket a character (`[-]`) or kill by exact pid.
- `getent`, `nslookup`, and similar core Linux tools are absent; `curl` and `ping` work fine.
- When a process "is running but not responding", first check for orphaned processes still holding the port before restarting.
- `rclone config create` can hang indefinitely on Termux — writing `rclone.conf` directly (with a JSON token blob) is equivalent and reliable.
- A healthy pidfile can point at a dead pid while an orphan holds the resource — trust the port/socket, not the pidfile.

## Add-ons & App Sources

- Termux add-ons (API, Boot, Widget) must come from the **same source** as the Termux app (signing keys differ between F-Droid, GitHub, and Play Store) — mixing breaks add-ons.
- Some add-ons (notably Termux:API) are not on the Play Store at all; F-Droid/GitHub is the complete source.
- `pm list packages | grep termux` reveals which add-ons are actually installed (separate package ids: `com.termux.boot`, `com.termux.api`, …) — the only reliable check, since a `~/.termux/boot/` script silently does nothing without the add-on.
- Since 2026, Android developer verification blocks installing unverified APKs on new devices; the escape hatch ("advanced flow" in developer settings) carries a 24-hour wait. Plan sideload-dependent setups accordingly.
