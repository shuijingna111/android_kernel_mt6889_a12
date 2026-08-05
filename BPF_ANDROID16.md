# Android 16 BPF 兼容修补记录（realme X7 Pro / MT6889 / 4.14.186）

> 给下一个 AI 的完整工作记录：**Android 16 GSI 的 BPF 生态对 4.14 内核到底要求什么、为什么不需要大改、改了什么、怎么复现、验证结果如何**。
>
> 最后更新：2026-08-05
>
> 关联项目：DSU 攻坚（`dsu_work/DSU_PROGRESS.md`）、kernel_a12 主线（`kernel_a12/HANDOFF.md`）

---

## 0. 一句话总结

**Android 16（API 36 / 26Q1）的 BPF 用户态生态（bpfloader / netd / tethering / clatd / perfetto）在源码层面明确支持 4.14 内核——所有必需程序都有 `$4_9`/`$4_14` 降级变体，4.14 没有的内核特性（ringbuf、bpf_ktime_get_boot_ns、bpf_skb_load_bytes_relative、JMP32 指令）全部被版本门槛跳过或编译期裁剪。因此不需要 backport 任何 BPF helper/map 类型，真正的改动只有一个内核配置项：`CONFIG_DEBUG_FS=y`。**

产物：`boot_ksu_a12_nomagisk_v5_bpf.img`（v4 基础上仅加 DEBUG_FS，A12 真机验证零回归）。

---

## 1. 背景与目标

| 项 | 值 |
|---|---|
| 设备 | realme X7 Pro 5G（RMX2121CN / MT6889） |
| 当前系统 | ColorOS 12 / Android 12 |
| 目标 | 通过 DSU 启动 Android 16 GSI（见 dsu_work/DSU_PROGRESS.md） |
| 内核 | 4.14.186+（kernel_a12，KernelSU 3.2.2-legacy，clang 22.1.8） |
| 任务 | 修补内核 BPF 以支持 Android 16 |

A16 GSI 的 netbpfload 有 `reboot_on_failure`（netbpfload.rc），BPF 加载失败会直接重启设备，因此先于 DSU 攻坚把 BPF 侧摸清并保障到位。

---

## 2. 核心结论（先说结论，再给证据）

1. **A16 的 BPF 程序（netd.o / offload.o / clatd.o / dscpPolicy.o）在 4.14 上全部可加载**——AOSP 为 4.9/4.14/4.19 内核编译了专用变体，这是 Android T 时代 4.14 ACK 设备的官方升级路径。
2. **启动必需路径（critical）零 backport**；非必需路径（perfetto 采样等）部分程序因内核缺 raw_tp(4.17)/iter(5.5)/FUSE-BPF(5.12) 必然加载失败，但**只影响功能、不影响启动**。
3. **唯一必需的内核改动：`CONFIG_DEBUG_FS=y`**。4.14 没有 tracefs，libbpf attach tracepoint 程序要读 `/sys/kernel/debug/tracing/events`；不开则 perfetto 系采样全部失效（不崩系统但白装）。

---

## 3. 证据链（逐条源码/实物验证）

### 3.1 分析材料

- AOSP `system/bpf` android16-release 分支（Rust bpfloader + legacy Loader.cpp）
- AOSP `packages/modules/Connectivity` android16-release 分支（netd/clatd/offload/dscpPolicy BPF 程序）
- AOSP `build/soong` android16-release（BPF 编译 flags）
- A16 GSI 实物：`dsu_work/gsi/content/system/etc/bpf/*.bpf`（反汇编验证指令集）

### 3.2 程序侧：全部有 4.14 变体（netd.c / offload.c / clatd.c）

| 程序 | 4.14 用的变体 | 证据（源码） |
|---|---|---|
| cgroup ingress/egress 统计 | `cgroupskb/ingress\|egress/stats$4_9`（KVER_NONE–4.19） | 注释明确 "Android T 4.9 & T/U 4.14" |
| tether offload v4/v6 | `tether_*$4_14`（KVER_4_14–5.10） | 4.14 变体用 `NO_UPDATETIME` |
| clatd 464xlat | `clat_*$4_14` / `clat_*$4_9` | 4.14 用 `bpf_skb_adjust_room`+`bpf_skb_change_proto`（4.14 均有） |
| dscpPolicy | 单一版本 | helper 全是 4.14 已有 |

### 3.3 4.14 缺失特性的处理方式（AOSP 设计内降级）

| 内核特性 | 上游引入 | A16 程序如何处理 |
|---|---|---|
| `bpf_ktime_get_boot_ns` (helper) | 5.8 | offload 4.14 变体编译期常量 `NO_UPDATETIME`，调用被优化掉；注释："backported to all Android Common Kernel 4.14+ trees" |
| `BPF_MAP_TYPE_RINGBUF` | 5.8 | map 定义 `min_kver=KVER_5_10`，bpfloader 在 4.14 上**跳过创建**（Loader.cpp: `if (kvers < md[i].min_kver) skip`） |
| `bpf_ringbuf_reserve/output` | 5.8 | `do_packet_tracing()` 开头 `if (!KVER_IS_AT_LEAST(kver, 5, 10, 0)) return;` 编译期剪掉 |
| `bpf_skb_load_bytes_relative` | 4.17 | `bpf_skb_load_bytes_net()` 在 <4.19 用 `bpf_skb_load_bytes`（代价：wifi 流量判断偏移可能错，4.14 上为已知取舍） |
| JMP32 / ALU32 指令 | 4.16 / 4.10 | soong 编译 flags **无 `-mcpu=v3`**（libbpf_prog.go: `--target=bpf -O2`），生成 v1/v2 兼容指令；GSI 实物 .bpf 反汇编确认无 JMP32 |
| BTF（.BTF 段） | 5.1 | 程序带 .BTF，但 libbpf 对内核无 BTF 只打警告（"BTF is optional"），不阻断 |
| `prog_name` / `expected_attach_type` attr | 4.15 / 4.17 | Loader.cpp 显式 `if (isAtLeastKernelVersion(4,15,0))` 才填 prog_name；旧内核忽略未知 attr 字段 |
| `BPF_F_MMAPABLE` | 5.5 | libbpf 自动探测，不支持则降级不用 |

### 3.4 Android 16 的 bpfloader 架构（两个加载器）

```
netbpfload (APEX com.android.tethering)
├─ Rust loader (bpfloader.rs)：只加载 timeInState.bpf，FILE_ARR critical=false → 失败仅 error 日志，不 panic
└─ legacy loader (NetBpfLoad.cpp)：扫描 /apex/com.android.tethering/etc/bpf/{tethering,netd_shared,netd_readonly,net_shared,net_private}/*.o
   ├─ 按内核版本选 $变体（4.14 → $4_9/$4_14）
   ├─ 按 CRITICAL 段标记：critical 失败 → exit(121) 触发重启；非 critical → 跳过
   └─ 系统启动必需程序全部 4.14 可加载
```

### 3.5 A16 GSI 里 4.14 必然加载失败的程序（预期内，无需处理）

| 文件 | 程序段类型 | 需要内核 | 影响 |
|---|---|---|---|
| `kernelWakelockDuration.bpf` | `raw_tp/*` | 4.17+ | 电池统计采样缺失 |
| `timeInState.bpf` 部分 | `raw_tp/sched_process_free` | 4.17+ | 同上（其余 tracepoint 正常） |
| `dmabufIter.bpf` | `iter/*`（BPF_PROG_TYPE_TRACING） | 5.5+ | dmabuf 调试 |
| `fuseMedia.bpf` | `fuse/*`（FUSE-BPF） | 5.12+ | FUSE passthrough 性能增强缺失 |
| `gpuMem/gpuWork/bpfMemEvents.bpf` | `tracepoint/*` | 4.14 ✓ | **能加载** |

> 注：这些是组件（perfetto/lmkd 等）各自加载的，非 bpfloader critical 路径；加载失败仅对应功能降级。

### 3.6 内核配置现状核查（kernel_a12/.config）

已具备（无需动）：`CONFIG_BPF_SYSCALL / BPF_JIT / BPF_JIT_ALWAYS_ON / CGROUP_BPF / BPF_EVENTS / TRACEPOINTS / PERF_EVENTS / NET_CLS_BPF / NET_SCH_INGRESS / NET_CLS_ACT / NETFILTER_XT_MATCH_BPF / FS_VERITY` 全 =y。

缺失且必需：`CONFIG_DEBUG_FS`（原厂关闭）。

---

## 4. 实际改动

### 4.1 内核配置

```bash
# 方法一（本次实际使用）：直接改 .config
cp kernel_a12/.config kernel_a12/.config.bak_bpf
cd kernel_a12 && ./scripts/config --enable DEBUG_FS
make ARCH=arm64 CC=clang LD=ld.lld LLVM=1 \
  CLANG_FLAGS="-target aarch64-linux-gnu --prefix=/usr/bin/aarch64-linux-gnu- -no-integrated-as" \
  CROSS_COMPILE=aarch64-linux-gnu- olddefconfig

# 方法二（已同步到 defconfig，见提交）：arch/arm64/configs/k6889v1_64_defconfig
#   CONFIG_DEBUG_INFO=y 之后新增 CONFIG_DEBUG_FS=y
```

### 4.2 编译

```bash
nohup /home/dev/桌面/mt6889/build_a12_clang.sh > out/build_bpf_dbgfs.log 2>&1 &
# 产物：arch/arm64/boot/Image.gz-dtb（16,295,122 字节）+ dts/mediatek/mt6885.dtb（198,529 字节）
```

### 4.3 打包 boot.img（v5，基于 v4 材料）

```bash
# 1. 解 v4 取 ramdisk（v4 = boot_ksu_a12_nomagisk_v4_bbr.img，真机验证过的版本）
unpack_bootimg --boot_img boot_ksu_a12_nomagisk_v4_bbr.img --out /tmp/opencode/v4_u

# 2. mkbootimg（参数与 v4 header 完全一致）
mkbootimg --kernel kernel_a12/arch/arm64/boot/Image.gz-dtb \
  --ramdisk /tmp/opencode/v4_u/ramdisk \
  --dtb kernel_a12/arch/arm64/boot/dts/mediatek/mt6885.dtb \
  --base 0x40000000 --kernel_offset 0x80000 --ramdisk_offset 0x7c80000 \
  --tags_offset 0xbc80000 --dtb_offset 0xbc80000 --pagesize 2048 \
  --header_version 2 --os_version 12.0.0 --os_patch_level 2022-12 \
  --cmdline "bootopt=64S3,32N2,64N2 buildvariant=user" -o boot_bpf_v5_raw.img

# 3. AVB footer（设备已解锁 orange，keyless 足够，与 DSU 方案一致）
cp boot_bpf_v5_raw.img boot_bpf_v5.img
avbtool add_hash_footer --image boot_bpf_v5.img --partition_name boot \
  --partition_size 33554432 --algorithm NONE --rollback_index 0

# 4. 交付
cp boot_bpf_v5.img /home/dev/桌面/mt6889/boot_ksu_a12_nomagisk_v5_bpf.img
```

---

## 5. 验证结果（2026-08-05，A12 真机，v5 内核 #15）

### 5.1 验证命令

```bash
# 配置生效 + 挂载
adb shell 'zcat /proc/config.gz | grep CONFIG_DEBUG_FS'          # =y ✓
adb shell 'mount | grep -E "debug|trace"'                        # debugfs + tracefs 双挂载 ✓
# BPF pin 完整性（su）
adb shell su -c 'ls -l /sys/fs/bpf/'
# 网络健康
adb shell 'dumpsys netstats | head -30'; adb shell 'ping -c 2 8.8.8.8'
```

### 5.2 结果

- ✅ 内核：`Linux version 4.14.186+ ... #15 SMP PREEMPT Tue Aug 4 21:19:11 CST 2026`
- ✅ `CONFIG_DEBUG_FS=y` 生效，`debugfs on /sys/kernel/debug` + `tracefs on /sys/kernel/debug/tracing` 已挂载
- ✅ A12 全量 BPF 零回归：netd（cgroupskb in/out、xt_bpf×4、cgroupsock inet_create）、clatd×4、oplus 系全部 map/prog 正常 pin
- ✅ 网络正常（ping 通、netstats 有数据）
- ✅ 无 bootloop、无 CRITICAL 报错

> 注：`cat /sys/fs/bpf/map_xxx` 报 `No such device or address` 是正常的——bpffs 文件只支持 ioctl 不支持 read，能 ls 出来即加载成功。

### 5.3 A16 侧的待验证项（DSU 攻坚完成后）

```bash
adb shell su -c 'dmesg | grep -iE "BpfLoader|CRITICAL FAILURE|BPF_PROG_LOAD" | tail -50'
adb shell su -c 'ls /sys/fs/bpf/netd_shared/ /sys/fs/bpf/tethering/ 2>&1'
```

预期：无 `CRITICAL FAILURE LOADING BPF PROGRAMS`，`netd_shared/` 下出现 `map_netd_*`/`prog_netd_cgroupskb_*`。

---

## 6. 已知限制（4.14 无法覆盖的 A16 BPF 功能）

| 功能 | 原因 | 影响 | 若要支持 |
|---|---|---|---|
| perfetto 部分采样（raw_tp） | BPF_RAW_TRACEPOINT 4.17+ | kernelWakelockDuration 等失效 | backport raw_tp（4.17 bpf_trace.c 改动较深，不推荐） |
| dmabuf iter | iter/tracing prog type 5.5+ | 调试工具 | 不推荐 |
| FUSE-BPF passthrough | BPF_PROG_TYPE_FUSE 5.12+ | 媒体 passthrough 性能增强缺失，功能正常 | 不推荐 |
| wifi 流量统计精确度 | 无 bpf_skb_load_bytes_relative（4.17） | 4.14 变体用绝对偏移读取，rawip 准确、Ethernet 有偏差（AOSP 官方已知取舍） | 无法解决 |

以上均为**非启动必需**路径，不影响 A16 正常使用。

---

## 7. 产物清单

| 文件 | 说明 |
|---|---|
| `boot_ksu_a12_nomagisk_v5_bpf.img` | **本次交付**（33,554,432B，v4 内核 + DEBUG_FS + v4 ramdisk + keyless AVB） |
| `boot_ksu_a12_nomagisk_v4_bbr.img` | v4 基线（BBR/FQ，真机验证） |
| `kernel_a12/.config.bak_bpf` | 改配置前备份 |
| `out/build_bpf_dbgfs.log` | 编译日志 |
| `kernel_a12/arch/arm64/boot/Image.gz-dtb` | 新内核镜像 |

---

## 8. 复现速查

```bash
# 完整重跑一遍
cd /home/dev/桌面/mt6889/kernel_a12
./scripts/config --enable DEBUG_FS && make olddefconfig   # 或从 defconfig 起：make k6889v1_64_defconfig
nohup /home/dev/桌面/mt6889/build_a12_clang.sh > /home/dev/桌面/mt6889/out/build_bpf_dbgfs.log 2>&1 &
# 等编译完 → 按 §4.3 打包
```

---

## 9. 结论

- **不需要给 4.14 backport 任何 BPF 代码**——A16 的 BPF 生态设计上就兼容 4.14（Android T 时代 4.14 ACK 设备官方升级路径）。
- 唯一的必需改动 `CONFIG_DEBUG_FS=y` 已落地、已提交、已打包、A12 真机验证零回归。
- 下一步：DSU 刷 A16 GSI，验证 netbpfload（§5.3），之后回到 DSU_PROGRESS.md 的主线（一阶段挂载问题）。
