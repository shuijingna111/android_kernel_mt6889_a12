# 项目交接文档（kernel_a12：realme X7 Pro Android 12 + KernelSU）

> 给下一个人的完整说明：这个仓库是什么、怎么编译、怎么打包、踩过哪些坑。
>
> 最后更新：2026-08-03

---

## 1. 一句话总结

**把 realme X7 Pro（RMX2121 / MT6889 / 天玑 1000+）官方 Android 12 内核源码（4.14.186）改造为带 KernelSU root 的自编译内核，并打包出可刷机的 boot.img。**

- 手机系统：**Android 12**（ColorOS 12，os version 12.0.0，patch 2022-12）
- 内核版本：`Linux version 4.14.186+`，原厂用 clang 11.0.1 构建
- 源码：realme 官方 AndroidS（= Android 12）开源包，**未做任何 12→11 移植**（上一轮 12→11 适配项目已废弃，见旧仓库）

---

## 2. 仓库结构

| 路径 | 说明 |
|---|---|
| `kernel_a12/` | **本仓库**：Android 12 源码 + KernelSU 集成（可用） |
| `boot.img` | Android 11 原厂备份（保留） |
| `magisk_patched-30700_wWwyg.img` | Android 12 原厂内核 + Magisk ramdisk（可开机的恢复基准） |
| `boot_shouhou.img` | **售后包原厂 Android 12 boot**（纯原厂 ramdisk + AVB，打包母本） |
| `boot_ksu_a12_nomagisk.img` | **当前交付：KSU 内核 + 纯原厂 ramdisk** |
| `boot_ksu_a12.img` | 旧版：KSU 内核 + Magisk ramdisk（已验证开机） |
| `build_a12_clang.sh` | 编译脚本（clang 22.1.8） |

---

## 3. KernelSU 集成内容（基于原始 Android 12 源码的 3 个提交）

| 提交 | 内容 |
|---|---|
| `d95a1373f` | KernelSU v3.2.2-legacy submodule（rsuntk/KernelSU @ 648e5988）+ `drivers/kernelsu` 符号链接 + `.gitmodules` |
| `07914163c` | 4 个系统调用 hook（fs/exec.c、open.c、stat.c、read_write.c）+ 编译兼容修复（lib/string.c 加 stpcpy、sensor_devinfo weak stub）+ defconfig 调整 |
| `a9eeebb36` | `kernel-4.14 -> .` 自引用软链（MTK 头文件 `#include "../../../../kernel-4.14/..."` 路径要求） |

defconfig（`k6889v1_64_defconfig`）关键改动：

| 配置 | 值 | 原因 |
|---|---|---|
| `CONFIG_KSU=y` / `CONFIG_KSU_MANUAL_HOOK=y` | 开 | KernelSU |
| `CONFIG_KALLSYMS_ALL=y` | 开 | KSU 需要 |
| `CONFIG_IKHEADERS` | 关 | 绕开 gen_kheaders.sh 的 cpio 路径 bug |
| `CONFIG_CC_STACKPROTECTOR_STRONG` | 关 | clang 22 检测误判 |
| `CONFIG_SECTION_MISMATCH_WARN_ONLY=y` | 开 | modpost 只警告 |
| `CONFIG_MTK_TINYSYS_SSPM_VERSION=""` | 空 | 与原厂一致（mt6885 自动选 v2） |
| `CONFIG_MTK_EMI=y` | 开 | 提供 emi_mpu_set_protection（原厂关，编译必需） |

---

## 4. 编译方法

### 环境
- clang + lld（Arch: `sudo pacman -S clang llvm lld`），不要用 GCC
- cpio

### 命令
```bash
cd /home/dev/桌面/mt6889/kernel_a12
make ARCH=arm64 k6889v1_64_defconfig
sh /home/dev/桌面/mt6889/build_a12_clang.sh
# 产物：arch/arm64/boot/Image.gz（15.7MB）、vmlinux（296MB）
```

### 坑
1. 缺 `kernel-4.14` 软链 → `oplus_battery_mtk6889R.h` 报 `tcpm.h file not found`
2. KCFLAGS 必须带几十个 `-I` include 路径（MTK 驱动头文件分散）
3. 换 clang 版本/配置后建议删旧 .o 强制重编：`find drivers -name "*.o" -delete`

---

## 5. 打包方法（Android 12 布局，与 A11 不同！）

### ⚠️ 关键差异：A12 boot 是「纯 Image.gz 内核段 + 独立 dtb 段」

| | Android 11 | Android 12 |
|---|---|---|
| 内核段 | Image.gz-dtb（DTB 附加在内核后） | **纯 Image.gz**（已解压验证无 DTB） |
| DTB | — | **独立 dtb 段**（198617 字节） |

### 正确命令（母本 = `boot_shouhou.img` 售后包原厂）

```bash
mkbootimg --kernel kernel_a12/arch/arm64/boot/Image.gz \
  --ramdisk stockboot/out/ramdisk \
  --dtb stockboot/out/dtb \
  --base 0x40000000 --kernel_offset 0x80000 --ramdisk_offset 0x7c80000 \
  --tags_offset 0xbc80000 --dtb_offset 0xbc80000 \
  --pagesize 2048 --header_version 2 \
  --os_version 12.0.0 --os_patch_level 2022-12 \
  --cmdline "bootopt=64S3,32N2,64N2 buildvariant=user" \
  -o raw.img

# 然后（python 脚本）：
# 1. padding 到 33554432（32MB）
# 2. 从原厂 boot 复制 AVB0 vbmeta blob（1600 字节）到内容末尾（页对齐）
# 3. 末尾写 AVBf footer（orig_size / vbmeta_off / vbmeta_size，大端）
```

### 验证要点
- magic `ANDROID!`、kernel_size = Image.gz 大小
- vbmeta 与原厂**逐字节一致**、footer 存在
- 全部 header 字段除 kernel_size 外与原厂一致

---

## 6. 关键 bug 修复记录（不开机的真凶）

### 现象
刷入后：解锁警告画面 → 启动约 2 秒 → panic 重启循环。

### 根因：clang 的 `.text.unlikely.` 段名（带尾点）与 MTK 链接脚本不匹配

- KernelSU 的 `ksu_install_rc_hook()` 带 `__attribute__((cold))`
- GCC 生成段名 `.text.unlikely`（无尾点）→ 链接脚本匹配 → 进主 .text
- **clang 生成 `.text.unlikely.`（带尾点）→ 不匹配 → 变成孤儿段 → 链接器把它放进 `__init` 区**
- 内核启动完成时 `free_initmem()` 释放 init 内存（日志：`Freeing unused kernel memory: 8000K`）
- 之后内核执行第一个用户程序（读 init.rc）时，`ksu_handle_vfs_read()` 调用 `ksu_install_rc_hook()` → **跳到已释放的内存 → IABT panic**

```
pc : ksu_install_rc_hook+0x0/0xdf0     ← 未映射地址 ffffff82556fa210
Call trace: ksu_install_rc_hook → vfs_read → kernel_read → prepare_binprm → do_execveat_common → kernel_init
```

### 修复
`KernelSU/kernel/runtime/ksud.c:372`：删掉 `__attribute__((cold))`（保留 `static noinline`），函数进入普通 `.text` 段。验证：`readelf -s vmlinux` 显示 `ksu_install_rc_hook` 在 section 2（.text）。

### 排查过程（教训）
1. 对比原厂内核配置（extract-ikconfig 从 magisk 镜像内核提取）：与我们的 .config **同键零差异** → 排除配置问题
2. 抓日志：刷 KSU 版让它 panic 2-3 次 → 刷回 Magisk 版开机 → `adb shell su -c "cat /proc/last_kmsg"`（MTK RAM console，`CONFIG_MTK_RAM_CONSOLE=y`）→ 拿到完整 panic

---

## 7. 镜像清单与状态（2026-08-03）

| 文件 | 说明 | 状态 |
|---|---|---|
| `boot_shouhou.img` | 售后包**原厂 A12 boot**（母本，永远保留） | 原厂 |
| `boot_ksu_a12_nomagisk.img` | **KSU 内核 + 纯原厂 ramdisk（无 Magisk）** | ✅ 当前交付，待装 Manager 验证 root |
| `boot_ksu_a12.img` | KSU 内核 + Magisk ramdisk | ✅ 已验证开机（有 verified boot 警告） |
| `magisk_patched-30700_wWwyg.img` | 原厂内核 + Magisk ramdisk | ✅ 恢复基准 |

> 开机警告「您的设备内部出现了问题」= 解锁 bootloader（verifiedbootstate=orange）+ veritymode=enforcing 的**标准警告**，与内核无关（dm-verity 配置与原厂完全一致），点掉即可。

---

## 8. 下一步

- [ ] 刷 `boot_ksu_a12_nomagisk.img` 验证纯 KSU 开机
- [ ] 安装 KernelSU Manager（**必须用 rsuntk/KernelSU 配套版本**，https://github.com/rsuntk/KernelSU/releases，v3.2.2 legacy 协议；KernelSU-Next 的 Manager 协议不兼容）
- [ ] Manager 注入 ksud 到 ramdisk（借助 Magisk 或直接补丁 boot）→ 验证 `adb shell su -c id` 返回 uid=0
- [ ] （可选）换用 KernelSU-Next（社区主流，4k⭐），需要重新移植 + 编译

---

## 9. 常见命令速查

```bash
# 编译
make ARCH=arm64 k6889v1_64_defconfig && sh /home/dev/桌面/mt6889/build_a12_clang.sh

# 抓 panic 日志
adb shell su -c "cat /proc/last_kmsg" > last_kmsg.txt

# 解包 boot
unpack_bootimg --boot_img boot_shouhou.img --out out

# 推送
git add -A && git commit -m "..." && git push origin master
```

---

*交接人：shuijingna111（2026-08-03）*
