# OpenWrts 现状梳理与友善 NanoPi R4S 定制指南

本文基于当前仓库（`dev` 分支）实际配置整理，目标是回答两个问题：

1. 这个仓库现在构建什么、包含哪些插件和功能；
2. 如果要针对友善 NanoPi R4S 做个人定制，应该改哪些文件、走哪条路径、注意哪些坑。

> 说明：仓库本身不存放 OpenWrt 源码，只负责“选源码 + 打补丁 + 合成 `.config` + 云编译 + 发布”。插件是否真正可用，最终仍以上游 LEDE/ImmortalWrt 的 feed 和编译日志为准。
>
> 当前仓库已完成 **R4S-only** 改造：`configs/targets/` 下只保留 `rockchip.config`（单 profile），默认 LAN IP 已改为 `192.168.1.2`，并新增了 `files/etc/uci-defaults/99-r4s-custom` 首次启动设置。`standard` 与 `lite` 插件集已按同一份清单精简，rootfs 均为 2048MB。

---

## 1. 项目定位与构建拓扑

当前支持两套上游源码：

| 源码 ID | 上游 | 默认分支 | 当前用途 |
| --- | --- | --- | --- |
| `lede` | `coolsnowwolf/lede` | `master` | standard + lite |
| `immortalwrt` | `immortalwrt/immortalwrt` | `master` | 仅 lite |

当前 `manifests/builds.json` 实际包含 **3 个构建条目**：

- LEDE：`rockchip` × 2 个风格（`standard`、`lite`）= 2 条
- ImmortalWrt：`rockchip` × 1 个风格（`lite`）= 1 条

设备 ID 与对应目标：

| 设备 ID | target 配置 | 覆盖设备 |
| --- | --- | --- |
| `rockchip` | `configs/targets/rockchip.config` | NanoPi R4S（单 profile） |

`configs/targets/` 下已删除 `x86_64`、`rpi3`、`rpi4`、`rpi5` 配置，仓库变为 R4S-only。`rockchip.config` 只保留 R4S 一个 profile，因此一次 `rockchip` 构建只产出 R4S 固件。R4S 没有独立的设备级应用配置，`standard` 和 `lite` 分别使用 `configs/apps/lede/standard.config` 与 `configs/apps/common/lite.config`；驱动配置已针对 R4S 裁剪。

---

## 2. 当前插件与功能清单

### 2.1 LEDE Standard（`configs/apps/lede/standard.config`）

已按 R4S 精简，只保留以下插件：

| 分类 | 插件/功能 |
| --- | --- |
| 代理 | `luci-app-ssr-plus` |
| DNS 与加速 | `luci-app-adguardhome`、`luci-app-turboacc`（仅本体，不含 PDNSD / DNSFORWARDER / DNSPROXY） |
| 服务 | `luci-app-dockerman`（自带 `docker`、`dockerd` 等依赖）、`luci-app-ttyd` |
| 系统与工具 | `luci-app-diskman`、`luci-app-filetransfer`、`luci-app-firewall`、`luci-app-pushbot` |
| 主题 | `luci-theme-argon` |
| 语言 | `en`、`zh-cn` |
| 基础 | IPv6 helper、ext4 rootfs、kernel 64MB、rootfs 2048MB |

已从 Standard 移除的典型项：`luci-app-store`、`luci-app-openclash`、`luci-app-passwall2`、`luci-app-samba4`、`luci-app-nfs`、`luci-app-frpc`、`luci-app-netdata`、`luci-app-oaf`、`luci-app-qbittorrent*`、`luci-app-docker`（保留 `dockerman` 即可）、多余主题和 `zh-tw`。

### 2.2 Common Lite（`configs/apps/common/lite.config`）

LEDE lite 和 ImmortalWrt lite 共用这份配置，插件清单与 Standard 一致：

| 分类 | 插件/功能 |
| --- | --- |
| 代理 | `luci-app-ssr-plus` |
| DNS 与加速 | `luci-app-adguardhome`、`luci-app-turboacc`（仅本体） |
| 服务 | `luci-app-dockerman`（自带 `docker`、`dockerd` 等依赖）、`luci-app-ttyd` |
| 系统与工具 | `luci-app-diskman`、`luci-app-filetransfer`、`luci-app-firewall`、`luci-app-pushbot` |
| 主题 | `luci-theme-argon` |
| 语言 | `en`、`zh-cn` |
| 基础 | IPv6 helper、ext4 rootfs、kernel 64MB、rootfs 2048MB |

Lite 与 Standard 共用同一份精简清单，两者构建产物只差 `app_config` 来源（`common/lite.config` 同时被 ImmortalWrt 使用），不再包含商店、OpenClash、Passwall2、Samba4、SmartDNS、OAF、Netdata、qBittorrent、`default-settings*` 和额外主题。

### 2.3 插件来源

| 来源 | 内容 |
| --- | --- |
| 上游 LEDE / ImmortalWrt 默认 feed | 大部分 `luci-app-*`、主题、协议、kmod |
| `feeds/common.conf` | `helloworld`（SSR+ 等）、`passwall_packages`、`passwall2`、`istore`（应用商店） |
| `packages/lede.sh` | 克隆 `OpenAppFilter`（当前 app 配置未启用） |
| `packages/immortalwrt.sh` | 克隆 `OpenAppFilter`、`OpenClash`（当前 app 配置未启用） |

注意：

- `feeds/lede.conf`、`feeds/immortalwrt.conf` 当前不存在，两套源码共用 `feeds/common.conf`。
- `packages/lede.sh` 已移除 `luci-theme-neobird` 和 `OpenClash` 的克隆步骤，减少 CI 拉取耗时；`packages/immortalwrt.sh` 仍保留 `OpenClash`，但当前精简后的 app 配置不会启用它。
- `OpenAppFilter`（`luci-app-oaf` + `appfilter` + `kmod-oaf`）也已从 app 配置移除，克隆步骤暂时保留，如确认不需要可一并删除。
- 很多插件的具体可用性取决于上游 feed，编译时若包名不存在会被 `make defconfig` 丢弃，需看编译日志确认。

---

## 3. 构建流程与关键文件

```text
manual-build.yml / schedule-release.yml
  -> node scripts/openwrts.mjs validate-manifest
  -> node scripts/openwrts.mjs generate-workflows --check
  -> node scripts/openwrts.mjs resolve-matrix
  -> build-openwrt.yml
     -> scripts/prepare-env.sh
     -> scripts/clone-source.sh
     -> scripts/apply-openwrt.sh system feeds
     -> feeds update -a / install -a
     -> scripts/apply-openwrt.sh packages
     -> scripts/compose-config.sh
     -> scripts/build.sh download/tools/toolchain/target/packages/images
     -> artifact / release
```

关键行为：

1. `scripts/apply-openwrt.sh system` 只做一件事：把上游 `config_generate` 中的 `192.168.1.1` 改成 `192.168.1.2`。
2. `scripts/apply-openwrt.sh feeds` 把 `feeds/common.conf` 和可选的 `feeds/<repo>.conf` 去重追加到 `feeds.conf.default`。
3. `scripts/apply-openwrt.sh packages` 执行 `packages/<repo>.sh`，从第三方仓库克隆插件。
4. `scripts/compose-config.sh` 按 **target → driver → app** 的顺序拼接 `.config`，然后执行两次 `make defconfig`，并强制取消所有 `CONFIG_DEFAULT_luci*`。因此：
   - 后面的片段可以覆盖前面的同名配置；
   - 不要依赖“默认 LuCI 选择”，要显式写 `CONFIG_PACKAGE_xxx=y`；
   - 若包不存在，`make defconfig` 会把它丢掉。
5. 支持根目录 `files/` 覆盖层，`compose-config.sh` 会把它复制到 `openwrt/files`。当前仓库已提供 `files/etc/uci-defaults/99-r4s-custom`。
6. `scripts/build.sh` 分阶段编译，CI 依次调用 `download → tools → toolchain → target → packages → images`。
7. 缓存 key 包含 `repo + branch + cache_scope + hash(feeds/packages/scripts/configs/manifests)`，不同源码/设备/风格不会混用缓存。

---

## 4. R4S 当前构建状态

### 4.1 已支持的 R4S profile

`configs/targets/rockchip.config` 现在只包含 R4S：

```config
CONFIG_MODULES=y
CONFIG_TARGET_rockchip=y
CONFIG_TARGET_rockchip_armv8=y
CONFIG_TARGET_MULTI_PROFILE=y
CONFIG_TARGET_DEVICE_rockchip_armv8_DEVICE_friendlyarm_nanopi-r4s=y
```

R4S 固件会出现在：

```text
openwrt/bin/targets/rockchip/armv8/
```

文件名以 `friendlyarm_nanopi-r4s-*` 开头，常见为 `squashfs-sysupgrade.img.gz` 和 `ext4-sysupgrade.img.gz`（以实际构建输出为准）。

### 4.2 R4S 当前使用的配置

- LEDE standard：`configs/apps/lede/standard.config`
- LEDE / ImmortalWrt lite：`configs/apps/common/lite.config`
- LEDE R4S 驱动扩展：`configs/drivers/lede/rockchip.config`（已按 R4S 裁剪，manifest 中所有 LEDE 条目都指向它）
- ImmortalWrt 驱动扩展：`configs/drivers/immortalwrt/common.config`（目前只有注释占位）
- 首次启动覆盖层：`files/etc/uci-defaults/99-r4s-custom`

也就是说，当前 target 已收窄为 R4S 单 profile；`standard` 与 `lite` 应用配置都已按 R4S 精简为同一份清单；驱动配置已去掉 R4S 不用的 USB 网卡/3G/4G/USB Audio/USB 串口等，只保留存储（exFAT/ext4/squashfs/vfat、USB 存储）和风扇相关 kmod。

### 4.3 硬件相关提醒

NanoPi R4S 是 RK3399、双千兆网口（一个原生、一个 PCIe）、4GB RAM、microSD/eMMC、USB 3.0/2.0、风扇接口、串口调试口，无 WiFi。定制时建议：

- **驱动已按 R4S 裁剪**：`configs/drivers/lede/rockchip.config` 只保留 exFAT/ext4/squashfs/vfat、I2C/hwmon/pwmfan、USB2/USB3/OHCI 和 USB 存储；已移除 `kmod-usb-net-*`、3G/4G（`comgt`/`umbim`/`uqmi`）、USB Audio、USB Printer、`kmod-usb-serial`、NFS、f2fs/xfs 以及 `kmod-inet-diag`。原 `configs/drivers/lede/common.config` 已随 R4S-only 一并删除。
- **确认 PCIe 网口驱动**：LEDE 的 `friendlyarm_nanopi-r4s` profile 已通过 `DEVICE_PACKAGES` 自带 `kmod-r8168`（PCIe 千兆网口）和 `kmod-gpio-button-hotplug`，一般不需要在 driver 配置里重复添加。若第二个网口不识别，优先检查 dmesg 中的 `r8168` 和 PCIe 枚举，而不是盲目加 USB 网卡驱动。
- **存储**：已保留 `kmod-usb2`/`kmod-usb3`/`kmod-usb-ohci`、`kmod-usb-storage`、`kmod-usb-storage-uas`、`kmod-fs-ext4`、`kmod-fs-exfat`、`kmod-fs-squashfs`、`kmod-fs-vfat`。需要 NTFS 时可启用注释中的 `ntfs-3g` 或 `kmod-fs-ntfs3`；确实需要 NFS/f2fs/xfs 时再按需加回对应 kmod。
- **eMMC/SD**：`CONFIG_TARGET_ROOTFS_PARTSIZE` 已提升到 `2048`，即 rootfs 2GB，给 Docker、AdGuardHome 留出余量；如仍不足可继续提高到 4096，或在首次启动后自行扩容分区。
- **风扇/温控**：已加入 `kmod-i2c-core`、`kmod-hwmon-core`、`kmod-hwmon-pwmfan`；RK3399 的 PWM provider 由上游 rockchip target 内核/DTS 提供，`kmod-pwm-rockchip` 并不是 LEDE 的独立 kmod 包。若实际风扇不走 PWM，需要按 R4S 设备树再确认。
- **串口调试**：R4S 的调试口是板载 3-pin UART，通常不依赖 USB 串口 kmod；如需外接 USB-TTL，可自行加回 `kmod-usb-serial` 及对应芯片驱动。
- **无 WiFi**：不需要保留 WiFi 相关包。
- **刷机**：首次刷写用完整镜像写 SD/eMMC；已在 OpenWrt 内升级用 sysupgrade。LEDE 与 ImmortalWrt 之间不要保留配置跨源升级，先备份再全新刷写。

---

## 5. R4S 定制路径

### 方案 A：把仓库改成 R4S 专用（当前仓库已采用）

适合“把这个 fork 固定为只构建 R4S”的场景。该方案已全部落地：`rockchip.config` 收窄为 R4S 单 profile，`x86_64`/`rpi3`/`rpi4`/`rpi5` 的 target 配置已删除并重新生成 manifest（现为 3 条），driver 配置、`files/` 覆盖层、默认构建、LAN IP、插件精简和 rootfs 2048 均已完成。

1. 修改 `configs/targets/rockchip.config`，只保留 R4S：

```config
CONFIG_MODULES=y
CONFIG_TARGET_rockchip=y
CONFIG_TARGET_rockchip_armv8=y
CONFIG_TARGET_MULTI_PROFILE=y
CONFIG_TARGET_DEVICE_rockchip_armv8_DEVICE_friendlyarm_nanopi-r4s=y
```

   保留 `CONFIG_TARGET_MULTI_PROFILE=y`（rockchip 使用 multi-profile 选择方式），只保留一个 `CONFIG_TARGET_DEVICE_...`；其他 6 个 `CONFIG_TARGET_DEVICE_...` 已删除。

2. 删除其余 target 配置：`configs/targets/x86_64.config`、`rpi3.config`、`rpi4.config`、`rpi5.config` 已删除，只保留 `rockchip.config`，然后运行 `node scripts\openwrts.mjs generate-workflows` 重新生成 manifest；条目从 15 条收敛为 3 条。

3. 应用配置：`configs/apps/lede/standard.config` 与 `configs/apps/common/lite.config` 都已精简为同一份清单（SSR+、AdGuardHome、TurboACC 本体、Docker（`dockerman`）、ttyd、filetransfer、diskman、firewall、pushbot、argon、en/zh-cn）。R4S-only 后这两份配置只服务 R4S；`lite` 仍与 ImmortalWrt 共用。

4. 新增 `configs/drivers/lede/rockchip.config`（当前仓库已新增），只给 R4S 使用：
   - 删掉 R4S 不需要的 USB 网卡、3G/4G、USB Audio、USB 串口、NFS、f2fs/xfs 等；
   - 只保留存储（exFAT/ext4/squashfs/vfat、USB 存储）和风扇（I2C/hwmon/pwmfan）相关 kmod；
   - 原 `configs/drivers/lede/common.config` 已删除（没有其他 LEDE target）；manifest 中所有 `lede/rockchip/*` 条目的 `driver_config` 指向 `configs/drivers/lede/rockchip.config`。

5. 已新增 `files/` 覆盖层做首次启动设置：

```text
files/etc/uci-defaults/99-r4s-custom
```

   当前设置 hostname 为 `NanoPi-R4S`、时区为 `Asia/Shanghai`、LAN IP 为 `192.168.1.2`。注意不要把 root 密码明文提交到公开仓库。

6. `manifests/builds.json` 中 `rockchip` 的 `op_name` 已改为 `NanoPi-R4S`；后续运行 `generate-workflows` 时 `op_name` 会被保留。

7. 已把 manifest 的默认值改成 LEDE + rockchip + standard：

```json
"workflow": {
  "manual": {
    "default_repo": "lede",
    "default_device": "rockchip",
    "default_flavor": "standard"
  },
  "schedule": {
    "default_repo": "lede",
    "default_devices": "rockchip",
    "default_flavors": "standard"
  }
}
```

   `repo=auto` 的轮换逻辑仍然存在，但定时任务默认使用 `lede`，不会再自动切到 ImmortalWrt。

8. 运行验证：

```powershell
node scripts\openwrts.mjs validate-manifest
node scripts\openwrts.mjs generate-workflows
node scripts\openwrts.mjs generate-workflows --check
node scripts\openwrts.mjs resolve-matrix --repo lede --device rockchip --flavor standard
```

9. 在 Actions 里手动构建：`repo=lede`、`device=rockchip`、`flavor=standard`，先不要上传 release，下载 artifact 测试。

**优点**：改动最小，不碰生成器，manifest 只需生成即可收敛为 3 条。
**现状**：`configs/targets/` 只剩 `rockchip.config`，仓库已是 R4S-only；`manifests/builds.json` 为 `lede/rockchip/standard`、`lede/rockchip/lite`、`immortalwrt/rockchip/lite` 三条。

### 方案 B：新增独立 `rockchip-r4s` target

适合“在保留其他设备构建的同时给 R4S 单独一个入口”的场景。当前仓库已完成 R4S-only，本方案仅作为需要恢复多设备时的参考。

1. 新增 `configs/targets/rockchip-r4s.config`，内容只选 R4S：

```config
CONFIG_MODULES=y
CONFIG_TARGET_rockchip=y
CONFIG_TARGET_rockchip_armv8=y
CONFIG_TARGET_DEVICE_rockchip_armv8_DEVICE_friendlyarm_nanopi-r4s=y
```

2. 运行：

```powershell
node scripts\openwrts.mjs generate-workflows
```

   由于 `openwrts.mjs` 会把 **所有 target × 所有 flavor** 做笛卡尔积，manifest 会自动新增：

- `lede/rockchip-r4s/standard`
- `lede/rockchip-r4s/lite`
- `immortalwrt/rockchip-r4s/lite`

   原来的 `rockchip` 条目不受影响。

3. 把新增条目的 `op_name` 改成 `NanoPi-R4S`，再次运行 `generate-workflows`（`op_name` 会被保留）。

4. 构建时只选 `device=rockchip-r4s`，即可只编 R4S 单 profile。

**局限**：

- 生成器不支持“某设备只允许某 flavor”。新增 target 会自动和全部 flavor 组合。
- `app_config` 在 `syncManifestData` 中始终按 `configs/apps/<repo>/<flavor>.config` 重算，手改 manifest 的 `app_config` 会被覆盖。
- `driver_config` 会被保留：可以新增 `configs/drivers/lede/r4s.config`，然后把 manifest 条目的 `driver_config` 改成它，生成时不会被重置。
- 如果新增 `configs/apps/lede/r4s.config`，它会成为新 flavor `r4s`，并给 **所有 target** 增加 `r4s` 条目，而不只是 R4S。可以接受“多出来的条目不构建”，但 manifest 和 workflow 选项会变多。

### 方案 C：改造生成器，支持设备级 profile

（历史备选方案）如果后续要长期维护多设备 + 每设备不同插件集，建议改造 `scripts/openwrts.mjs`，而不是靠 flavor 笛卡尔积。

可考虑在 manifest 中增加设备级覆盖：

```json
"device_profiles": {
  "rockchip-r4s": {
    "flavors": ["r4s"],
    "app_config": "configs/apps/lede/r4s.config",
    "driver_config": "configs/drivers/lede/r4s.config"
  }
}
```

然后修改 `syncManifestData`：

- 按 target 读取允许的 flavor 列表，而不是全局 `flavorOrder`；
- 对带 `app_config` 覆盖的设备，不再强制重算 `app_config`；
- 保留现有 `op_name`、`cache_scope`、`driver_config`；
- 同步更新 `validateManifestData`、workflow 选项和 README。

如果还需要“只有 R4S 克隆某个第三方包”，再给 `build-openwrt.yml` 增加 `DEVICE` 输入，并把它传给 `apply-openwrt.sh packages`，或者支持 `packages/<repo>/<device>.sh`。当前 `packages/<repo>.sh` 只按源码区分，不区分设备。

---

## 6. R4S 插件取舍建议

R4S 4GB 内存足够跑大部分功能，但固件体积、稳定性和冲突风险主要来自“同类插件叠加”。建议按用途做减法：

| 用途 | 建议保留 | 建议二选一/谨慎 |
| --- | --- | --- |
| 代理 | 只选一个：OpenClash / Passwall2 / SSR+ | 三者都装会争抢防火墙、TUN、DNS 和 53 端口 |
| DNS | 只选一个：AdGuardHome / SmartDNS / TurboACC DNS 选项 | 同时开启容易端口冲突、解析链混乱 |
| Docker | `luci-app-dockerman` 或 `luci-app-docker` 选一个 | 两者功能重叠；当前只保留 `luci-app-dockerman` |
| 下载 | qBittorrent static 或 simple_dynamic 选一个 | 当前已移除，需要时再加回 |
| NAS | Samba4 / NFS / FileBrowser 按需 | 全开会增加攻击面和资源占用 |
| 流量整形 | QoS / mwan3 / TurboACC 按需 | 与代理、OAF 组合时注意兼容性 |
| 主题 | 保留 1 个常用主题 | 6 个主题会明显增大固件 |
| 监控 | Netdata / nlbwmon / onliner 按需 | 常驻服务会占内存 |
| 通知 | Pushbot / ServerChan 按需 | 二选一即可 |
| 3G/4G/USB 网卡 | R4S 不需要，已从 LEDE driver 配置移除 | 需要 USB 2.5GbE 时再启用 `kmod-usb-net-rtl8152` |

另外，`luci-app-oaf` 会编译内核模块 `kmod-oaf` 并接入 netfilter，和流量整形/代理组合时建议单独测试。

---

## 7. 当前仓库中值得注意的不一致与风险

这些不是 R4S 专属，但会影响后续定制判断：

1. **README 与 manifest 已对齐**
   - README 构建表只列 3 条：`lede/rockchip/standard`、`lede/rockchip/lite`、`immortalwrt/rockchip/lite`，与 `manifests/builds.json` 一致。
   - README 的 Standard/Lite 插件表均已同步为当前精简集。
2. **`rockchip` target 已收窄为 R4S**
   - `configs/targets/rockchip.config` 不再包含 R68S、R2C、R2S、R5C、R5S、Orange Pi R1 Plus。
   - 如果以后要重新支持这些设备，需要把对应 profile 加回 target 配置，或走方案 B/C。
   - `configs/targets/` 下只有 `rockchip.config`，仓库为 R4S-only；原 `configs/drivers/lede/common.config` 已删除。
3. **`configs/apps/lede/standard.config` 与 `configs/apps/common/lite.config` 已重写为同一份精简集**
   - 旧的重复 `luci-app-samba4` 随重写一起移除；
   - 当前只保留第 2.1 节列出的插件。
4. **`manifests/builds.json` 是生成文件**
   - `generate-workflows` 会重算 `id`、`flavor`、`target_config`、`app_config`、`cache_scope`；
   - 只有 `op_name`、`driver_config` 和额外自定义字段会被保留；
   - 不要指望手改 `app_config` 能长期生效。
5. **新增 target 会全量笛卡尔积**
   - 新增 `configs/targets/foo.config` 会自动给所有 repo、所有 flavor 增加条目；
   - 新增 `configs/apps/<repo>/bar.config` 会自动给所有 target 增加 `bar` flavor。
6. **`common/lite.config` 被 LEDE 和 ImmortalWrt 共用**
   - 两套源码的包集合并不完全一致；
   - 若某包只在其中一套存在，`make defconfig` 会静默丢弃，必须看编译日志。
7. **默认密码/默认设置未在仓库中显式定义**
   - `scripts/apply-openwrt.sh` 只改 LAN IP；
   - README 写 `root/password`，但 `lite.config` 已不再启用 `default-settings-chn`，实际密码取决于上游默认；
   - 个人定制建议用 `files/` 或首次启动手动设置，不要把密码写进公开仓库。
8. **`.codex/agents/op-dev.toml` 已过时**
   - 里面仍引用 `scripts/resolve-matrix.py`、`apply-system.sh`、`apply-feeds.sh`、`apply-packages.sh`、`configs/targets/<repo>/`；
   - 当前实际是 `scripts/openwrts.mjs`、`scripts/apply-openwrt.sh`、`configs/targets/*.config`。
9. **`repo=auto` + `flavor=standard` 可能在 ImmortalWrt 周失败**
   - ImmortalWrt 当前只有 lite；
   - 默认已固定为 `lede`，但手动选择 `auto` 且 `flavors=standard` 时，轮换到 ImmortalWrt 会没有匹配条目。
10. **跨源码升级不支持**
   - LEDE 与 ImmortalWrt 的 feeds、LuCI、包集合不同；
   - 不要用一方的配置备份恢复到另一方。

---

## 8. 推荐落地顺序

当前仓库已完成方案 A 的全部落地：`configs/targets/` 只保留 `rockchip.config`（R4S 单 profile），`x86_64`/`rpi3`/`rpi4`/`rpi5` 配置已删除，manifest 收敛为 3 条；R4S 专用 driver 配置、`files/` 首次启动设置、默认值 `lede + rockchip + standard`、LAN IP `192.168.1.2`、Standard/Lite 插件精简、rootfs 2048 均已完成。后续建议：

1. 跑 manifest/矩阵校验，确认 `generate-workflows --check` 通过。
2. 在 Actions 手动构建 `repo=lede`、`device=rockchip`、`flavor=standard`，先下载 artifact。
3. 在 R4S 上确认：能启动、双网口正常、eMMC/SD 正常、LAN IP 为 `192.168.1.2`、风扇按需可用。
4. 如后续还要增减插件，继续改 `configs/apps/lede/standard.config` 和 `configs/apps/common/lite.config`（两份保持同步）。
5. 跑 Docker/AdGuardHome 后如空间不足，可把 `CONFIG_TARGET_ROOTFS_PARTSIZE` 继续提到 4096，并复核 USB 存储 kmod。
6. 如果以后要恢复多设备或每设备不同插件集，再考虑方案 B/C 改造生成器。

---

## 9. 验证命令

```powershell
node scripts\openwrts.mjs validate-manifest
node scripts\openwrts.mjs generate-workflows --check
node scripts\openwrts.mjs resolve-matrix --repo lede --device rockchip --flavor standard
node scripts\openwrts.mjs resolve-matrix --repo lede --device rockchip --flavor lite
node scripts\openwrts.mjs resolve-matrix --repo immortalwrt --device rockchip --flavor lite
```

R4S-only 改造完成后已用临时 Node.js 重新生成并校验：`generate-workflows` 更新了 `manifests/builds.json`（3 条）和 `manual-build.yml`；`validate-manifest` 返回 `Manifest validation passed.`，`generate-workflows --check` 返回 `Generated files are up to date.`；三个 R4S 矩阵（`lede/standard`、`lede/lite`、`immortalwrt/lite`）均能正确解析。修改 feeds 后还可以运行：

```bash
bash tests/apply-openwrt-feeds.sh
```

---

## 10. 关键文件速查

| 目的 | 文件 |
| --- | --- |
| 选设备/裁剪 R4S target | `configs/targets/rockchip.config` |
| 选 R4S 插件 | `configs/apps/lede/standard.config`、`configs/apps/common/lite.config` |
| R4S 驱动/kmod | `configs/drivers/lede/rockchip.config` |
| 第三方 feed | `feeds/common.conf` |
| 第三方插件克隆 | `packages/lede.sh`、`packages/immortalwrt.sh` |
| 构建矩阵与默认值 | `manifests/builds.json` |
| 矩阵/工作流生成器 | `scripts/openwrts.mjs` |
| 默认 LAN IP | `scripts/apply-openwrt.sh` |
| 首次启动覆盖层 | `files/etc/uci-defaults/99-r4s-custom` |
| 配置合成 | `scripts/compose-config.sh` |
| CI 构建流程 | `.github/workflows/build-openwrt.yml` |
