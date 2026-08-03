# android_kernel_mt6889_4.14 (Android 12 + KernelSU)

realme X7 Pro（RMX2121 / MT6889 / Dimensity 1000+）**Android 12** 自编译内核，集成 **KernelSU root**，提供可刷机的 boot.img。

- 内核版本：`4.14.186+`（Android 12 / ColorOS 12，os version 12.0.0，patch 2022-12）
- 当前状态：✅ 已编译通过、✅ 真机验证能开机、✅ **无任何警告弹窗**（VINTF 检测已修复）、root 激活见下方说明

> 注意：本仓库是 Android 12 版本。旧的 Android 12→11 移植版已废弃并删除。

---

## 代码来源（完全开源）

| 组件 | 来源 | 说明 |
|---|---|---|
| 内核源码 | [realme-kernel-opensource/realme_X7...AndroidS-kernel-source](https://github.com/realme-kernel-opensource/realme_X7_X7Pro_X7ProExtreme_X7-5G_Q2Pro_V15_V5_Q2_Narzo30pro-5G_7-5G-AndroidS-kernel-source.git) | realme 官方开源（Android S = Android 12），base commit `f0c2afc4d`，未改动官方代码逻辑 |
| KernelSU | [rsuntk/KernelSU](https://github.com/rsuntk/KernelSU)（[tiann/KernelSU](https://github.com/tiann/KernelSU) 的 fork，GPL-3.0） | `v3.2.2-legacy` 分支，commit `648e5988`，4.14 老内核兼容版 |
| 打包参考 | `boot_shouhou.img`（realme 售后包原厂 Android 12 boot.img） | ramdisk/dtb/vbmeta/footer 全部取自原厂，未修改 |

## 本仓库的改动

基于 base commit `f0c2afc4d` 之上的提交：

| 提交 | 内容 |
|---|---|
| `d95a1373f` | 集成 KernelSU v3.2.2-legacy（submodule + `drivers/kernelsu` 软链） |
| `07914163c` | KernelSU 系统调用 hook（fs/exec.c、open.c、stat.c、read_write.c）+ 编译兼容修复（`lib/string.c` 补 stpcpy、sensor_devinfo weak stub）+ defconfig 调整 |
| `a9eeebb36` | `kernel-4.14 -> .` 自引用软链（MTK 头文件路径要求） |
| `c5626c215` | **修复 VINTF 弹窗**：开启 `CONFIG_CC_STACKPROTECTOR_STRONG=y` |
| `f3836d61f` | 本文档 + [HANDOFF.md](HANDOFF.md) |

### ⚠️ 必读 1：KernelSU 的冷属性修复（编译前必须做）

KernelSU 的 `kernel/runtime/ksud.c:372` 中 `ksu_install_rc_hook()` 原带 `__attribute__((cold))`。
GCC 生成段名 `.text.unlikely`，而 **clang 生成 `.text.unlikely.`（带尾点）**，MTK 链接脚本不匹配该孤儿段，函数被链接进 `__init` 区，内核启动后随 init 内存被释放 → **首次执行用户程序必 panic**（IABT 跳入未映射内存）。

**修复**：删掉 `__attribute__((cold))`（保留 `static noinline`），函数即进入普通 `.text` 段。详细分析见 [HANDOFF.md §6](HANDOFF.md)。

### ⚠️ 必读 2：`CONFIG_CC_STACKPROTECTOR_STRONG` 必须开启

realme/OPPO 系统启动时执行 **VINTF 内核配置兼容性检查**（`/system/etc/vintf/compatibility_matrix.*.xml` 要求 4.14 内核 `CONFIG_CC_STACKPROTECTOR_STRONG=y`）。
**关闭它会导致锁屏弹「您的设备内部出现了问题。请联系您的设备制造商了解详情。」**（defconfig 已默认开启，勿关闭）。详细排查过程见 [HANDOFF.md §7](HANDOFF.md)。

---

## 编译

```bash
# 环境：clang + lld（Arch: sudo pacman -S clang llvm lld），不要用 GCC；需要 cpio
git clone --recurse-submodules https://github.com/shuijingna111/android_kernel_mt6889_a12.git
cd android_kernel_mt6889_a12
ln -sfn . kernel-4.14    # 必须：MTK 头文件引用 ../../../../kernel-4.14/...
make ARCH=arm64 k6889v1_64_defconfig

# KCFLAGS 需要几十个 -I 路径（MTK 驱动头文件分散），见 build_a12_clang.sh
sh build_a12_clang.sh
# 产物：arch/arm64/boot/Image.gz（约 15.9MB）、vmlinux（约 300MB）
```

> 注意：`ksud.c` 的冷属性修复在工作树中（submodule 无法提交），clone 后需手动修改（见必读 1）。

## 打包 boot.img（Android 12 布局）

> Android 12 与 Android 11 不同：内核段是**纯 Image.gz**，DTB 在**独立的 dtb 段**（198617 字节），不能像 A11 那样用 Image.gz-dtb。

```bash
mkbootimg --kernel arch/arm64/boot/Image.gz \
  --ramdisk <原厂 ramdisk> \
  --dtb <原厂 dtb> \
  --base 0x40000000 --kernel_offset 0x80000 --ramdisk_offset 0x7c80000 \
  --tags_offset 0xbc80000 --dtb_offset 0xbc80000 \
  --pagesize 2048 --header_version 2 \
  --os_version 12.0.0 --os_patch_level 2022-12 \
  --cmdline "bootopt=64S3,32N2,64N2 buildvariant=user" \
  -o raw.img
```

之后：padding 到 32MB（33554432）→ 从原厂 boot 复制 `AVB0` vbmeta blob → 末尾写 `AVBf` footer（大端，`orig_size`/`vbmeta_off`/`vbmeta_size`）。完整脚本与验证方法见 [HANDOFF.md §5](HANDOFF.md)。

---

## 刷机

```bash
fastboot flash boot boot_ksu_a12_nomagisk_v3.img
```

| 镜像 | 说明 |
|---|---|
| `boot_ksu_a12_nomagisk_v3.img` | **当前交付**：KSU 内核 + 纯原厂 ramdisk，VINTF 修复（无弹窗） |
| `boot_shouhou.img` | 售后包原厂 A12 boot（恢复用母本） |
| `magisk_patched-30700_wWwyg.img` | 原厂内核 + Magisk ramdisk（恢复基准） |

## 激活 root（KernelSU Manager）

1. 安装 **KernelSU Manager（rsuntk legacy 配套版）**：https://github.com/rsuntk/KernelSU/releases （v3.2.2 legacy 协议；KernelSU-Next / ReSukiSU 的 Manager **不兼容**）
2. 打开 Manager → 它会把 `ksud` 注入到 `/data/adb/ksud` → 重启
3. 验证：`adb shell su -c id` 应返回 `uid=0`

---

## License

- 内核源码：GPL-2.0（realme/MTK 官方开源）
- KernelSU：GPL-3.0（[rsuntk/KernelSU](https://github.com/rsuntk/KernelSU) / [tiann/KernelSU](https://github.com/tiann/KernelSU)）

刷机有风险，请自行备份原厂 boot.img。
