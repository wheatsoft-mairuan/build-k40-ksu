# build-k40-ksu — 红米 K40 (alioth) 内核自动编译

使用 GitHub Actions 自动编译 红米 K40 / POCO F3 / Mi 11X（alioth，内核 4.19）内核，
已移除 KernelSU 集成，改为 **Droidspaces + APatch** 方案：

- 编译出**纯净内核**（不在编译期集成任何 Root）
- 应用 **Droidspaces 官方 non-GKI kABI 补丁**，安全启用 `CONFIG_SYSVIPC` / `CONFIG_IPC_NS`，解决**卡第一屏**
- 编译后自动校验关键内核配置，不满足直接失败，避免刷进去才发现
- Root 通过 **APatch Manager 修补 boot 镜像**获得

## 工作原理

Droidspaces 需要 `CONFIG_SYSVIPC` 和 `CONFIG_IPC_NS`，但直接开启会破坏内核 ABI 布局，
导致预编译厂商驱动加载失败而卡第一屏。官方 non-GKI kABI 补丁在不改变
`task_struct` 内存偏移的前提下安全开启这些功能。

补丁来源：`ravindu644/Droidspaces-OSS` → `Documentation/resources/kernel-patches/non-GKI/`

| 补丁 | 作用 |
|---|---|
| `01.fix_kernel_panic_in_xt_qtaguid.patch` | 修复 xt_qtaguid 在不活跃接口上调用 `dev_get_stats` 导致的内核 panic |
| `02.fix_restore cgroup file prefix handling .patch` | 恢复 cgroup 文件前缀处理，保证 Droidspaces/LXC 的 cgroup 兼容性 |

补丁必须应用到**内核源码根目录**（single-tree layout），不能放进 GKI 的 `common/`。

## 关于 boot.zip（重要）

`boot.zip` 是**打包 boot.img 的基底**：工作流会解包它，把新编译出的 `Image` 替换进去
（`magiskboot repack`），再用 APatch 修补。

**替换方法**：把自己的 `boot.img` 压缩成 zip，命名 `boot.zip` 放到仓库根目录覆盖即可
（zip 内直接是 `boot.img`，不要多套一层目录）。

> 请务必先备份当前可正常开机的 `boot.img`，一旦内核崩溃可立即救砖。

## 使用方法

1. Fork / 使用本仓库
2. 用自己的 `boot.img` 生成 `boot.zip` 覆盖仓库根目录的 `boot.zip`
3. 打开 **Actions** → **Build K40 Kernel** → **Run workflow**
4. 编译完成后在 Artifacts 下载 `alioth_hylab_defconfig-Droidspaces-APatch`
   （内含 `Kernel.zip`：`Image` / `dtbo.img` / 打包好的 `boot.img`）

### 输入参数

| 参数 | 默认值 | 说明 |
|---|---|---|
| `ksu` | `false` | **已废弃**，KernelSU 集成已移除，此开关不再生效 |
| `config_path` | `arch/arm64/configs/` | 内核配置文件路径 |
| `config_name` | `alioth_hylab_defconfig` | 内核编译配置文件名 |
| `kernel_tag` | `master` | 内核代码分支 / Tag |
| `git_url` | `https://github.com/zfdx123/kernel_xiaomi_alioth.git` | 内核仓库地址 |
| `clang_url` | WeebX-Clang 19.1.5 | 编译器下载地址 |

### 工作流步骤

```
Checkout → Initialization environment → Init Config → Install Clang
→ Clone Kernel                 克隆内核源码
→ Apply Droidspaces kABI Patch 【新增】应用两个官方 non-GKI 补丁
→ Append Droidspaces Config    【新增】追加 Droidspaces / APatch 所需配置
→ Make                         编译内核
→ Verify Kernel Config         【新增】校验 out/.config 关键项，缺失则失败
→ Patch Boot                   用 boot.zip 作基底 repack boot.img
→ Zip → Kernel REL             上传产物
```

## 刷机与 Root

1. 从 [APatch Releases](https://github.com/bmax121/APatch/releases) 下载安装 APatch Manager
2. 打开 APatch Manager → 右上角 → **选择一个启动镜像并修补** → 选中产物里的 `boot.img`
3. 设置 **SuperKey**（8–63 字符，必须含数字和字母，不能有特殊字符）→ **开始**
4. 修补结果在 `Download/apatch_patched_*.img`
5. 刷入：

```bash
fastboot flash boot apatch_patched_*.img
fastboot reboot
```

> APatch 仅支持修补 **boot 分区**，不要修补 `init_boot` 或其他分区。
> APatch 支持 Linux 内核 3.18–6.12，4.19 在支持范围内；核心要求 `CONFIG_KALLSYMS=y`。

## 验证内核配置

编译产物已启用 `CONFIG_IKCONFIG_PROC`，刷入后可直接核对：

```bash
adb shell zcat /proc/config.gz | grep -E "CONFIG_SYSVIPC|CONFIG_IPC_NS|CONFIG_NAMESPACES|CONFIG_KALLSYMS"
```

期望输出：

```
CONFIG_SYSVIPC=y
CONFIG_IPC_NS=y
CONFIG_NAMESPACES=y
CONFIG_KALLSYMS=y
```

## 常见问题

### 仍然卡第一屏

提取崩溃日志定位：

```bash
adb shell cat /proc/last_kmsg > last_kmsg.txt
# 或
adb shell cat /sys/fs/pstore/console-ramoops > console-ramoops.txt
```

搜索 `panic`、`kABI`、`sysvipc` 等关键词。常见原因：补丁未正确应用、
`task_struct` 布局仍被破坏、或个别厂商模块不兼容。

### 补丁应用失败

工作流已内置三级兜底（`git apply` → `git apply --3way` → `patch -p1 --forward`）。
如仍失败，可在本地手动确认：

```bash
patch -p1 --dry-run < 01.patch   # 先测试
patch -p1 --forward < 01.patch   # 跳过已应用的 hunk
```

### 容器内无法联网

旧内核需要关闭 `CONFIG_ANDROID_PARANOID_NETWORK`。若你的 defconfig 已开启该项，
可在内核源码对应 defconfig 中改为：

```
# CONFIG_ANDROID_PARANOID_NETWORK is not set
```

### 备选补丁方案

- `Goldzxcbug/Droidspaces_Kernel_patch`：作者声明包含"一定能开机"的补丁，主要针对 5.10+ GKI 内核
- `WildKernels/kernel_patches`：含 OnePlus 设备的 `enable_sysvipc.patch`，可作 4.19 内核参考

## 测试库

```
#支持关闭KSU
https://github.com/LowTension/BAALAM_android_kernel_xiaomi_sm8250

#https://github.com/fumirul/kernel_xiaomi_alioth

#不支持关闭KSU
https://github.com/zfdx123/kernel_xiaomi_alioth
```
