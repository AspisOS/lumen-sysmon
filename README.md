# lumen-sysmon

The system monitor for **AspisOS**, a capability-based, no-ambient-authority
x86-64 operating system built on the from-scratch
[Aegis](https://github.com/AspisOS/Aegis) kernel. sysmon is a live task and
resource monitor: a memory bar, an uptime/process-count line, and a sortable
process table with a kill action. It is a standalone client of the
[lumen](https://github.com/AspisOS/lumen) compositor, distributed as a
[herald](https://github.com/AspisOS/AspisOS) system package and installed into
the `/apps` bundle tree.

## Where sysmon fits

AspisOS is decomposed into independent repositories. sysmon is a leaf: a GUI
application that talks only to the compositor and reads `/proc`.

| Repo | Role |
|------|------|
| [`AspisOS/Aegis`](https://github.com/AspisOS/Aegis) | The kernel. Provides `procfs` (`/proc/meminfo`, `/proc/<pid>/status`), `AF_UNIX` sockets, `memfd`, signals, and the capability model. |
| [`AspisOS/lumen`](https://github.com/AspisOS/lumen) | The compositor / display server. sysmon's window is a proxy surface owned by lumen. |
| [`AspisOS/glyph`](https://github.com/AspisOS/glyph) | The GUI toolkit. Supplies the software renderer (`draw_*`) and the **client side of lumen's window protocol** (`lumen_client.h`). |
| `AspisOS/lumen-sysmon` | **This repo.** An external Lumen client, the same pattern as the calculator, settings, and terminal. |

sysmon does not touch the framebuffer or input devices. It connects to lumen
over `/run/lumen.sock`, receives a shared `memfd` to draw into, and gets input
events back. In herald terms it declares `depends=lumen`.

## What it does

`main()` (`src/main.c`) connects to lumen, creates a window titled "System
Monitor", and runs a 1000 ms-timeout event loop that re-scans `/proc` each tick
(or on demand when input arrives). It requires **no new kernel support** — every
metric comes from reading `/proc`:

- **Memory bar.** `/proc/meminfo` `MemTotal` / `MemFree` drive a used-memory bar,
  colored green / yellow / red at 70% / 90%, labelled `used / total MB (pct%)`.
- **Uptime + count.** `CLOCK_MONOTONIC` is shown as `up Xh Ym Zs`, alongside the
  process count.
- **Process table.** `readdir("/proc")` collects the all-digit entries as pids,
  and each `/proc/<pid>/status` yields `Name` / `State` / `VmSize`. The table is
  sorted by `VmSize` descending (pid ascending on ties).
- **The size column** reports `VmSize` (the sum of VMA lengths, in kB), labelled
  `MEM(KB)`. Aegis procfs does not yet expose `VmRSS` — there is no resident-set
  accounting — so sysmon mirrors exactly what `ps` reads.
- **Input.** Arrow keys (raw or Lumen's synthetic `0xF1`/`0xF2`) move the
  selection, a left click selects a row, the wheel and a scrollbar scroll the
  list, and `k` sends `SIGTERM` to the selected pid. pid 0 and pid 1 (init) are
  guarded against signalling. The selected process is tracked by pid across a
  re-sort.
- **Window sizing.** 720×560, clamped to the framebuffer reported via `LUMEN_FB_W`
  / `LUMEN_FB_H`. The process table is capped at 256 entries (truncation is
  logged to stderr).

## Status

sysmon is **early-stage** and intentionally minimal: a roughly 1 Hz `/proc`
reader with a memory bar, a `VmSize`-sorted table, and a `SIGTERM` kill. It is
expected to grow — for example per-process CPU accounting, resident-set memory
once the kernel exposes `VmRSS`, richer system-wide metrics, and additional
signal/priority actions — as the kernel's introspection surface expands. The
current build is deliberately scoped to what `/proc` already provides, so it
needs no new kernel support to ship.

## Capabilities

AspisOS has **no ambient authority**: a process can do nothing except through
capabilities granted by kernel policy at exec time. sysmon's policy
(`pkg/etc/aegis/caps.d/sysmon`, installed to `/etc/aegis/caps.d/sysmon`) is the
baseline service profile:

```
service
```

It needs nothing beyond `service`: it reads `/proc` and connects to the
compositor over the Lumen socket. Sending `SIGTERM` to a selected process uses
the ordinary `kill` path within that profile. It holds no `AUTH`, `SETUID`,
`FB`, or network capability.

## Building

sysmon builds with a musl cross-compiler against the prebuilt glyph toolkit. The
`Makefile` fetches a pinned toolkit artifact, compiles `src/*.c` against it, and
packs a signed herald package.

```sh
make MUSL_CC=/path/to/musl-gcc HERALD_KEY=/path/to/signing.key
```

- `GLYPH_VERSION` pins the [glyph](https://github.com/AspisOS/glyph) toolkit
  release fetched by `tools/fetch-glyph.sh` (it unpacks `include/` and `lib/`
  into `toolkit/`).
- `MUSL_CC` is the musl cross-compiler (defaults to `musl-gcc` on `PATH`; the
  only toolchain assumption — point it at an Aegis-native `cc` to build on-device
  in the future).
- `HERALD_KEY` is the ECDSA P-256 signing key for the package.
- The link line pulls all four toolkit archives (`-lcitadel -laudio -lauth
  -lglyph`); only the objects actually referenced are contributed.

Output: `lumen-sysmon.hpkg` (a `class=system` herald package) +
`lumen-sysmon.hpkg.sig`.

## Package payload

`lumen-sysmon.hpkg` is a manifest-first, uncompressed POSIX `ustar` archive with
a detached ECDSA-P256/SHA-256 signature (`tools/pack.sh`). The herald package
**id** (`lumen-sysmon`) differs from the bundle/exec name (`sysmon`), and the
package installs an `/apps` binary alongside a cap policy under `/etc` — so it is
a `class=system` package: first-party, signature-trusted, installed verbatim by
herald.

```
manifest                          id=lumen-sysmon, class=system, depends=lumen
apps/sysmon/sysmon                the system monitor binary (stripped)
apps/sysmon/app.ini               the Lumen app-bundle manifest (name=System Monitor, exec=sysmon)
etc/aegis/caps.d/sysmon           its capability policy (service)
```

## Repository layout

```
.
├── Makefile          fetch toolkit → build component.elf → pack the .hpkg
├── VERSION           this component's version
├── GLYPH_VERSION     pinned glyph toolkit artifact version
├── src/
│   └── main.c        /proc scan, memory bar, process table, kill, event loop
├── pkg/              install-tree skeleton shipped verbatim
│   ├── apps/sysmon/app.ini      the /apps bundle manifest
│   └── etc/aegis/caps.d/sysmon  the capability policy
└── tools/
    ├── fetch-glyph.sh   fetch + unpack the pinned glyph toolkit artifact
    └── pack.sh          build + sign lumen-sysmon.hpkg
```

Build outputs (`component.elf`, `*.hpkg`, `*.hpkg.sig`) and the fetched
`toolkit/` are git-ignored.

## Dependencies

`depends=lumen` — sysmon is a client of the
[lumen](https://github.com/AspisOS/lumen) compositor, so installing it pulls
lumen, which in turn ships the desktop fonts every Lumen client inherits. There
is no separate font package.

## Sibling components

- [bastion](https://github.com/AspisOS/bastion) — display manager / login greeter
- [lumen-imageviewer](https://github.com/AspisOS/lumen-imageviewer) — image viewer
- [lumen-netman](https://github.com/AspisOS/lumen-netman) — network status panel
