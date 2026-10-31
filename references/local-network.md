# Local network access

Android can block an app's LAN traffic through the `ACCESS_LOCAL_NETWORK` app op. The failure mode is silent: ICMP (ping) still works, TCP connections to LAN hosts just time out, so it does not look like a permission problem.

## Getting the permission

The app op is not a platform permission, so there is no manifest entry to add and no guaranteed Settings toggle. Grant it from a privileged shell instead — `adb shell`, or a Shizuku-backed shell such as `rish` — as the **uid** mode, not the package mode:

```
appops set --uid com.termux ACCESS_LOCAL_NETWORK allow
```

Package mode alone has no effect: enforcement reads the uid mode.

## The grant only takes effect after a reboot

**Reboot the phone.** Restarting Termux, the app process, or its sessions does not apply the grant — the process keeps the state it started with. Do not conclude the permission failed before rebooting once; a grant that "did nothing" is usually an unrebooted device.

## Verify

From Termux, not from the privileged shell (the shell uid always has LAN access, so it proves nothing):

1. `ping` a LAN host — a reply only means the block is ICMP-permeable.
2. `nc -vz <host> <port>` (or any TCP connect) — that is the real test.
3. When outside the LAN, a VPN tun (WireGuard, Tailscale) works regardless of the app op, since the VPN reaches the LAN on the device's behalf.

## Before the reboot, or without permission changes

Route through a uid that does have LAN access. A Shizuku shell can act as a loopback forwarder — `toybox nc -L` is server mode, so it accepts multiple connections and streams:

```
rish -c 'toybox nc -L -s 127.0.0.1 -p 2222 toybox nc <host> <port>'
```

Start it detached from Termux (`setsid nohup ... &`); a child started inside a `rish -c` call is reaped as soon as the call returns. The forwarder keeps running as the shell uid, so it is invisible to Termux's process tools — kill it through `rish` again.
