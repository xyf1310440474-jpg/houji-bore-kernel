# 更新日志

本文件记录 houji-bore-kernel 的变更。格式遵循 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)。

## [Unreleased] — 2026-11-22（ZRAM 优化栈启用，本地未推送）

> 真机（小米 14 / houji，6.1.138）实测发现：仓库早有 LZ4KD + NEON 优化栈，但入口开关
> `use_zram` 默认 false，从未编译进内核（`/proc/crypto` 无 lz4k/lz4kd、写回 not set）。

### Changed（变更）
- **`use_zram` 默认 `false → true`**（`kernel-custom.yml`）：根因开关，之前 build.yml 收到的
  一直是 false，LZ4KD/NEON 补丁栈从未启用。现在默认构建即带上优化栈；描述同步更新。

### Added（新增）
- **冷页写回能力**（`config/zram.config`）：新增 `CONFIG_ZRAM_WRITEBACK=y`，编译期打开
  zram 写回能力（运行时是否指定 `backing_dev` 由用户决定，默认不配置、零风险）。
- **ZRAM 优化方案文档**：[`docs/zram-optimization.md`](docs/zram-optimization.md)——真机证据、
  改动说明、与 v1.4 运行时模块的算法分工、CI 构建与刷机步骤、验证清单、风险回退。

### 未改（刻意）
- zram 默认算法仍为 lzo-rle（不动 `CONFIG_ZRAM_DEF_COMP_*`，规避 Kconfig choice 与厂商
  defconfig 冲突）；算法切换交给 houji-bore-tweaks 模块在运行时做（有安全闸门 + 失败回滚）。

## [Unreleased] — 2026-11-21（本地优化，尚未推送上游）

> 这一节是 2026-11-21 的代码审查优化，均以独立 commit 提交，见 git log。
> 每条都做了可核验的验证（语法检查 / YAML 解析 / 逻辑模拟）。

### Fixed（修复）
- **SUSFS uname 伪装已验证生效**（P0-1 核实结论）：实机确认 BakaSU 内置 SUSFS 管理器
  提供运行时下发（持久化配置 `ksu_susfs config uname add`），重启后 `uname -r` 与官方
  boot.img 值逐字一致（`g0c3d559bcd85-ab14529422`）。故**无需**额外 susfs-init 模块，
  避免重复下发冲突。README 的 SUSFS 章节记录了完整配置与验证方法。
- **README 自相矛盾的 uname 说法**：原 BORE 章节写 `...houji-BORE-R1`、SUSFS 章节写官方伪装值，
  未说明两者关系。现已明确：真实值 vs 伪装值的区别、伪装需装模块才生效、验证内核身份的可靠方法。

### Added（新增）
- **CI 自动发布 Release + SHA256**（P0-2）：`kernel-custom.yml` 新增 `publish-release` job，
  构建成功后自动下载产物、计算 SHA256（生成 `SHA256SUMS.txt`）、用 softprops/action-gh-release 发版。
  此前 Release 正文写了 SHA 但 assets 为空，产物只在 Actions artifact 里。新增 `publish_release` 输入开关（默认开）。
- **BORE 场景预设**（P1-3）：[`scripts/houji/bore-profiles/`](scripts/houji/bore-profiles/) 提供
  省电 / 均衡 / 游戏三套参数 + KSU 开机模块，参数值取自 BORE 补丁实现语义，可通过
  `/data/adb/bore_profile` 持久化选择。
- **CHANGELOG**（本文件）：建立版本追溯。

### Changed（变更）
- README 刷机章节补充 `sha256sum -c SHA256SUMS.txt` 校验步骤，以及 SUSFS 伪装模块的安装指引。

---

## [1.0] — 项目初始基线

> 上游基线快照，来源于 `xyf1310440474-jpg/houji-bore-kernel` main 分支 2026-11-21 状态
> （本地基线 commit `b004410`）。以下为此前已存在的功能，非本次优化新增。

### 内核基线
- Linux 6.1.138 GKI，KMI `android14-11`（小米 14 / houji / SM8650 / 骁龙 8 Gen 3）
- 基于 LingLuo17/AnyKernel3 构建链，clang LTO
- 其它：MGLRU、UCLAMP、FQ qdisc

### 调度器
- BORE v7.0.0（CFS KABI-safe，Masahito Suzuki；android14-6.1 backport 由 dikafarabian/workbench 提供）
- 设计原则：只改 CFS 调度算法本身，不抢高通 WALT、HyperOS miperf2/perf2-hal/powerkeeper 的 sysfs 控制权

### Root / 隐藏
- BakaSU（KernelSU 变体）
- SUSFS v2.4.0（GKI），内核内置 + 管理器内置 SUSFS 管理界面

### 安全
- CVE-2026-43499 补丁（多内核版本，见 `security_patch/`）

### 构建系统
- GitHub Actions 多版本 workflow（android12~17 / 5.10~6.18）
- 入口 `kernel-custom.yml`，`build.yml` / `get-manager.yml` 可复用
- zram LZ4 NEON、bbg、net 增强等可选组件
