# marble（Redmi Note 12 Turbo / POCO F5）Android 12 5.10.149 GKI 内核：GitHub Actions 编译与刷机教程

本文对应设备：

- 代号：`marble`
- 机型：Redmi Note 12 Turbo / POCO F5
- 系统：Android 12
- 内核：`5.10.149`
- GKI 分支：`android12-5.10.149-2023-01`
- 仓库：`kwebi/GKI_KernelSU_SUSFS`
- 工作流：`.github/workflows/kernel-a12-5-10.yml`

> 本文的“官方内核”产物已经在真机刷入并启动。
> BakaSU 产物构建成功，但没有在真机上刷入验证；如果你不需要 BakaSU 的额外功能，建议先用 Official。

---

## 0. 先看结论：应该下载哪个文件

### Official 内核（推荐，稳定）

| 用途 | 文件 |
|---|---|
| Recovery / KernelSU 刷入（推荐） | `anykernel3-5.10.149-marble-no-vbmeta.zip` |
| 备用 AK3（可能自动处理 vbmeta） | `anykernel3-5.10.149-marble.zip` |
| fastboot 临时启动 / 永久刷入 | `boot-5.10.149-marble-repacked.img` |
| 与 Official 内核匹配的管理器 | `KernelSU-Manager-32701-official.apk` |

### BakaSU 内核（可选）

| 用途 | 文件 |
|---|---|
| Recovery / KernelSU 刷入 | `BakaSU-anykernel3-5.10.149-marble-no-vbmeta.zip` |
| fastboot 启动 / 刷入 | `BakaSU-boot-5.10.149-marble-repacked.img` |
| 与 BakaSU 内核匹配的管理器 | `BakaSU-Manager-35236.apk` |

**不要混用：**

- Official 内核 + BakaSU 管理器 = 不匹配。
- BakaSU 内核 + Official 32701 管理器 = UAPI 不匹配。
- Official 内核 + PR 管理器（包名 `me.weishu.kernelsu.pr`）= 不匹配。

---

## 1. 原理和版本关系（先理解，少踩坑）

这次编译的内核包含：

- KernelSU
- Droidspaces
- NTSync
- ZRAM 增强（LZ4 / LZ4K / LZ4KD / LZ4K_OPLUS）
- 可选 SUSFS
- 可选 BBG
- 可选 KPM（本教程关闭）

KernelSU 是否“完整可用”，不是只看管理器版本号，而是看 **UAPI 版本是否相等**：

```text
isFullFeatured() = isManager && kernelUAPIVersion == managerUAPIVersion && rootAvailable()
```

本次两套组合：

| 组合 | 内核 | 管理器 | 结果 |
|---|---|---|---|
| Official | `32702` / UAPI `6` | `32701` / UAPI `6` | 可用 |
| BakaSU | `35236` / UAPI `5` | `35236` / UAPI `5` | 理论可用，未实机验证 |

**关键点：**

- 版本号差 1 没关系，UAPI 相等才关键。
- Official 的 `v3.3.1 / 32680 / UAPI 5` 不能配 `32702 / UAPI 6` 内核。
- PR 管理器 `32703` 的包名和签名都不是官方，内核不会认。

---

## 2. 准备工作和风险

### 2.1 必须准备

1. 手机已解锁 Bootloader。
2. 电脑安装 `adb` 和 `fastboot`。
3. 手机开启开发者选项和 USB 调试。
4. 备份当前能开机的 **stock boot.img**。
5. 确认手机型号是 `marble`，系统是 Android 12，内核是 5.10.149。
6. 保留一条能回滚的 stock boot；不要删除。

检查命令：

```bash
adb devices
adb shell getprop ro.product.device
adb shell getprop ro.build.version.release
adb shell uname -r
```

期望：

```text
marble
12
5.10.149...
```

### 2.2 备份 stock boot

有 root 或 recovery 时，直接备份当前 boot 分区。没有 root 时，从对应固件包中提取 `boot.img`。

**必须使用和你当前系统版本匹配的 stock boot。** 不要拿别的固件版本的 boot 来回滚。

### 2.3 风险提示

- 刷内核有变砖风险，必须能进 fastboot / recovery。
- 不要随便刷 `vbmeta`。
- 不要重新锁 Bootloader。
- 不要在没有 stock boot 备份的情况下长期使用。
- 本教程的 BakaSU 产物没有实机验证，先备份再试。

---

## 3. Fork 仓库并开启 Actions

1. 打开：

```text
https://github.com/kwebi/GKI_KernelSU_SUSFS
```

2. 点右上角 **Fork**，fork 到自己的账号。
3. 进入自己的 fork，点 **Actions**。
4. 如果看到 “Workflows aren’t being run on this forked repository”，点 **I understand my workflows, go ahead and enable them**。
5. 确认工作流存在：

```text
.github/workflows/kernel-a12-5-10.yml
```

> 不需要把 GitHub Token 写进仓库，也不需要在网页端填 Token。
> 用网页 **Run workflow** 就够了。Token 只用于命令行触发，普通用户不需要。

---

## 4. 修改构建矩阵（很重要）

打开 `.github/workflows/kernel-a12-5-10.yml`，找到矩阵部分。

只保留你需要的版本：

```yaml
strategy:
  matrix:
    include:
      - version: "149"
        sub_level: "2023-01"
```

**坑：**

- 不要把 `149` 以外的版本一起跑。矩阵每多一个版本，就多一份编译时间和失败概率。
- `149` 对应 5.10.149；`2023-01` 对应 Android 12 GKI 分支。
- 如果你改了矩阵但没保存到默认分支，网页 Run workflow 可能仍然用旧矩阵。

---

## 5. 运行 Action：Official 参数

进入 **Actions → 内核构建 - Android 12 (5.10) → Run workflow**。

分支选择你修改过的分支，例如 `dev`。

参数：

| 参数 | 值 | 说明 |
|---|---|---|
| `kernelsu_variant` | `Official` | 官方 KernelSU，稳定 |
| `ksu_mode` | `禁用SUSFS` | 启用 KSU，关闭 SUSFS |
| `use_kpm` | `disabled (关闭)` | 本教程不用 KPM |
| `droidspaces` | `678` | Droidspaces 补丁槽位 |
| `droidspaces_ntsync` | `true` | 启用 NTSync |
| `use_zram` | `true` | 启用 ZRAM 增强 |
| `use_bbg` | `false` | 不需要 BBG |
| `artifact_upload_mode` | `上传全部` | 方便下载 AK3 和 boot |
| `version` | 留空 | 可选 |
| `build_time` | 默认即可 | 可选 |

### 5.1 最大的坑：`ksu_mode=关闭` 不是“关闭 KSU”

这个工作流里：

```yaml
enable_susfs: ${{ inputs.ksu_mode != '禁用SUSFS' && inputs.ksu_mode != '禁用KSU' }}
```

所以：

| 选项 | 实际含义 |
|---|---|
| `关闭` | **启用 SUSFS**（不是关闭 KSU） |
| `禁用SUSFS` | 启用 KSU，关闭 SUSFS |
| `禁用KSU` | 不编译 KernelSU |

你要的是 **KSU 可用、SUSFS 关闭**，所以必须选：

```text
禁用SUSFS
```

### 5.2 其他坑

- `use_kpm=patched` 目前不支持，别选。
- BakaSU 不支持 KPM，选 `disabled`。
- `droidspaces=off` 时不要开 `droidspaces_ntsync=true`。
- 只跑一个矩阵，别一次跑全部版本。
- Action 失败时先看“整理编译失败日志”和“收集补丁冲突文件”两个步骤。

---

## 6. 运行 Action：BakaSU 参数（可选）

如果你明确要 BakaSU：

| 参数 | 值 |
|---|---|
| `kernelsu_variant` | `BakaSU` |
| `ksu_mode` | `禁用SUSFS` |
| `use_kpm` | `disabled (关闭)` |
| `droidspaces` | `678` |
| `droidspaces_ntsync` | `true` |
| `use_zram` | `true` |
| `use_bbg` | `false` |
| `artifact_upload_mode` | `上传全部` |

BakaSU 的 UAPI 是 `5`，所以之后必须装 BakaSU 自己的管理器，不能装 Official 的 32701。

---

## 7. 等待构建并下载产物

一次完整构建大约 20–40 分钟，取决于缓存和 GitHub 负载。

构建成功后，在 run 页面底部下载：

- `Official_kernel-android12-5.10-149` 或 `BakaSU_kernel-android12-5.10-149`
- `Official-Manager(...)` 或 `BakaSU-Manager(...)`
- 可选的 SUSFS 模块

**坑：**

- GitHub Actions 的 artifact 会过期（默认 90 天），过期后要重新跑。
- run 页面里的 `boot*.img` 不一定是给你直接刷的，先看文件名和说明。
- 不要只下载 `boot.img` 就刷；优先用 AK3，或者用已经用 stock ramdisk 重打包的 boot。
- 下载后先解压，确认 `Image`、`anykernel.sh` 存在。

---

## 8. 产物怎么选：AK3 还是 boot.img

### 8.1 推荐：AK3

`anykernel3-5.10.149-marble-no-vbmeta.zip` 只改 boot，不主动写 vbmeta：

```text
NO_VBMETA_PARTITION_PATCH=1
patch_vbmeta_flag=0
```

适合大多数情况。

普通版 `anykernel3-5.10.149-marble.zip` 是：

```text
patch_vbmeta_flag=auto
```

某些设备/ROM 上它可能自动处理 vbmeta。**不确定时优先用 no-vbmeta 版。**

### 8.2 备用：boot.img

`boot-5.10.149-marble-repacked.img` 是用 stock ramdisk 重打包的，适合：

```bash
fastboot boot boot-5.10.149-marble-repacked.img
```

临时启动测试；确认能开机后再永久刷入。

BakaSU 的 `BakaSU-boot-5.10.149-marble-repacked.img` 也已经用同一份 stock ramdisk 重打包；
但它没有在真机上验证过，建议先用 `fastboot boot` 试，或者直接用 BakaSU AK3。

**坑：**

- Action 里生成的通用 `boot.img` 可能使用测试 AVB key，直接刷可能无法开机或 AVB 报错。
- 不要刷 `/root/gki-build/action/android12-5.10.149-2023-01-boot*.img` 这类测试镜像。
- 优先用 AK3，或者用 `fastboot boot` 先试。

---

## 9. 刷入方法

### 9.1 方法 A：AK3（推荐）

1. 把 `anykernel3-5.10.149-marble-no-vbmeta.zip` 放到手机。
2. 用 KernelSU 管理器、OrangeFox 或 TWRP 刷入。
3. 重启。
4. 进系统后装匹配的管理器。

不要在没备份 stock boot 的情况下刷。

### 9.2 方法 B：fastboot 临时启动

```bash
adb reboot bootloader
fastboot devices
fastboot boot boot-5.10.149-marble-repacked.img
```

如果临时启动能进系统，说明内核基本可用。

### 9.3 方法 C：fastboot 永久刷入

```bash
adb reboot bootloader
fastboot devices
fastboot flash boot boot-5.10.149-marble-repacked.img
fastboot reboot
```

**坑：**

- A/B 设备不要手动乱刷 `boot_a` / `boot_b`；用 AK3 或 `fastboot flash boot` 让工具处理当前槽位。
- 如果 `fastboot boot` 失败但 `fastboot flash boot` 可用，先确认 stock boot 已备份。
- 刷 boot 不会清数据，但解锁 Bootloader 会清数据。
- 不要刷 `vbmeta`，除非你明确遇到 AVB/verity 错误。

### 9.4 什么时候才考虑 vbmeta

只有出现明确的 AVB / dm-verity 错误，且你确认需要时，才考虑刷与固件匹配的 `vbmeta`。

**不要：**

- 不要为了“保险”先刷 vbmeta。
- 不要刷别的固件版本的 vbmeta。
- 不要刷完 vbmeta 后锁 BL。

---

## 10. 安装管理器（最容易错的一步）

### 10.1 Official 内核

当前 Official 内核：

```text
KernelSU 32702 / UAPI 6
```

安装：

```text
KernelSU-Manager-32701-official.apk
```

验证信息：

| 项目 | 值 |
|---|---|
| 包名 | `me.weishu.kernelsu` |
| 版本 | `32701` |
| UAPI | `6` |
| 证书 SHA256 | `c371061b19d8c7d7d6133c6a9bafe198fa944e50c1b31c9d8daa8d7f1fc2d2d6` |
| 证书长度 | `0x033b` |
| SHA256 | `30678984bc9899a631d74260fb9a6854d0f4e3fc2a12e865d37d90b2e0b3c2d9` |

命令：

```bash
adb uninstall me.weishu.kernelsu.pr
adb install -r KernelSU-Manager-32701-official.apk
```

如果提示 `Unknown package`，说明没装 PR 管理器，忽略第一条。

如果提示签名冲突：

```bash
adb uninstall me.weishu.kernelsu
adb install KernelSU-Manager-32701-official.apk
```

只会清管理器设置，不会清内核。

### 10.2 BakaSU 内核

BakaSU 内核：

```text
KernelSU 35236 / UAPI 5
```

安装：

```text
BakaSU-Manager-35236.apk
```

验证信息：

| 项目 | 值 |
|---|---|
| 包名 | `org.bakasu.bakasu` |
| 版本 | `35236` |
| UAPI | `5` |
| 证书 SHA256 | `d3469712b6214462764a1d8d3e5cbe1d6819a0b629791b9f4101867821f1df64` |
| 证书长度 | `0x0377` |
| SHA256 | `48889ddf1b9e7f1c3e56fdfb92eee65d954fcc5cac122451a910225943eea537` |

```bash
adb install -r BakaSU-Manager-35236.apk
```

### 10.3 管理器对照表（背下来）

| 内核 | 可以装 | 不能装 |
|---|---|---|
| Official 32702 / UAPI 6 | Official 32701 / UAPI 6 | Official 32680 / UAPI 5、PR 32703、BakaSU 管理器 |
| BakaSU 35236 / UAPI 5 | BakaSU 35236 / UAPI 5 | Official 32701 / UAPI 6、PR 32703 |

### 10.4 为什么 32701 能配 32702

因为 `isFullFeatured()` 比较的是 UAPI：

```text
32702 内核 UAPI 6 == 32701 管理器 UAPI 6
```

版本号差 1 不影响。相反，`32680` 虽然也是官方签名，但 UAPI 是 5，所以会提示“需要更新管理器”。

### 10.5 PR 管理器为什么不能用

之前提供的 `KernelSU-Manager-32703.apk`：

```text
包名：me.weishu.kernelsu.pr
证书：291628287ca876f33e2cd07dcd29d992a4db1aee34db83a3123ed4174873b76f
```

内核只认：

```text
包名：me.weishu.kernelsu
证书：c371061b19d8c7d7d6133c6a9bafe198fa944e50c1b31c9d8daa8d7f1fc2d2d6
```

所以 PR 管理器打开后会显示“KernelSU 未安装”，没有 Superuser。

---

## 11. 验证是否成功

### 11.1 基本检查

```bash
adb shell uname -r
adb shell getprop ro.product.device
adb shell su -c id
```

期望：

```text
5.10.149marble
marble
uid=0(root) gid=0(root) ...
```

### 11.2 ZRAM

```bash
adb shell cat /sys/block/zram0/comp_algorithm
```

期望输出中包含：

```text
[repl] lz4 lz4k lz4kd lz4k_oplus ...
```

方括号表示当前算法，目标是 `[lz4kd]`。

如果当前不是：

```bash
adb shell su -c 'echo lz4kd > /sys/block/zram0/comp_algorithm'
```

如果写入失败，说明内核没有编进对应算法，回看 `use_zram=true` 和构建日志。

### 11.3 NTSync

```bash
adb shell ls -l /dev/ntsync
```

期望能看到 `/dev/ntsync`。

没有的话：

- 确认 `droidspaces_ntsync=true`。
- 确认构建日志里 NTSync 配置注入成功。
- 确认 `CONFIG_NTSYNC=y`。

### 11.4 Droidspaces

```bash
adb shell getprop ro.product.device
```

安装 Droidspaces 后先跑一个 Alpine/OpenRC 容器，确认能启动，再上 QDebian KDE + Anland。

**坑：**

- Droidspaces 槽位要和构建参数 `678` 对应。
- 容器里缺 `/dev/ntsync` 时，先修内核，不要在容器里硬凑。
- 不要一上来就装 KDE；先用最小容器验证内核和挂载。

### 11.5 SELinux

```bash
adb shell getenforce
```

不要为了省事关掉 SELinux；先看是不是管理器/权限问题。

---

## 12. 回滚

### 12.1 fastboot 回滚

```bash
adb reboot bootloader
fastboot devices
fastboot flash boot stock-boot.img
fastboot reboot
```

### 12.2 recovery 回滚

用 recovery 恢复之前备份的 boot，或者重刷当前固件包里的 boot。

### 12.3 如果开不了机

1. 长按电源 + 音量键进 fastboot。
2. 刷回 stock boot。
3. 不要锁 BL。
4. 不要刷来历不明的 vbmeta。

---

## 13. 常见错误排查

### 13.1 Action 在“添加 KernelSU”失败

原因：

- KernelSU / BakaSU 上游仓库改了。
- 网络拉取失败。
- 分支/tag 不存在。

处理：

- 看日志里的具体命令。
- Official 可以临时 pin 到 tag，例如 `v3.3.1`，但要注意管理器 UAPI 也要匹配。
- BakaSU 用 `main`，上游改动可能导致编译失败。

### 13.2 补丁冲突

看 run 页面里的：

```text
收集补丁冲突文件
上传补丁冲突文件
```

下载冲突文件，不要忽略 `.rej`。补丁冲突时不要硬编，先修 patch。

### 13.3 编译成功但刷完不开机

优先怀疑：

- 刷了测试 AVB key 的通用 boot。
- 刷了不匹配的 vbmeta。
- boot 分区大小/格式不对。
- 没有用 stock ramdisk 重打包。

处理：

- 用 AK3 no-vbmeta。
- 用 `fastboot boot` 先试。
- 刷回 stock boot。

### 13.4 开机了但没有 root

按顺序检查：

1. 管理器包名是不是 `me.weishu.kernelsu`。
2. 管理器证书是不是 `c371...`。
3. 内核和管理器 UAPI 是否相等。
4. 是否装了 PR 管理器。
5. 是否装了旧 v3.3.1 / UAPI 5。

### 13.5 提示“需要更新管理器”

说明：

```text
managerUAPIVersion < kernelUAPIVersion
```

处理：

- Official 32702 内核：装 32701 / UAPI 6。
- BakaSU 内核：装 BakaSU 35236 / UAPI 5。
- 不要装 32680 / UAPI 5 配 32702 内核。

### 13.6 ZRAM 不是 lz4kd

- 确认 `use_zram=true`。
- 看构建日志里 ZRAM 配置步骤。
- 看 `/sys/block/zram0/comp_algorithm` 是否包含 `lz4kd`。
- 不要在不支持的情况下强行写算法。

### 13.7 没有 `/dev/ntsync`

- 确认 `droidspaces_ntsync=true`。
- 确认 NTSync patch 没被跳过。
- 确认 `CONFIG_NTSYNC=y`。
- 重新编译，不要只改运行时。

### 13.8 GitHub artifact 下载失败

- artifact 可能过期，重新跑 Action。
- 大文件下载慢时用 `curl -L -C -` 续传。
- 优先从 Release 下载，Release 不会像 artifact 那样过期。

### 13.9 Token 泄露

- 不要把 PAT 写进 workflow、README、issue、聊天。
- 一旦泄露，立即去 GitHub **Revoke**。
- 命令行只需要 fine-grained PAT：
  - Contents: Read and write
  - Actions: Read and write
- 普通网页编译不需要 PAT。

---

## 14. 本次构建记录

### Official

- 工作流：`kernel-a12-5-10.yml`
- Run ID：`39786909426`
- 结果：成功
- 内核：`5.10.149marble`
- KernelSU：`32702` / UAPI `6`
- 管理器：`32701` / UAPI `6`
- 已实机刷入并启动

### BakaSU

- 工作流：`kernel-a12-5-10.yml`
- Run ID：`39805325592`
- 结果：构建成功
- KernelSU：`35236` / UAPI `5`
- 管理器：`35236` / UAPI `5`
- 未实机刷入验证

---

## 15. 文件校验

发布页会附带 `SHA256SUMS.txt`。下载后先校验：

Linux：

```bash
sha256sum -c SHA256SUMS.txt
```

Windows PowerShell：

```powershell
Get-FileHash .\KernelSU-Manager-32701-official.apk -Algorithm SHA256
```

重点文件：

```text
30678984bc9899a631d74260fb9a6854d0f4e3fc2a12e865d37d90b2e0b3c2d9  KernelSU-Manager-32701-official.apk
48889ddf1b9e7f1c3e56fdfb92eee65d954fcc5cac122451a910225943eea537  BakaSU-Manager-35236.apk
25ba61928f9d00c3409bf5707283e281c9c2bf167b5c9064d5d8b9947e89cd5e  anykernel3-5.10.149-marble-no-vbmeta.zip
51e885a8716c22de2249175fc2a068150c7c83fe78f3e5326c9147251a579ad5  boot-5.10.149-marble-repacked.img
57ae7ad07ed14e364a271ff611e60b7567329d271c52f78ce0a11528f5d9d516  BakaSU-anykernel3-5.10.149-marble-no-vbmeta.zip
d8a0eec19efe314171ea0d16d0858d41519e611bec070163b591bc059e5ae853  BakaSU-boot-5.10.149-marble-repacked.img
```

---

## 16. 最短操作清单

如果你只想照着做：

1. 备份 stock boot。
2. Fork 仓库，开启 Actions。
3. 矩阵只留 `149 / 2023-01`。
4. 运行 Official：
   - `ksu_mode=禁用SUSFS`
   - `use_kpm=disabled`
   - `droidspaces=678`
   - `droidspaces_ntsync=true`
   - `use_zram=true`
   - `use_bbg=false`
   - `artifact_upload_mode=上传全部`
5. 下载 `anykernel3-...-no-vbmeta.zip`。
6. 用 KernelSU/OrangeFox/TWRP 刷入。
7. 安装 `KernelSU-Manager-32701-official.apk`。
8. 验证：
   ```bash
   adb shell uname -r
   adb shell su -c id
   adb shell cat /sys/block/zram0/comp_algorithm
   adb shell ls -l /dev/ntsync
   ```
9. 出问题就刷回 stock boot。

---

## 17. 致谢

- KernelSU：`tiann/KernelSU`
- BakaSU：`Baka-SU/BakaSU`
- 上游 GKI 构建项目及补丁作者
- Droidspaces / SUSFS / ZRAM 补丁作者

本项目只是把编译流程、产物和踩坑记录整理出来。刷机有风险，请先备份。
