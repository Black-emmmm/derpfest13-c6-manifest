# DerpFest 13 · C6 (Unisoc UD710) GSI 构建清单

给 CB-C6-STU（`ud710_2h10` / UD710 / 1200×1920 竖屏）编译 DerpFest 13 GSI 的 repo manifest。

不需要 fork 整棵 AOSP 树。本仓库只做一件事：在官方 manifest 之后追加
`snippets/c6.xml`，用 `remove-project` 把三个上游项目替换成本人分支。

## 覆盖的项目

| 项目 | 分支 | 改了什么 |
|---|---|---|
| `device/phh/treble` | `13.0` | **设备树**：`aosp_arm64_avS.mk` 产品配置、`c6-patches/`（SystemUI 三处补丁 + 完整文件覆盖脚本）、`sepolicy`、`overlay`、`lightsfix` |
| `packages/modules/Bluetooth` | `main` | LDAC 默认 96000Hz / **24bit** / 660kbps；Gd stack 启动超时 3s → 15s |
| `vendor/derp` | `main` | `build/soong/Android.bp`、`config/common.mk`、`config/telephony.mk` |

其余 1154 个项目仍是 AOSP（清华镜像 `android-13.0.0_r75`）与 DerpFest 官方。
设备树这一条替换掉了 `default.xml` 里原本指向 `sounddrill31/device_phh_treble_aosp` 的通用 phh treble。

## 同步

```bash
mkdir derpfest13 && cd derpfest13
repo init -u https://github.com/Black-emmmm/derpfest13-c6-manifest -b main
repo sync -c -j8 --force-sync --no-clone-bundle --no-tags
```

首次同步约 60–80 GB，建议预留 150 GB 磁盘。

同步完成后**必须**再跑一次设备树自带的系统层补丁，否则编出来的东西和
`system_avS_v22_gms.img` 不一致：

```bash
cd device/phh/treble
bash c6-patches/apply.sh ../..        # 完整文件覆盖，最稳
bash c6-patches/apply.sh ../.. --check # 只看差异，不改动
```

覆盖前原文件会备份到 `c6-patches/backup-<时间戳>/`。改完必须重编。

## 构建

```bash
source build/envsetup.sh
lunch aosp_arm64_avS-userdebug

# 完整构建 system image（内部串行，勿并行，否则 OOM）
bash build_systemimage.sh
```

**成功判据**：日志出现 `#### build completed successfully`，且 `out/.../system.img`
的修改时间确实是本次。脚本没有 `set -e`，失败也会打印「完成」，必须两个条件同时满足。

只重编蓝牙库（改 LDAC 参数时）：

```bash
m libbluetooth_jni -j2
```

需要 `ALLOW_MISSING_DEPENDENCIES=true`，否则 `cts_defaults` 之类的缺失模块会让
soong bootstrap 直接失败。

## 为什么这几个参数是这样

**LDAC 24bit**：96kHz/32bit 的原始 PCM 码率约 6.1 Mbps，压到 660 kbps 是 9:1，
编码器保不住会丢帧，听感就是卡顿。24bit 才是 660 这一档的正确搭配。
采样率改不动——实测 88.2k / 48k / 44.1k 全被设备忽略，只有位深和质量档可切。

**栈超时 15s**：`stack_manager.cc` 里硬编码 3 秒等待栈启动，每次蓝牙进程重启
（开机、开关、被 WCND 杀）都会连崩 4–5 次，还会连带 WCND 的 `cp2_recovery`
把 WiFi 一起拉断。放宽到 15 秒后实测崩溃为 0。

## 已知问题（未在本仓库解决）

- **内存碎片**：4G 内存跑 A13 + GMS，`/proc/buddyinfo` 的 Normal zone 经常
  order 8（1MB 连续块）为 0。展锐 SC2355 WiFi 驱动拿不到连续内存会
  `WIFI_CMD_RND_MAC` 失败，进而 WCND 重置 CP2，**蓝牙和 WiFi 同时断开**。
  缓解手段是 `vm.min_free_kbytes=65536`、`vm.extfrag_threshold=50` 加定期
  `compact_memory`，但这治标不治本。
- **WiFi HAL fdsan**：`android.hardware.wifi@1.0-service` 在 `Wifi::stop()` 时
  重复 close 一个 `DIR*` 的 fd，撞 A13 bionic 的 fdsan 直接 abort。
  已用二进制 patch（`close` → `nop`）修掉，不在本 manifest 范围内。
- **`com.sprd.logmanager`**：展锐自带的老应用会在 fork 时 SELinux 上下文设置
  失败，导致 **zygote abort**（全部 App 重启）。建议 `pm disable-user` 掉。

## 云构建

仓库里有 `.github/workflows/build.yml`，手动触发（`workflow_dispatch`），可选
`systemimage` / `libbluetooth_jni` 两个目标。

**先说资源账**，这决定了能不能用托管 runner：

| 项 | 实测/需求 |
|---|---|
| `out/` 目录 | **111 GB** |
| 源码树 | **95 GB** |
| 合计 | **206 GB** |
| `system.img` | 2.7 GB |
| 完整构建耗时 | 2–4 小时（串行，并行会 OOM） |

对照 GitHub 托管 runner：`ubuntu-latest` 是 4 vCPU / 16GB RAM / 14GB SSD + 84GB 临时盘
≈ **98GB**，连源码都放不下，更别说 `out/`。所以 workflow 默认 `runs-on: [self-hosted, linux]`。

### 方案一：自托管 runner（推荐）

在已经能正常构建的机器上跑，源码常驻、增量构建只需几分钟：

```bash
# 在 GitHub 仓库 Settings → Actions → Runners → New self-hosted runner
mkdir -p ~/actions-runner && cd ~/actions-runner
# 按页面给出的命令下载并注册，然后：
./run.sh          # 前台；或 sudo ./svc.sh install && sudo ./svc.sh start 常驻
```

然后在仓库 **Settings → Variables** 里加一个 `AOSP_SRC_DIR`，指向源码根目录
（例如 `/home/black/WorkBuddy/Work/derpfest-build/derpfest13/aosp`）。不配则默认 `$HOME/derpfest13`。

runner 机器需要具备：`repo`、`git`、`openjdk-11`、`ccache`、`python3`、`unzip`，
以及 **至少 250GB 可用磁盘**。workflow 第一步会把磁盘/内存/工具全打印出来，跑挂了先看那段。

### 方案二：larger runner（付费）

想完全交给云端，需要 larger runner（16C / 64GB / 1TB），按公开价约 **$0.032/min**，
一次 3 小时构建 ≈ **$6**。把 `runs-on` 改成对应 label 即可。**手动触发前先算成本**，
别绑在 push 上。

### 两个不能省的前置

1. **`ALLOW_MISSING_DEPENDENCIES=true`**：这棵树的 `cts_defaults` 等模块缺失，
   soong bootstrap 会直接失败，workflow 里已经导出。
2. **构建成功判据不能只看日志**：`build_systemimage.sh` 没有 `set -e`，
   失败了一样打印「完成」。必须同时满足日志出现 `#### build completed successfully`
   **且** `system.img` 的 mtime 是本次，workflow 的「校验产物」步骤会打 mtime + md5。

## 配套 Magisk 模块

见 [c6-gsi-modules](https://github.com/Black-emmmm/c6-gsi-modules)。
