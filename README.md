# Termux Skill

An [agent skill](https://github.com/earendil-works/pi) that gives AI agents hard-learned, non-obvious knowledge about working on **Termux / Android** — so they stop rediscovering the same pitfalls every session.

## Why

Agents are mayflies: every session they try `/tmp`, assume `/usr/bin`, build `node-pty` without `clang`, `pkill` their own command, and wonder why shared storage ignores symlinks. On a phone, Linux intuition is wrong in specific, recurring ways. This skill encodes those ways as terse hints — no tutorials, no examples, just the traps and the way around them.

## Scope

**Only non-obvious, experience-earned facts:**

- Paths & filesystem (`$PREFIX`, FUSE storage, sandbox walls)
- Native builds (node-gyp on aarch64 Android)
- Process lifecycle (detaching, wake locks, OOM, orphaned ports)
- Debugging pitfalls (self-matching pkill patterns, missing tools)
- Add-ons & app sources (signing keys, sideloading rules)

Everything that behaves like standard Linux is deliberately omitted.

## Install

Copy or symlink this folder into your agent's skills directory, e.g. as `~/.agents/skills/termux`:

```
ln -s /path/to/termux-agent-skill ~/.agents/skills/termux
```

## Layout

| File | Audience |
|------|----------|
| `SKILL.md` | The agent — loaded when working on a Termux device |
| `README.md` | You, right now |

## Contributing

The bar for adding a line: **it must have actually bitten someone**, and the fix must fit in one sentence. If a lesson needs an example, it's too long — sharpen it.
