# DerpFest 13 · C6 (Unisoc UD710) GSI 构建清单

给 CB-C6-STU（`ud710_2h10` / UD710 / 1200×1920 竖屏）编译 DerpFest 13 GSI 的 repo manifest。

不需要 fork 整棵 AOSP 树。本仓库只做一件事：在官方 manifest 之后追加
`snippets/c6.xml`，用 `remove-project` 把两个改动过的上游项目替换成本人分支。

## 覆盖的项目

| 项目 | 改了什么 |
|---|---|
| `packages/modules/Bluetooth` | LDAC 默认 96000Hz / **24bit** / 660kbps；Gd stack 启动超时 3s → 15s |
| `vendor/derp` | 启动动画、`build/soong/Android.bp`、`config/common.mk`、`config/telephony.mk` |

其余 1154 个项目仍是 AOSP（清华镜像 `android-13.0.0_r75`）与 DerpFest 官方。

## 同步

```bash
mkdir derpfest13 && cd derpfest13
repo init -u https://github.com/Black-emmmm/derpfest13-c6-manifest -b main
repo sync -c -j8 --force-sync --no-clone-bundle --no-tags
```

首次同步约 60–80 GB，建议预留 150 GB 磁盘。

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

## 配套 Magisk 模块

见 [c6-gsi-modules](https://github.com/Black-emmmm/c6-gsi-modules)。
