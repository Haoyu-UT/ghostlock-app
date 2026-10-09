# GhostLock — `diting` port (Redmi K50 Ultra, 5.10.236)

> 中文: [README_ZH.md](README_ZH.md)

## Supported devices

| Kernel | Devices | Codename | Status |
|---|---|---|---|
| `5.10.236-android12-9-00003-gfb24cf99ad97-ab14313284` | Redmi K50 Ultra · Xiaomi 12T Pro | `diting` | **tested end to end** |

The Redmi K50 Ultra is codename `diting` (SM8475 / Snapdragon 8+ Gen 1). **Xiaomi 12T Pro is the
same physical device** — one codename, three retail names (also sold as the Redmi K50 Extreme
Edition). It is not a second port.

Reported as possibly sharing the build, but **untested**:

| Retail name | Codename | SoC | vs. the tested device |
|---|---|---|---|
| Xiaomi 12S Ultra | `thor` | SM8475 | same SoC |
| Xiaomi 12 | `cupid` | SM8450 | different SoC |
| Redmi Note 13 Pro 5G | `garnet` | SM7435 | different SoC |
| POCO X6 5G | `garnet` | SM7435 | different SoC |
| POCO F5 | `marble` | SM7475 | different SoC |
| Redmi Pad Pro | `dizi` / `ruan` | SM7435 | different SoC |

None of these has been run. Only the Xiaomi 12S Ultra is on the same SoC as the tested device; for
the rest, **a dedicated profile may be required** — a device on a different SoC is not expected to
ship the same certified kernel build.

This fork also **removes 24 upstream kernels** from the supported list, because its
compact-waiter change breaks them. The full table, including which devices those 24 covered, is in
[`docs/kernel_profiles/SUPPORTED_DEVICES.md`](docs/kernel_profiles/SUPPORTED_DEVICES.md).

## About this fork

A fork of **[YuKongA/ghostlock-app](https://github.com/YuKongA/ghostlock-app)**. Upstream supports
5.15 / 6.1 / 6.6 / 6.12 — **5.10 is not among them**: there is no built-in profile for it, and no
5.10 family entry in the offset extractor. This fork adds one, plus the fixes that turning it on
required.

**For the app itself — usage, the offset extractor, the profile schema, the porting guide — read
[upstream's README](https://github.com/YuKongA/ghostlock-app#readme).** This page only covers what
the port changed and what it has been shown to do.

## Status

Verified end-to-end on a **stock, unpatched kernel** — no root beforehand, no instrumentation:

| stage | result |
|---|---|
| W1 — SELinux → permissive | works |
| W2 — child credential write (`child uid = 0` → `child is root!`) | works |
| W3 — seccomp filter bypass | works |
| Handoff — root script as `uid=0`, KernelSU loaded, SELinux back to enforcing | works, `native exited code=0` |

Root is **temporary**. KernelSU is late-loaded from the exploit, so it is gone on reboot; the
bootloader is unlocked throughout. Relocking (which wipes data) is a separate, later step and has
not been done.

### Known gap

The chain completes and hands off, but the KernelSU module only loads if a **late-load-capable
`ksud`** is reachable. The manager installed here is `com.sukisu.ultra`, which the root script's
`ksud` search order does not match — so it falls through to `/data/adb/ksu/bin/ksud`, which has no
`late-load` subcommand:

```
[ksu] [*] late-load kmi=android12-5.10
[ksu] error: unrecognized subcommand 'late-load'
[ksu] [*] late-load exit=2 → [!] KernelSU module not loaded
```

Dropping a late-load-capable `ksud` (executable) at **`/data/local/tmp/ksud`** is sufficient: it
wins the search, and per KernelSU's LKM design that binary carries its own `kernelsu.ko` — no
separate module file is needed.

### Open regression — read this before using the fork on anything but a 5.10 device

**This fork breaks 24 of upstream's built-in profiles.** It is a defect in the fork, not in
GhostLock — upstream supports those devices normally, and nothing here affects it.

The change that makes the compact waiter work on 5.10 sits on a code path shared with every kernel
family, so the **19 built-in 6.1 profiles and 5 built-in 5.15 profiles** now build a waiter whose
priority is written to the wrong field. The families disagree on the waiter layout — 5.10 has
`int prio` at `0x40`, while 5.15 and 6.1 have `unsigned int wake_state` at `0x40` and `int prio` at
`0x44` — and the fork writes the 5.10 pair unconditionally.

6.6 and 6.12 are unaffected, as they don't use the compact waiter. The devices those 24 kernels
covered are listed in
[`docs/kernel_profiles/SUPPORTED_DEVICES.md`](docs/kernel_profiles/SUPPORTED_DEVICES.md), and the
analysis with the planned fix is in
[`docs/analysis/waiter-layout-family-gating-plan.md`](docs/analysis/waiter-layout-family-gating-plan.md).
**Do not build this fork for a 5.15 or 6.1 device until that is fixed.**

## What the port changed

The route is **`select_stack`** (the pselect path), with `waiter_shift = -2` and the 5.10
**compact** waiter — the 10-word
`{tree_entry, pi_tree_entry, task, lock, int prio, u64 deadline, ww_ctx}` shape, derived statically
from `boot.img` and confirmed live on the device with kprobes (the waiter and the `stack_fds`
buffer land on the same stack address, delta 0).

The chain on top of the profile:

| commit | change |
|---|---|
| `6c8b18a` | Disarm the ghost on 5.x kernels; line-buffer the run log |
| `c1b8569` | Keep the victim's protocol fds out of the select route's fd range |
| `765b137` | Snapshot and restore every fd the select route's `dup2` pass clobbers |
| `9acf2f1` | Perform zero writes through the erase's collateral (the 5.10 leaf layout) |
| `a703604` | Carry `lock_anchor_image` from the profile document through to the wire |
| `6e70e8c` | Port plans and analysis documents |

The last two fd fixes were the difference between "the write lands" and "the chain completes" —
before them the route's `dup2` pass silently destroyed long-lived descriptors, so the W2 victim saw
EOF one dance before its write landed.

## Reproducing

The profile is **not** a built-in asset; it is imported as a user document.

1. Extract on a host (`--iomem /dev/null` — the tool otherwise reads the *build host's*
   `/proc/iomem` and silently produces a plausible wrong `kernel_phys_load`):

   ```sh
   ghostlock-extract boot.img --iomem /dev/null --phys 0xa8000000 --format conf --out profile.conf
   ```

2. Import it in the app: **配置参数 → 导入 offsets.conf（v2）**. An imported user document takes
   precedence over the built-ins.
3. Set `settings_enable_monitor_phantom_procs = false` from a root shell. Android's phantom-process
   trimmer otherwise SIGKILLs the native binary mid-run (the 272 spray children blow past the
   platform cap), which is indistinguishable from an exploit failure. This needs
   `WRITE_SECURE_SETTINGS`, so `adb shell` cannot set it.
4. Run.

A profile is specific to one exact kernel build, so extract one for your own device as above rather
than reusing a file from here — the values differ per build, and a mismatch fails at W1.

## Layout

- [`docs/analysis/`](docs/analysis/) — design notes and analysis behind this port, including the
  regression described above.
- [`docs/kernel_profiles/`](docs/kernel_profiles/) — upstream's porting guide and profile schema,
  unchanged.

## Credits & License

Fork of [YuKongA/ghostlock-app](https://github.com/YuKongA/ghostlock-app), Apache License 2.0 (see
[LICENSE](LICENSE)). Upstream in turn credits:

- [NebuSec/CyberMeowfia](https://github.com/NebuSec/CyberMeowfia)
- [JoinChang/ghostlock-oneplus](https://github.com/JoinChang/ghostlock-oneplus)
- [x-spy/CVE-2026-43499-popsicle](https://github.com/x-spy/CVE-2026-43499-popsicle)
