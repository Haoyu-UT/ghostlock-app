# GhostLock — `diting` port (Redmi K50 Ultra, 5.10.236)

> 中文: [README_ZH.md](README_ZH.md)

This is a fork of **[YuKongA/ghostlock-app](https://github.com/YuKongA/ghostlock-app)** carrying a
port to the **Redmi K50 Ultra** (codename `diting`, SM8475 / Snapdragon 8+ Gen 1), running:

```
5.10.236-android12-9-00003-gfb24cf99ad97-ab14313284
```

Upstream supports 5.15 / 6.1 / 6.6 / 6.12. **5.10 is not among them** — there is no built-in
profile for it, and 5.10 has no family entry in the offset extractor. This fork adds one, plus the
fixes that turning it on required.

**For the app itself — usage, the offset extractor, the profile schema, the porting guide — read
[upstream's README](https://github.com/YuKongA/ghostlock-app#readme).** This page only covers what
the port changed and what it has been shown to do.

## Status

Verified end-to-end on a **stock, unpatched kernel** — no root beforehand, no instrumentation
(run `Q4`; see `docs/analysis/`):

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

### Open regression — read before using this fork for anything but 5.10

The fix that makes the compact waiter work on 5.10 rewrote a **shared, unguarded** line in the page
builder, and `compact_waiter` is a document-global flag covering three different
`struct rt_mutex_waiter` layouts. The result is that **19 built-in 6.1 profiles and 5 built-in 5.15
profiles build a waiter whose `prio` reads 0 (highest) on their kernels.**

- 5.10 has `int prio` at `0x40` as plain padding-free layout
- 5.15 / 6.1 have `unsigned int wake_state` at `0x40` and `int prio` at `0x44`

6.6 / 6.12 are unaffected (they set no `compact_waiter`). The full analysis, blast radius, and the
planned fix are in
[`docs/analysis/waiter-layout-family-gating-plan.md`](docs/analysis/waiter-layout-family-gating-plan.md).
**Do not build this branch for a 5.15 or 6.1 device until that is fixed.**

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

The port's own profile, with every value annotated with how it was derived and verified, is kept
outside this repository along with the evidence trail (app logs, kernel traces, oops texts).

## Layout

- [`docs/analysis/`](docs/analysis/) — plan documents: the payload page, the waiter lifetime, the
  zero-write descent, and the waiter-layout regression above.
- [`docs/kernel_profiles/`](docs/kernel_profiles/) — upstream's porting guide and profile schema,
  unchanged.

## Credits & License

Fork of [YuKongA/ghostlock-app](https://github.com/YuKongA/ghostlock-app), Apache License 2.0 (see
[LICENSE](LICENSE)). Upstream in turn credits:

- [NebuSec/CyberMeowfia](https://github.com/NebuSec/CyberMeowfia)
- [JoinChang/ghostlock-oneplus](https://github.com/JoinChang/ghostlock-oneplus)
- [x-spy/CVE-2026-43499-popsicle](https://github.com/x-spy/CVE-2026-43499-popsicle)
