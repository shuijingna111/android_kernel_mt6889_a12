# 项目交接文档（kernel_a12：realme X7 Pro Android 12 + KernelSU）

> 给下一个 AI 的完整交接：**这个项目干了什么、为什么、怎么复现**。
>
> 最后更新：2026-08-03

---

## 1. 这个项目干了什么（一句话）

**把 realme X7 Pro（RMX2121 / MT6889）官方 Android 12 内核源码（4.14.186）加上 KernelSU root，修掉两个会导致不开机/弹窗的 bug，开启 BBR 拥塞控制，编译并打包出可刷机的 boot.img（v3 真机验证通过：能开机、无弹窗；v4 在 v3 基础上默认 BBR，待真机复验）。**

- 手机：realme X7 Pro 5G（RMX2121CN），ColorOS 12（Android 12，build `Q.cadd1b_557`，2022-12-12）
- 内核：`Linux version 4.14.186+`，原厂 clang 11.0.1 构建
- 源码：realme 官方开源 AndroidS（= Android 12）包，base commit `f0c2afc4d`
- **不做什么**：不做 12→11 移植（上一个 AI 做的那个已废弃删除，因为它适配的是 Android 11，现在手机已是 Android 12）

---

## 2. 做了什么（完整工作记录）

### 2.1 集成 KernelSU（提交 d95a1373f / 07914163c）

| 改动 | 位置 | 干嘛的 |
|---|---|---|
| KernelSU submodule | `KernelSU/`（rsuntk/KernelSU v3.2.2-legacy @ 648e5988，官方 tiann/KernelSU 的 4.14 兼容 fork） | 提供 root 的内核模块 |
| 符号链接 | `drivers/kernelsu -> ../KernelSU/kernel` | 挂进内核构建树 |
| 构建挂接 | `drivers/Kconfig` + `drivers/Makefile` | 让 `CONFIG_KSU` 生效 |
| 4 个系统调用 hook | `fs/exec.c`（execveat）、`fs/open.c`、`fs/stat.c`、`fs/read_write.c`（vfs_read） | KSU 要求的手动 hook，`#ifdef CONFIG_KSU` 包住 |
| defconfig 开启 | `CONFIG_KSU=y`、`CONFIG_KSU_MANUAL_HOOK=y`、`CONFIG_KALLSYMS_ALL=y` | 启用 KSU |
| 编译兼容修复 | `lib/string.c` 补 `stpcpy`（clang 需要）；`sensor_devinfo.c` weak stub 补参数 | 老代码过 clang 22 编译 |

### 2.2 编译兼容处理（提交 a9eeebb36）

| 改动 | 干嘛的 |
|---|---|
| `kernel-4.14 -> .` 自引用软链 | MTK 驱动头文件写死 `#include "../../../../kernel-4.14/drivers/..."`，源码目录名必须是 kernel-4.14，用软链欺骗 |
| defconfig 关 `CONFIG_IKHEADERS` | 绕开 gen_kheaders.sh 的 cpio 路径 bug |
| defconfig 开 `CONFIG_SECTION_MISMATCH_WARN_ONLY=y` | modpost 致命错误降级为警告 |
| defconfig 开 `CONFIG_MTK_EMI=y` | 提供 `emi_mpu_set_protection`（原厂是关的，但编译必需） |
| `CONFIG_MTK_TINYSYS_SSPM_VERSION=""` | 与原厂一致（让 mt6885/Makefile 自动选 v2） |

### 2.3 修 bug 一：不开机 panic（冷属性段名问题）

- **现象**：启动约 2 秒 panic 无限重启
- **原因**：KSU 的 `ksu_install_rc_hook()` 带 `__attribute__((cold))`。GCC 生成段名 `.text.unlikely`，**clang 生成 `.text.unlikely.`（多一个尾点）**，MTK 链接脚本不认这个孤儿段 → 被放进 `__init` 区 → 内核启动后 init 内存被释放 → 执行首个用户程序时跳到已释放内存 → IABT panic
- **修复**：`KernelSU/kernel/runtime/ksud.c:372` 删掉 `__attribute__((cold))`（保留 `static noinline`）
- ⚠️ 此修复在 submodule 工作树里（gitlink 提交不了），**别人 clone 后编译前必须手动改**

### 2.4 修 bug 二：锁屏弹"您的设备内部出现了问题"（VINTF 检测）

- **现象**：能开机，但锁屏弹窗「您的设备内部出现了问题。请联系您的设备制造商了解详情。」（每开必弹）
- **排查过程**：
  1. 不是解锁警告（同样的 orange+enforcing 状态，原厂/Magisk 开机不弹）
  2. 不是 ramdisk 版本问题（换同版本 Magisk ramdisk 也弹）
  3. `dumpsys window windows` 定位弹窗：`ty=SYSTEM_ERROR`、uid 1000、system_server（PID 1454）创建
  4. 酷安帖子启发：**VINTF 设备兼容性矩阵检测**——开机时系统把内核配置和 `/system/etc/vintf/compatibility_matrix.*.xml` 里对 4.14 内核的要求逐条对比
  5. 拉取手机上的 4 个矩阵文件（3/4/5/6），解析全部 `<kernel version="4.14.*">` 的 `<config>` 要求（737 条），与我们的 `.config` 逐条对比
  6. **唯一真实不匹配：`CONFIG_CC_STACKPROTECTOR_STRONG` 要求 =y，我们是 =n**（上一个 AI 为绕 clang 22 编译问题关的，原厂是开的）
- **修复**（提交 c5626c215）：defconfig 开启 `CONFIG_CC_STACKPROTECTOR=y` + `CONFIG_CC_STACKPROTECTOR_STRONG=y`——clang 22 编译实测没问题（之前的担忧不成立）
- **验证**：VINTF 对比脚本全部通过 + 真机锁屏无弹窗 ✅

### 2.5 开启 BBR 拥塞控制（2026-08-03）

- defconfig 增加 `CONFIG_TCP_CONG_BBR=y`（内核自带 4.14 官方 BBR v1）+ `CONFIG_NET_SCH_FQ=y`（FQ qdisc，官方推荐搭配）+ `CONFIG_DEFAULT_BBR=y`（默认拥塞控制改为 bbr）
- ⚠️ 默认拥塞控制要写 **`CONFIG_DEFAULT_BBR=y`**（Kconfig choice 符号），直接写 `CONFIG_DEFAULT_TCP_CONG="bbr"` 会被忽略
- **默认 qdisc 也改为 FQ（编译期内置）**：`net/sched/sch_generic.c:35` 的 `default_qdisc_ops` 从 `&pfifo_fast_ops` 改为 `&fq_qdisc_ops`；该树的 sch_fq.c 里 ops 名是 `fq_qdisc_ops`（非上游 `fq_ops`）且原为 static，需去掉 `static` 并 `EXPORT_SYMBOL`
- **编掉 BIC**：`# CONFIG_TCP_CONG_BIC is not set`（defconfig）+ 删 `drivers/misc/mediatek/Kconfig.default` 的 `select TCP_CONG_BIC`——OPPO 的 `networksetting.rc` 在 early-init 写死 `write /proc/sys/net/ipv4/tcp_congestion_control bic`，BIC 不存在后该写入失败，默认保持 bbr
- 验证：`tcp_bbr_cong_ops`、`fq_qdisc_ops` 在 System.map；default_qdisc_ops 的 R_AARCH64_RELATIVE 重定位 addend = fq_qdisc_ops 地址；编译/打包/AVB 均通过

### 2.6 修复 wifi/热点完全不能用（模块签名+CRC 校验，2026-08-04）

- **现象**：自 v1 起所有 vendor 模块（wlan/bt/fm/gps/conninfra）都加载不了 → wlan0 不存在 → wifi/热点全挂（`persist.sys.oplus.wifi.fail.count` 累计 53 次）
- **排查**：conninfra_loader 循环报 `Can't open device node(/dev/conninfra_dev)`；手 `insmod conninfra.ko` 报 `Required key not available`
- **根因一**：defconfig 里 `CONFIG_MODULE_SIG_FORCE=y`（OPLUS_FEATURE_SECURITY_COMMON 段），自编译内核没有厂商签名公钥 → 所有签名模块被拒。修复：defconfig 该段全部置 `# CONFIG_MODULE_SIG... is not set`
- **根因二**：关掉签名后报 `disagrees about version of symbol module_layout` —— MTK Kconfig 强制 `select MODVERSIONS`（defconfig 关不掉），我们配置与原厂不同导致 module_layout CRC 不一致。修复：`kernel/module.c` 的 `check_version()` 和 `check_modstruct_version()` 直接返回 1（跳过所有符号 CRC 校验），并删除不再使用的 `resolve_rel_crc()`
- ⚠️ `check_version` 里删掉了 `try_to_force_load` 的唯一调用者，该函数仍被 vermagic 分支使用（`module.c:3041`），保留
- 验证：真机 `lsmod` 全部模块加载、`wlan0/wlan1/ap0` 出现、wifi 扫描正常、热点开启且有设备连接、流量经 ap0→NAT→ccmni1 全通、0 丢包 0 panic
- 安全取舍：模块签名校验与 CRC 校验均跳过（自编译内核常规做法，模块仍要求 vermagic 匹配）

### 2.7 打包（boot_ksu_a12_nomagisk_v4_bbr.img，当前交付）

- 母本：`boot_shouhou.img`（售后包原厂 A12 boot）——ramdisk、dtb、AVB vbmeta、footer 全部取自它，逐字节未改
- 内核段：**纯 Image.gz**（A12 与 A11 不同！A11 是 Image.gz-dtb 带附加 DTB，A12 是独立 dtb 段）
- 组装：mkbootimg（header v2、os_version 12.0.0/2022-12、原厂 cmdline）→ padding 32MB → 复制原厂 AVB0 vbmeta → 末尾写 AVBf footer（大端字段）
- 已验证：magic/kernel_size/ramdisk/vbmeta 逐字节正确、AVB 通过（bootloader 接受）

### 2.8 交付与发布

- GitHub：https://github.com/shuijingna111/android_kernel_mt6889_a12 （公开，README + 本 HANDOFF）
- 旧 GitHub 仓库 `android_kernel_mt6889_4.14`（12→11 废弃版）**未删**（gh token 缺 delete_repo 权限）

---

## 3. 当前状态与镜像清单

| 文件 | 状态 | 说明 |
|---|---|---|
| **`boot_ksu_a12_nomagisk_v4_bbr.img`** | ✅ **当前交付**（真机验证：开机+无弹窗、wifi/热点正常、BBR 默认+无 BIC） | KSU 内核（默认 bbr、FQ、模块加载修复）+ 纯原厂 ramdisk，md5 `1b5de839...` |
| `boot_ksu_a12_nomagisk_v3.img` | 上一版 | 同 v4 但默认 BIC，md5 `059c83f2...` |
| `boot_shouhou.img` | 原厂母本（永远保留） | 售后包 A12 boot，md5 `cfb90f82...` |
| `magisk_patched-30700_wWwyg.img` | 恢复基准（在桌面） | 原厂内核 + Magisk ramdisk，能开机 |
| `boot_ksu_a12.img` | 废弃 | KSU + Magisk ramdisk（有 VINTF 弹窗） |
| `boot_ksu_a12_nomagisk.img` / `_v2.img` | 废弃 | VINTF 未修复版 |

root 状态：**已激活**——KernelSU legacy Manager 注入 ksud 成功，`adb shell su -c id` 返回 `uid=0`。

---

## 4. 复现方法（编译 + 打包）

### 环境
- Arch Linux，clang 22.1.8 + lld + cpio（`sudo pacman -S clang llvm lld cpio`）
- 编译脚本：`/home/dev/桌面/mt6889/build_a12_clang.sh`（KCFLAGS 带几十个 `-I` MTK 头文件路径 + 14 个 `-Wno-error=`）

### 编译
```bash
cd kernel_a12
ln -sfn . kernel-4.14          # 必须
make ARCH=arm64 k6889v1_64_defconfig
sh /home/dev/桌面/mt6889/build_a12_clang.sh
# 产物: arch/arm64/boot/Image.gz（15945321 字节）、vmlinux（297MB）
```

### 打包
```bash
mkbootimg --kernel arch/arm64/boot/Image.gz \
  --ramdisk <原厂 ramdisk> --dtb <原厂 dtb> \
  --base 0x40000000 --kernel_offset 0x80000 --ramdisk_offset 0x7c80000 \
  --tags_offset 0xbc80000 --dtb_offset 0xbc80000 \
  --pagesize 2048 --header_version 2 \
  --os_version 12.0.0 --os_patch_level 2022-12 \
  --cmdline "bootopt=64S3,32N2,64N2 buildvariant=user" -o raw.img
# 然后 python：padding 32MB → 复制原厂 AVB0 vbmeta → 写 AVBf footer（大端）
```

---

## 5. 坑清单（重要）

1. **`ksud.c` 冷属性**：clone 后必须手动删 `__attribute__((cold))`，否则必 panic（见 §2.3）
2. **`CONFIG_CC_STACKPROTECTOR_STRONG` 必须 =y**：VINTF 检测要求，否则锁屏弹窗（见 §2.4）
3. **`kernel-4.14` 软链必须有**：否则 `oplus_battery_mtk6889R.h` 报 tcpm.h not found
4. **A12 打包用纯 Image.gz + 独立 dtb 段**：别用 A11 的 Image.gz-dtb 思路
5. **AVB 必须复制原厂 vbmeta + footer**：缺了 bootloader 拒绝引导（无限重启）
6. **KCFLAGS 的 `-Isspm/mt6885` 必须在 `-Isspm/v1` 之前**：否则 sspm_sbuf_get undefined
7. **换配置后删旧 .o**：`find drivers -name "*.o" -delete` 再编
8. **不要用 GCC 编译**（几十个兼容错误），必须 clang
9. **KernelSU Manager 必须用 rsuntk legacy 配套版**：ReSukiSU / KernelSU-Next 的 Manager 协议不兼容（包名也不在白名单）
10. **默认拥塞控制写 `CONFIG_DEFAULT_BBR=y`**：直接写 `CONFIG_DEFAULT_TCP_CONG="bbr"` 会被 Kconfig 忽略，默认仍是 bic
11. **MTK Kconfig 会强制 select（MODVERSIONS / TCP_CONG_BIC）**：defconfig 里 `# ... is not set` 无效，必须删 `drivers/misc/mediatek/Kconfig.default` 里的对应 `select` 行
12. **模块签名必须关**：defconfig 的 `CONFIG_MODULE_SIG_FORCE=y` 会拒掉所有厂商签名模块（wifi 等全挂），要置 `# CONFIG_MODULE_SIG... is not set`
13. **模块 CRC 必须跳过**：关签名后还会有 `disagrees about version of symbol module_layout`，需把 `kernel/module.c` 的 `check_version`/`check_modstruct_version` 改为恒返回 1
14. **KSU 的 su 上下文写不了 `/data/adb`**（SELinux），开机脚本方案不可行，改配置要直接走内核（如编掉 BIC）

---

## 6. 待办（下一个 AI 做）

- [x] 装 KernelSU legacy Manager（`下载/KernelSU_v3.2.2-10-legacy-42-g915b4872_32490-release.apk`）→ 注入 ksud → 验证 `adb shell su -c id` 返回 uid=0（✅ 已完成）
- [ ] 若 Manager 无法注入：把 ksud 直接打进 ramdisk（ksuinit 替换 init + build.prop 检查），重打包（已不需要）
- [ ] （可选）换 KernelSU-Next（社区主流 4k⭐），需重新移植编译
- [ ] （可选）更新 README 状态

---

## 7. 常用命令速查

```bash
# 编译
make ARCH=arm64 k6889v1_64_defconfig && sh /home/dev/桌面/mt6889/build_a12_clang.sh

# 解包/检查 boot
unpack_bootimg --boot_img boot_shouhou.img --out out
python3 -c "d=open('boot_ksu_a12_nomagisk_v4_bbr.img','rb').read();print(d[:8],int.from_bytes(d[8:12],'little'),len(d))"

# 抓手机日志（需 root）
adb shell su -c "cat /proc/last_kmsg" > last_kmsg.txt
adb logcat -d -b events | grep boot_progress

# VINTF 弹窗排查
adb pull /system/etc/vintf/compatibility_matrix.5.xml .

# 推送
git add -A && git commit -m "..." && git push origin master
```

---

*交接人：shuijingna111（2026-08-03）*
