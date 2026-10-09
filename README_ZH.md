# GhostLock — `diting` 移植版（Redmi K50 Ultra，5.10.236）

> English: [README.md](README.md)

本仓库是 **[YuKongA/ghostlock-app](https://github.com/YuKongA/ghostlock-app)** 的 fork，
移植目标是 **Redmi K50 Ultra**（代号 `diting`，SM8475 / 骁龙 8+ Gen 1），运行内核：

```
5.10.236-android12-9-00003-gfb24cf99ad97-ab14313284
```

上游支持 5.15 / 6.1 / 6.6 / 6.12。**5.10 不在其中**——既没有内置配置，偏移提取器也没有
5.10 的族条目。本 fork 补上了这一款，以及让它跑通所必需的几处修复。

**关于 App 本身——使用方法、偏移提取器、配置 schema、移植指南——请阅读
[上游 README](https://github.com/YuKongA/ghostlock-app/blob/main/README_ZH.md)。**
本页只说明移植改了什么、以及已经验证到什么程度。

## 当前状态

已在**原厂、未打补丁的内核**上端到端验证通过——事先无 root，无任何插桩（run `Q4`，
详见 `docs/analysis/`）：

| 阶段 | 结果 |
|---|---|
| W1 — SELinux 改为 permissive | 通过 |
| W2 — 子进程凭据覆写（`child uid = 0` → `child is root!`） | 通过 |
| W3 — 绕过 seccomp 过滤器 | 通过 |
| 交接 — root 脚本以 `uid=0` 运行、KernelSU 加载、SELinux 恢复 enforcing | 通过，`native exited code=0` |

root 是**临时的**。KernelSU 由漏洞利用现场 late-load，因此重启即失效；全程 bootloader 保持
解锁。重新上锁（会清空数据）是后续独立步骤，目前尚未执行。

### 已知缺口

整条链会跑完并交接，但 **KernelSU 模块只有在能找到「支持 late-load 的 `ksud`」时才会加载**。
本机安装的管理器是 `com.sukisu.ultra`，root 脚本的 `ksud` 搜索顺序匹配不到它，于是回退到
`/data/adb/ksu/bin/ksud`，而该版本没有 `late-load` 子命令：

```
[ksu] [*] late-load kmi=android12-5.10
[ksu] error: unrecognized subcommand 'late-load'
[ksu] [*] late-load exit=2 → [!] KernelSU module not loaded
```

把一个**支持 late-load 的 `ksud`**（可执行文件）放到 **`/data/local/tmp/ksud`** 即可解决：
它会赢得搜索顺序，且按 KernelSU 的 LKM 设计，该二进制内部自带 `kernelsu.ko`，无需另放模块文件。

### 未修复的回归——除 5.10 之外的机型请先读这一段

让 compact waiter 在 5.10 上工作的那处修复，改写了页面构造器中**一行被共享且未加保护**的代码；
而 `compact_waiter` 是一个文档级全局开关，覆盖了三种不同的 `struct rt_mutex_waiter` 布局。
结果是：**19 个内置 6.1 配置与 5 个内置 5.15 配置，会在各自内核上构造出 `prio` 读作 0（最高优先级）
的 waiter。**

- 5.10：`int prio` 位于 `0x40`，其后无字段填充
- 5.15 / 6.1：`unsigned int wake_state` 位于 `0x40`，`int prio` 位于 `0x44`

6.6 / 6.12 不受影响（它们不设置 `compact_waiter`）。完整分析、影响范围与修复方案见
[`docs/analysis/waiter-layout-family-gating-plan.md`](docs/analysis/waiter-layout-family-gating-plan.md)。
**在该问题修复之前，请勿用本分支为 5.15 或 6.1 设备构建。**

## 移植改动了什么

路线为 **`select_stack`**（pselect 路径），`waiter_shift = -2`，使用 5.10 的 **compact**
waiter —— 10 字布局
`{tree_entry, pi_tree_entry, task, lock, int prio, u64 deadline, ww_ctx}`。该几何由 `boot.img`
静态推导，并用 kprobe 在设备上实测确认（waiter 与 `stack_fds` 缓冲区落在同一栈地址，delta 为 0）。

配置之外，链路改动如下：

| commit | 改动 |
|---|---|
| `6c8b18a` | 在 5.x 内核上关闭 ghost；运行日志改为行缓冲 |
| `c1b8569` | 让受害进程的协议 fd 避开 select 路线的 fd 区间 |
| `765b137` | 快照并恢复 select 路线 `dup2` 过程覆盖的每一个 fd |
| `9acf2f1` | 零值写入改由 erase 的 collateral 完成（5.10 leaf 布局） |
| `a703604` | 让 `lock_anchor_image` 从配置文档一路传到 wire |
| `6e70e8c` | 移植计划与分析文档 |

后两处 fd 修复是「写入能落地」与「整条链跑通」的分界：在此之前，路线的 `dup2` 过程会静默销毁
长期存活的描述符，导致 W2 的受害子进程在写入落地前一轮就看到 EOF 退出。

## 复现步骤

配置文件**不是**内置资源，而是以用户文档方式导入。

1. 在主机上提取（务必加 `--iomem /dev/null`——否则工具会去读**构建主机**的 `/proc/iomem`，
   并静默给出一个看似合理的错误 `kernel_phys_load`）：

   ```sh
   ghostlock-extract boot.img --iomem /dev/null --phys 0xa8000000 --format conf --out profile.conf
   ```

2. 在 App 中导入：**配置参数 → 导入 offsets.conf（v2）**。导入的用户文档优先于内置配置。
3. 用 root shell 设置 `settings_enable_monitor_phantom_procs = false`。否则 Android 的
   phantom-process 清理机制会在运行中途 SIGKILL 掉 native 进程（272 个 spray 子进程远超平台上限），
   其现象与漏洞利用失败无法区分。该设置需要 `WRITE_SECURE_SETTINGS`，`adb shell` 无法写入。
4. 运行。

本移植所用的配置文件（每个取值都标注了推导与验证方式）与证据链（App 日志、内核 trace、
oops 文本）保存在本仓库之外。

## 目录

- [`docs/analysis/`](docs/analysis/) —— 计划文档：payload page、waiter 生命周期、零值写入的
  descent，以及上文所述的 waiter 布局回归。
- [`docs/kernel_profiles/`](docs/kernel_profiles/) —— 上游的移植指南与配置 schema，未改动。

## 致谢与许可

本仓库是 [YuKongA/ghostlock-app](https://github.com/YuKongA/ghostlock-app) 的 fork，
采用 Apache License 2.0（见 [LICENSE](LICENSE)）。上游另致谢：

- [NebuSec/CyberMeowfia](https://github.com/NebuSec/CyberMeowfia)
- [JoinChang/ghostlock-oneplus](https://github.com/JoinChang/ghostlock-oneplus)
- [x-spy/CVE-2026-43499-popsicle](https://github.com/x-spy/CVE-2026-43499-popsicle)
