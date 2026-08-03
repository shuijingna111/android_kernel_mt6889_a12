# 项目交接文档（kernel_a12：realme X7 Pro Android 12 + KernelSU）

> 给下一个人的完整说明：这个仓库是什么、怎么编译、怎么打包、踩过哪些坑。
>
> 最后更新：2026-08-03（VINTF 弹窗已修复，v3 镜像真机验证通过）

---

## 1. 一句话总结

**把 realme X7 Pro（RMX2121 / MT6889 / 天玑 1000+）官方 Android 12 内核源码（4.14.186）改造为带 KernelSU root 的自编译内核，并打包出可刷机的 boot.img。**

- 手机系统：**Android 12**（ColorOS 12，os version 12.0.0，patch 2022-12，build `Q.cadd1b_557`）
- 内核版本：`Linux version 4.14.186+`，原厂用 clang 11.0.1（2022-12-12 构建）构建
- 源码：realme 官方 AndroidS（= Android 12）开源包，**未做任何 12→11 移植**（上一轮 12→11 适配项目已废弃删除）

**当前状态：✅ 能开机、无任何警告弹窗、KSU 已集成（root 激活见 §8）**

---

## 2. 仓库结构与镜像清单

| 路径/文件 | 说明 |
|---|---|
| `kernel_a12/` | **本仓库**：Android 12 源码 + KernelSU 集成 |
| `boot_shouhou.img` | 售后包**原厂 Android 12 boot**（打包母本：ramdisk/dtb/vbmeta/AVB 均取自它） |
| `magisk_patched-30700_wWwyg.img` | 原厂内核 + Magisk ramdisk（**可开机的恢复基准**，备份勿删） |
| **`boot_ksu_a12_nomagisk_v3.img`** | **✅ 当前交付：KSU 内核 + 纯原厂 ramdisk + VINTF 修复（真机验证无弹窗）** |
| `boot_ksu_a12.img` | KSU 内核 + Magisk ramdisk（能开机但有 VINTF 弹窗，已废弃） |
| `boot_ksu_a12_nomagisk.img` / `_v2.img` | 早期版（VINTF 未修复，会弹窗，已废弃） |
| `boot.img` | Android 11 原厂备份（保留） |
| `build_a12_clang.sh` | 编译脚本（clang 22.1.8） |

---

## 3. KernelSU 集成内容（基于原始 Android 12 源码的 4 个提交）

| 提交 | 内容 |
|---|---|
| `d95a1373f` | KernelSU v3.2.2-legacy submodule（rsuntk/KernelSU @ 648e5988）+ `drivers/kernelsu` 符号链接 + `.gitmodules` |
| `07914163c` | 4 个系统调用 hook（fs/exec.c、open.c、stat.c、read_write.c）+ 编译兼容修复（lib/string.c 加 stpcpy、sensor_devinfo weak stub）+ defconfig 调整 |
| `a9eeebb36` | `kernel-4.14 -> .` 自引用软链（MTK 头文件 `#include "../../../../kernel-4.14/..."` 路径要求） |
| `c5626c215` | **修复 VINTF 弹窗：开启 `CONFIG_CC_STACKPROTECTOR_STRONG=y`**（详见 §7） |

defconfig（`k6889v1_64_defconfig`）关键改动：

| 配置 | 值 | 原因 |
|---|---|---|
| `CONFIG_KSU=y` / `CONFIG_KSU_MANUAL_HOOK=y` | 开 | KernelSU |
| `CONFIG_KALLSYMS_ALL=y` | 开 | KSU 需要 |
| `CONFIG_CC_STACKPROTECTOR_STRONG=y` | **开（必须）** | **VINTF 兼容性矩阵要求，不开会弹"设备内部出现问题"** |
| `CONFIG_IKHEADERS` | 关 | 绕开 gen_kheaders.sh 的 cpio 路径 bug |
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
# 产物：arch/arm64/boot/Image.gz（约 15.9MB）、vmlinux（约 300MB）
```

### 坑
1. 缺 `kernel-4.14` 软链 → `oplus_battery_mtk6889R.h` 报 `tcpm.h file not found`（`ln -sfn . kernel-4.14`）
2. KCFLAGS 必须带几十个 `-I` include 路径（MTK 驱动头文件分散，见 build_a12_clang.sh）
3. 换 clang 版本/配置后建议删旧 .o 强制重编：`find drivers -name "*.o" -delete`
4. `CONFIG_CC_STACKPROTECTOR_STRONG=y` 用 clang 22 编译**没有**问题（此前"clang 误判"的担忧是错的）

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
```

然后（python 脚本）：
1. padding 到 33554432（32MB）
2. 从原厂 boot 复制 `AVB0` vbmeta blob（1600 字节）到内容末尾（页对齐）
3. 末尾写 `AVBf` footer（大端：`orig_size` / `vbmeta_off` / `vbmeta_size`）

### 验证要点
- magic `ANDROID!`、kernel_size = Image.gz 大小
- vbmeta 与原厂**逐字节一致**、footer 存在
- header 字段除 kernel_size 外与原厂一致

---

## 6. 关键 bug 修复记录（一）：KSU 冷属性 panic（不开机的真凶）

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

> ⚠️ **这个修复在 KernelSU submodule 工作树里，不在 git 提交里**（submodule 是 gitlink）。别人 clone 后编译前必须手动改这一行，否则必 panic。

---

## 7. 关键 bug 修复记录（二）：VINTF 弹窗"您的设备内部出现了问题"（真凶）

### 现象
能开机，但锁屏弹系统警告「您的设备内部出现了问题。请联系您的设备制造商了解详情。」（每开必弹，点掉不影响使用）

### 排查过程（完整记录）

1. **排除"解锁警告"理论**：弹窗出现时 `ro.boot.verifiedbootstate=orange` + `ro.boot.veritymode=enforcing`（解锁 BL 的标准状态），但**同样状态的原厂/Magisk 开机不弹** → 不是解锁状态问题
2. **排除 ramdisk 版本理论**：售后包 ramdisk（11:02 构建）vs 手机系统（14:33 构建）——最初怀疑版本不匹配，但 **KSU 内核 + Magisk ramdisk（与手机同版本）也弹** → 问题在**内核**
3. **定位弹窗创建者**：
   - `dumpsys window windows` → 找到 `Window{... Android 系统}`，类型 `ty=SYSTEM_ERROR`
   - 窗口 Session 显示 PID 1454 / UID 1000 = **system_server**
   - token 名含 `OplusViewRootImplHooks`（ColorOS 框架机制）
   - `uiautomator dump` 确认文案为 `android:id/message` 标准 AlertDialog
4. **找到根因线索**（酷安帖子启发）：VINTF **设备兼容性矩阵检测**——系统启动时把内核配置与 `/system/etc/vintf/compatibility_matrix.*.xml` 中声明的内核要求逐条对比，不匹配就弹此提示
5. **实测验证**：
   - 从手机拉取 `compatibility_matrix.3/4/5/6.xml`
   - 解析所有 `<kernel version="4.14.*">` 段的 `<config>` 要求（共 737 条 4.14 条目）
   - 与我们的 `.config` 逐条比对 → **唯一真实不匹配：`CONFIG_CC_STACKPROTECTOR_STRONG` 要求 =y，实际 =n**（4.14.105 和 4.14.180 两个 level 都要求）
   - 原厂配置是 `=y`；是上一个 AI 为绕 clang 22 编译问题关闭的

### 修复
`k6889v1_64_defconfig` 开启：
```
CONFIG_CC_STACKPROTECTOR=y
CONFIG_CC_STACKPROTECTOR_STRONG=y
```
- clang 22 编译**无任何问题**（"clang 误判"不成立）
- 修复后 VINTF 检查脚本跑：**4.14 全部要求通过** ✅
- 真机验证：锁屏无弹窗 ✅

### 排查工具速查
```bash
# 拉 VINTF 矩阵
adb pull /system/etc/vintf/compatibility_matrix.5.xml .

# 提取 4.14 内核要求并与 .config 对比（python 脚本，见 HANDOFF 历史版本或重新写）
# 对比规则：要求 n = 配置为 n 或未设置；要求 y/字符串 = 必须相等（.config 值要去掉引号）

# 抓弹窗窗口
adb shell dumpsys window windows | grep -B2 -A25 'Android 系统'
adb shell uiautomator dump /sdcard/ui.xml && adb shell cat /sdcard/ui.xml
```

---

## 8. root 激活（KernelSU Manager）

内核 KSU 模块已生效，但 root 还需要 ksud 用户态进程：

1. **卸载不兼容的 Manager**：`com.resukisu.resukisu`（ReSukiSU v4.1.0，协议不兼容）
2. **安装配套 Manager**：`下载/KernelSU_v3.2.2-10-legacy-42-g915b4872_32490-release.apk`（rsuntk/KernelSU 的 legacy 配套版，包名 `me.weishu.kernelsu` 在内核白名单）
3. 打开 Manager → 会借助内核授权把 `ksud` 拷到 `/data/adb/ksud` → 重启
4. 验证：`adb shell su -c id` 应返回 `uid=0`

> 内核的 ksud 机制：ksud 二进制位于 `/data/adb/ksud`（`KSUD_PATH`），由内核注入的 sepolicy 规则（`apply_kernelsu_rules`）让 init 以 `post-fs-data`/`services` 事件启动它。ro.ksu.* 属性为空 = ksud 未运行。

### KSU 相关知识点（排查时用）
- IOCTL 协议：`KSU_IOCTL_*` 命令号 1-20 与 ReSukiSU 完全一致（20 之后 ReSukiSU 多了 21/100 号，不影响核心功能）
- `CONFIG_KSU_FEATURE_ADBROOT=y` 的 adb_root 功能**默认关闭**，需 Manager 通过 supercall 开启
- `dmesg` 被 `kernel.dmesg_restrict` 限制（无 root 读不了）；`/proc/last_kmsg`（MTK RAM console）有 root 可读

---

## 9. 下一步（待办）

- [ ] 安装 KernelSU legacy Manager → 注入 ksud → 验证 `su` root
- [ ] 更新 README 的镜像状态
- [ ] （可选）换用 KernelSU-Next（社区主流，4k⭐），需重新移植 + 编译

---

## 10. 常见命令速查

```bash
# 编译
make ARCH=arm64 k6889v1_64_defconfig && sh /home/dev/桌面/mt6889/build_a12_clang.sh

# 抓 panic 日志（需 root）
adb shell su -c "cat /proc/last_kmsg" > last_kmsg.txt

# 解包 boot
unpack_bootimg --boot_img boot_shouhou.img --out out

# VINTF 弹窗排查
adb pull /system/etc/vintf/compatibility_matrix.5.xml .

# 推送
git add -A && git commit -m "..." && git push origin master
```

---

*交接人：shuijingna111（2026-08-03）*
