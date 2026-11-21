# houji-bore-kernel

> 小米 14（houji / SM8650 / 骁龙 8 Gen 3）专用 GKI 内核
> 基于小凌洛（LingLuo17/AnyKernel3）构建链，加入 **BORE v7.0.0 CPU 调度器**与 **BakaSU + SUSFS 官方级伪装**。

## 核心特性

| 项目 | 说明 |
|---|---|
| 内核基线 | Linux 6.1.138 GKI，KMI `android14-11` |
| 调度器 | **BORE v7.0.0**（Burst-Oriented Response Enhancer，CFS KABI-safe），提升 UI 响应与触控延迟 |
| Root | BakaSU（KernelSU 变体） |
| 隐藏 | SUSFS v2.4.0（GKI），内核内置 + BakaSU 管理器内置 SUSFS 管理界面 |
| 其他基线 | MGLRU、UCLAMP、KSU_SUSFS、FQ qdisc、clang LTO |
| 不含 | 无第三方用户态调度守护进程，不改 GPU 频率策略（保留系统原生调频） |

> **设计原则**：只改调度算法本身（CFS 核心），不与高通 WALT governor、HyperOS 原生 miperf2/perf2-hal/powerkeeper 抢 sysfs 控制权。BORE 决定「哪个任务先跑」，原生调度决定「频率给多少」，两者正交共存。

## BORE 调度器

BORE 通过 burst penalty 对任务重加权：优先调度「不那么突发」的任务，改善重负载下的响应性。默认参数（运行时可调）：

| 参数（`/proc/sys/kernel/`） | 默认值 |
|---|---|
| `sched_burst_smoothness` | 1 |
| `sched_burst_penalty_offset` | 24 |
| `sched_burst_penalty_scale` | 1536 |
| `sched_burst_cache_lifetime` | 75000000 (75ms) |
| `sched_burst_inherit_type` | 2 |

验证生效：
```sh
uname -r                                  # 6.1.138-android14-11-houji-BORE-R1
ls /proc/sys/kernel/sched_burst_*         # 5 个节点存在即生效
grep bore /proc/kallsyms                  # effective_prio_bore / sched_init_bore 等
```

## SUSFS 隐藏（官方级伪装）

内核已内建 SUSFS v2.4.0，BakaSU 管理器内置 SUSFS 管理界面。伪装值取自官方 boot.img 的真实 uname，与官方 `user/release-keys` 系统逐字一致：

```
uname -r → 6.1.138-android14-11-g0c3d559bcd85-ab14529422
uname -v → #1 SMP PREEMPT Wed Dec  3 02:05:30 UTC 2025
```

已配置隐藏项（`ksu_susfs config list_all` 可查）：
- `sus_path`：`/data/adb`、`/data/local/tmp`（--loop，对新进程持续生效）
- `sus_map`：`/data/adb`
- `hide_sus_mnts_for_non_su_procs`、`avc_log_spoofing`：开启
- `logging`：关闭（不留日志痕迹）

命令行工具：`/data/adb/ksu/bin/ksu_susfs`（`config restore` 可恢复默认）。

## 刷机

1. 下载 Release 里的 `*-AnyKernel3.zip`（标准 AnyKernel3 包）。
2. 在第三方 Recovery（OrangeFox 等）中刷入，或刷入 **boot** 分区。
3. **不碰 recovery / data / vbmeta**。A/B 设备注意刷当前活动槽位。

刷前务必备份当前 boot：
```sh
dd if=/dev/block/by-name/boot_a of=/sdcard/Download/boot_backup.img bs=1M
```

回退：
```sh
dd if=/sdcard/Download/boot_backup.img of=/dev/block/by-name/boot_a bs=1M
```

## ZRAM 压缩优化（LZ4KD / NEON / 冷页写回）

仓库内置 LZ4KD + ARM64 NEON 加速 + 冷页写回优化栈，`kernel-custom.yml` 的 `use_zram`
默认开启（6.1 走完整栈，6.12/6.18 上游无补丁源会自动跳过）。编译后 zram 支持
`lz4k` / `lz4kd`，写回能力就绪。算法默认仍是 lzo-rle，运行时切换交给
`houji-bore-tweaks` 模块（有安全闸门），避免编译期 choice 冲突。

完整说明、真机证据、构建/刷机/验证/回退步骤见
[`docs/zram-optimization.md`](docs/zram-optimization.md)。

## 构建

Fork 自 LingLuo17/AnyKernel3。GitHub Actions 自动构建，入口 workflow `kernel-custom.yml`。BORE 补丁位于 `patches/bore/bore-v7.0.0.patch`，由 `build.yml` 的「集成 BORE」步骤应用并打开 `CONFIG_SCHED_BORE=y`。

CI 触发用 `workflow_dispatch`，参数 `kernel_version=6.1 / kernelsu_variant=BakaSU / sew_stage=off`。

## 致谢

- LingLuo17/AnyKernel3 — 构建链与 GKI 集成
- BORE CPU Scheduler — Masahito Suzuki
- dikafarabian/workbench — BORE android14-6.1 KABI-safe 补丁
- afarhandanu/ZephyrKernel — GKI 6.1 调度优化组装参考
- WildKernels — AnyKernel3 核心

> 本内核为个人学习与折腾产物，按「现状」提供，刷机有风险，请自行备份。
