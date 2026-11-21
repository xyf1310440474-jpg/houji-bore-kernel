# ZRAM 优化栈启用方案（LZ4KD + NEON + 冷页写回）

> 适用：`houji-bore-kernel`，小米 14（houji / SM8650），内核 6.1.138 / KMI android14-11
> 日期：2026-11-22　对象：真机 `30a4ef35`（实测数据见文末「基线实测」）

## 一、为什么要做（真机证据）

| 检查项 | 真机实测 | 结论 |
|---|---|---|
| zram 当前算法 | `lzo-rle`（`lzo [lzo-rle] lz4 zstd`） | 默认算法，无 NEON 加速 |
| `/proc/crypto` 算法 | 只有 `zstd / lz4 / lzo / lzo-rle` | **没有 lz4k / lz4kd**，优化栈没编进来 |
| `CONFIG_ZRAM_WRITEBACK` | `is not set` | 冷页写回能力未开 |
| `kernel-custom.yml` 的 `use_zram` | 默认 `false` | **根因**：构建时开关没开，仓库里的优化栈从来没被启用 |
| zram 内存占用 | `mm_stat` orig≈5.9G / compr≈2.3G | 压缩比约 2.5:1，还有提升空间 |

仓库里其实早就有完整的 LZ4KD + ARM64 NEON 优化栈（`zram/` 目录 + `build.yml` 集成步骤），
只是因为入口开关默认关闭，从未编译进你正在用的内核。本方案把它打开，并补上冷页写回。

## 二、改了什么（本次 patch）

1. **`.github/workflows/kernel-custom.yml`**：`use_zram` 默认 `false → true`。
   - 这是根因开关：`build.yml` 收到的 `use_zram` 来自这里，之前一直是 false。
2. **`config/zram.config`**：新增 `CONFIG_ZRAM_WRITEBACK=y`（带注释）。
   - 6.1 走 build.yml 的「剔除 `ZRAM_BACKEND_*` 后追加」分支，本行会进 defconfig。
   - 上游符号在 5.10~6.6 Kconfig 均存在，bazel `kernel_config` 校验可过。

**没有动的**：zram 默认算法（`CONFIG_ZRAM_DEF_COMP` 仍是 lzo-rle）。
理由见下节——算法切换交给已有的运行时模块，避免编译期 choice 冲突。

## 三、启用后能得到什么

### 1. LZ4KD + NEON 加速（编译期）
- `build.yml` 的「配置 ZRAM LZ4 补丁栈」会：升级 `lib/lz4` → 打 `apply_lz4_neon.sh`（解压走 ARM64 NEON 指令）→ 集成 lz4k / lz4kd / lz4k_oplus（来自 `ShirkNeko/SukiSU_patch`，6.1 补丁已核实存在）。
- 编译后 `/proc/crypto` 会出现 `lz4k` / `lz4kd`，`zram comp_algorithm` 支持列表新增这两个算法。
- LZ4KD 相比 LZ4 主打更高压缩比，NEON 加速降低解压 CPU 开销 → 同样内存能装更多页、换出换入更快。

### 2. 冷页写回（编译期开能力，运行时按需启用）
- `CONFIG_ZRAM_WRITEBACK=y` 让 zram 具备把「长期不访问的压缩页」写回磁盘的能力，进一步释放 zram 占用。
- **重要**：编译期只打开能力，默认不生效。真正启用要运行时指定后端块设备：
  ```sh
  echo /dev/block/by-name/<某个分区> > /sys/block/zram0/backing_dev
  ```
- 手机没有独立 swap 分区，**不建议随便指一个分区**（写错会破坏数据）。默认保持不配置，
  即"能力就绪但不用"，零风险。是否启用由你决定，见第六节风险。

## 四、和运行时模块（houji-bore-tweaks v1.4）的配合

这是关键分工，避免两边抢节点：

- **内核侧**：负责"把更好的算法编进来"（lz4kd/lz4k + NEON）——这是模块做不到的。
- **模块侧**：负责"运行时切换算法"（已有完整 swapoff→切换→swapon→失败回滚 + 内存安全闸门）。
  - 默认 `ZRAM_ALGO=zstd`，`ENABLE_ZRAM=0`（关）。
  - 刷完新内核后，若要改用 LZ4KD，把模块配置改成 lz4kd 再开：
    ```sh
    su -c '/data/adb/houji-tweaks/tweaks set ZRAM_ALGO lz4kd'
    su -c '/data/adb/houji-tweaks/tweaks feature zram 1'
    ```
  - 模块会探测 lz4kd 是否真的在支持列表里，不在就拒绝切换并回滚，不会硬写。
- **为什么不让内核编译期直接设 lz4kd 为默认**：`CONFIG_ZRAM_DEF_COMP_*` 是 Kconfig choice，
  和厂商 defconfig 的 `LZORLE=y` 有冲突风险；且算法切换涉及 swapoff，放运行时做有安全闸门兜底，更稳。

## 五、构建与刷机步骤

### 构建（GitHub Actions）
1. 把本 patch 应用到你的 `houji-bore-kernel` 仓库（`git apply houji-kernel-zram-optimizations.patch`），
   或用 `houji-kernel-zram-optimizations.tar.gz` 覆盖对应文件。
2. Actions → `kernel-custom.yml` → `workflow_dispatch`，参数：
   - `kernel_version=6.1`，`kernelsu_variant=BakaSU`，`ksu_mode`（按你现状）
   - **`use_zram=true`**（现在默认也是 true，显式确认一次）
   - `publish_release=true`（自动发 Release + SHA256）
3. 构建完成后从 Release 下载 `*-AnyKernel3.zip`，先校验：
   ```sh
   sha256sum -c SHA256SUMS.txt --ignore-missing
   ```

### 刷机（先备份！）
```sh
# 1. 备份当前 boot（正在用的、已验证可用的内核）
su -c 'dd if=/dev/block/by-name/boot_a of=/sdcard/Download/boot_zram_backup.img bs=1M'
# A/B 设备确认活动槽位：getprop ro.boot.slot_suffix
```
第三方 Recovery（OrangeFox 等）刷入新 AK3 到当前活动槽位 boot。**不碰 recovery / data / vbmeta**。

## 六、验证清单（刷完逐条核）

```sh
su
# 1. 内核身份没变（KMI 仍是 android14-11）
cat /proc/version
# 2. LZ4KD 真的编进来了
grep -iE 'lz4k|lz4kd' /proc/crypto | head
cat /sys/block/zram0/comp_algorithm      # 应含 lz4k / lz4kd
# 3. 写回能力已开（节点不再是空/不可用）
zcat /proc/config.gz | grep ZRAM_WRITEBACK   # =y
# 4. BORE / SUSFS 没被撞掉（本次改动不涉及，但要回归）
ls /proc/sys/kernel/sched_burst_*
uname -r
# 5. 模块切算法（可选，确认运行时链路通）
/data/adb/houji-tweaks/tweaks zram status
/data/adb/houji-tweaks/tweaks set ZRAM_ALGO lz4kd
/data/adb/houji-tweaks/tweaks feature zram 1
/data/adb/houji-tweaks/tweaks zram status
# 6. 日常观察 1% low 帧、发热、有无 OOM
```

**任一项异常立即回退**：
```sh
su -c 'dd if=/sdcard/Download/boot_zram_backup.img of=/dev/block/by-name/boot_a bs=1M'
su -c 'reboot'
```
模块侧回退算法：`tweaks zram restore` + `tweaks feature zram 0`。

## 七、风险与边界（诚实说明）

| 风险 | 等级 | 说明 / 规避 |
|---|---|---|
| LZ4KD 补丁与 6.1.138 冲突 | 中 | build.yml 对 lz4kd.patch 应用失败只 warning 不中断（`-F 3` fuzz）。**验证第 2 条就是查这个**——没有 lz4kd 说明补丁没吃上，此时算法切换会跳过，不影响系统。 |
| 写回指错 backing_dev | **高** | 默认不配置，零风险。若启用，务必指向专用分区/文件，写错会损坏数据。**建议先不启用**。 |
| 编译期 choice 冲突 | 低 | 本次没动 `CONFIG_ZRAM_DEF_COMP`，沿用 lzo-rle，规避了和厂商 defconfig 的冲突。 |
| KMI / vendor 模块 ABI | 低 | KMI 仍是 android14-11，未升基线，wlan/触摸模块 ABI 不变。 |
| 与模块抢算法节点 | 低 | 内核只提供算法，不设默认；切换全归模块，且有闸门+回滚。 |

**不做的事**：不升内核基线（6.1.138 已在真机全项验证过）、不改调度层（BORE 已生效且是 gaming 档）、
不碰 thermal / dtsi（GKI 不含设备树）。

## 八、基线实测（2026-11-22，真机 30a4ef35）

```
内核: 6.1.138-android14-11-houji-BORE-R1, KMI android14-11
BORE: sched_burst_* 5 节点全在；当前 gaming 预设 (offset=16 scale=2048 smoothness=0 lifetime=100ms)
SUSFS: uname -r = 6.1.138-android14-11-g0c3d559bcd85-ab14529422（已伪装）
zram: comp=lzo-rle, disksize=16G, /proc/crypto 无 lz4k/lz4kd, CONFIG_ZRAM_WRITEBACK not set
内存: MemTotal 15.5G, swap 已用约 5.8G
补丁源: ShirkNeko/SukiSU_patch 的 6.1 zram 补丁 4 项资源全部 HTTP 200 存在
```
