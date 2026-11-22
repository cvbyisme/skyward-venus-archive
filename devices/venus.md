# venus（小米 11）· HyperOS

范围只有一个：**小米 11（`venus` / M2011K2C）+ 澎湃 HyperOS**。不追求跨机型、跨内核通用。

## 镜像 = 内核 + ramdisk

`fastboot boot` 用的是镜像里的内核，不是手机里装的那个。所以：

- 同一台机器换了内核（原厂 qgki、Neko、别的第三方），**镜像也得跟着换**，否则
  `/vendor/lib/modules/*.ko` 的 vermagic 对不上，模块加载会 `Exec format error`；
  自定义内核改过的 config（SUSFS、uname 隐藏、binderfs 选项）也可能让 recovery 黑屏。
- `devices/venus.toml` 的 `[stock]` 指向 `stock-venus-boot.zip`，那是这台机器
  boot 分区的快照（TWRP ramdisk + Neko 内核）。**CI 产出的包就带这个内核**，
  纯原厂系统想用就得换内核，见下。

## 三种给别人用的方式（按稳妥排序）

1. **固化为常驻恢复（推荐）**
   `skyward embed <镜像>`，或界面里 高级 -> 固化为常驻恢复。
   它读的是**对方手机上 boot 分区此刻的内容**，只替换 ramdisk，内核和 boot 头原样保留，
   写前备份原分区。跟内核无关，也不用打包。

2. **`fastboot boot` 临时测试：用对方的 boot 当模板**
   ```sh
   adb shell su -c 'dd if=/dev/block/by-name/boot_a of=/sdcard/boot_a.img'
   skyward-pack boot --kernel boot_a.img --ramdisk skyward-venus-<sha>.img --out own-kernel.img
   ```
   `--kernel` 只取内核，头部（cmdline、OS_VERSION）仍来自 Skyward 的镜像。

3. **别做**：直接拿别人机器产出的镜像 `fastboot boot`。内核不同，风险自负。

## 另一处与内核无关的绑定

`devices/venus.toml` 里的 `[modules] load`、`[wlan] writes/firmware`、服务列表是按这台
机器的系统写的。换系统（比如 MIUI 14，见 `venus-miui14.md`）要另开分支改这些，内核不用改。
## 面板：实际值 1440x3200@120，配置里写的是 1080x2400@420

| 项 | 值 | 来源 |
|---|---|---|
| 实际面板 | hactive=1440、vactive=3200、fps=120 | 本机内核日志 `msm_drm` 初始化那几行 |
| `[display] width/height/density` | 1080 / 2400 / 420 | 旧文件，**来源不明**，无法核对到任何 prop 或文件 |
| 为什么没改 | 程序绘制前按 1440/1080 = 1.333 缩放，屏幕上是对的；只是注释把“低模式”说成了“面板” | 代码里 `ui` 的缩放常数 |

结论：**显示正确、来源错误**。这一项只记录不改 UI，改动 UI 会连带影响所有已适配机型。

## 解密链路：三态判据

`skyward data open` 结束时只会给出三种之一，写在 /mnt/vendor/persist/skyward.log：

| 态 | 含义 | 判据 |
|---|---|---|
| `Plain` | 这台机器没加密 | `/data` 上没有 `fileencryption`/`metadata_encryption`，且 `/data/media/0` 能列出**真名** |
| `Opened` | 加密了，每一层都开了 | metadata 解出并挂上 → DE 进 keyring → CE 进 keyring → `/data/media/0` 能列出真名 |
| `Sealed { layer, why }` | 加密了，卡在某一层 | `layer` ∈ `Metadata` / `DeviceEncryption` / `CredentialEncryption`，`why` 是那一层的原话 |

**绝不用“mount 成功”当解密成功**：FBE 下没有 key 也能 mount，只是文件名和内容是密文。
唯一判据是 `/data/media/0` 能列出非乱码的真实文件名。

共享存储另有一行：`the shared storage is not shown at /sdcard: <原因>`。
出现这行说明 `/sdcard` 没挂上，原因会写明是哪一步（`/sdcard` 是符号链接时会落到它指向的真实路径再 bind）。

## 环境指纹字段（fingerprint）

启动时写进 kmsg，前缀 `fingerprint:`，用来让两台机器的日志可比：

| 段 | 内容 | 为什么需要 |
|---|---|---|
| `fstab` | 每个 entry 的原文与 flags | 解密方式由 fstab 决定，抄错就全错 |
| `policy` | `/data`、`/data/media`、`/data/system_de`、`/data/misc/vold` 四处的 fscrypt policy | 判断 v1/v2、wrappedkey_v0、inlinecrypt |
| `props` | build/vendor/vndk 的补丁级别 | TA 会拿它比对 |
| `boot` | boot 分区头 offset 16 的 os_version | 内核与 ramdisk 的实际来源 |
| `manifest` | 实际存在哪些 HAL 实例 | 服务发现不能靠抄 |
| `nodes` | `/dev` 下 8 个节点的属主与权限 | TEE 通路的最小前提 |
| `kernel` | boot_a kernel 段长度、该段 sha256、`/proc/config.gz` 的 sha256 | **uname 会被伪装**，只能认字节 |

## 加密机测试步骤（给持有真加密机的人）

0. 先确认：**在系统里能正常解锁并进桌面**，否则后面所有材料都无意义。
1. 在系统里以 root 跑 `devices/venus-materials.sh`（读，不写，不打印任何密钥内容）：
   ```sh
   su -c 'sh /sdcard/skyward-materials.sh'
   ```
   产出 `/sdcard/skyward-materials/*.txt`，最要紧的是 `2-fstab.txt`、`4-keys.txt`、`6-kernel-crypto.txt`。
2. 刷 Skyward，进 recovery，跑自检：
   ```sh
   /system/bin/recovery tee-check      # 只碰临时密钥，证明 TEE 通路可用
   /system/bin/recovery data open      # 三态结论 + 共享存储原因
   ```
   二进制名是 `/system/bin/recovery`（TWRP 的 init 就这么起它），没有 `skyward` 这个命令。
3. 把 `/mnt/vendor/persist/skyward.log` 与 `skyward-kmsg.log` 导出。
4. 红线：`/metadata/vold/**`、`user_keys/**`、`spblob/**`、`gatekeeper.*.key`、`locksettings.db`
   全部**只读**；不对真实 handle 做失败性 gatekeeper verify；不对真实密钥 `upgradeKey`；不 format data。

## 归档后值得带走的（给下一个 fork 看）

本仓库在 `last-standalone-port` 停在原地，后续开发在 [cvbyisme/Skyward-venus](https://github.com/cvbyisme/Skyward-venus)（fork 自 [hushangda/Skyward-venus](https://github.com/hushangda/Skyward-venus)）。下面这些是本仓库有、那边没有或做得不同的东西：

| 文件 | 作用 | 为什么值得拿 |
|---|---|---|
| `crates/skyward-crypto/src/data.rs` 的 `show_owner_storage()` | `/sdcard` 是符号链接时跟随到真实路径再 bind，失败原因写日志 | 未加密机上 `/sdcard` 往往是 `→ /storage/self/primary`，直接 bind 会失败且没声音 |
| `crates/skyward-crypto/src/keymaster.rs` 的 `check()` + `skyward tee-check` | 不碰用户密钥，只用临时密钥做 generate/begin/update/finish 往返，每步打印 ErrorCode 与耗时 | 任何一次真机启动都能证明 qseecomd+keymaster+ICE 通路是否真的可用 |
| `crates/skyward/src/fingerprint.rs` | fstab 原文与 flags、四处目录 policy、props、boot 头、manifest 实例、8 个 /dev 节点属主、内核段长度与该段 sha256 | 两台机器的日志能逐项对比，而不是“都失败” |
| `crates/skyward/src/selfstated.rs` | 从 `/vendor/etc/init/*.rc` 与 `manifest*.xml` 读出真实服务补进列表 | 服务清单不该写死；不同固件名字会变 |
| `devices/venus-materials.sh` | 在**已加密**的手机系统内以 root 跑，产出七份 txt（fstab 两行、keys 目录名与大小、内核是否有 blk-crypto、系统自己成功解锁的 logcat） | 给持有加密机的人一张清单就够，不用来回问 |
| `skyward-check.sh` | 只认 boot 分区字节、kernel 段 sha256、`/proc/config.gz` sha256 | `uname` 会被伪装（SUSFS 等），这脚本不会被骗 |
| WLAN 的 `[wlan] writes` 写两个路径 | `b0000000.qcom,cnss-qca6490/fs_ready` 与 `/sys/kernel/cnss/fs_ready` 都写，缺一个只记一行日志 | 不同内核节点名不同，两条都不贵 |

**不要搬的**：本仓库曾把 `FS_IOC_GET_ENCRYPTION_POLICY_EX` 写成 `0xc0086613`（正确是 `0xc0096616`），把 key status 的 offset 写成 56（正确是 64），并用 `ENOTTY` 当“未加密”的判据（正确是 `ENODATA`）。以 AOSP `kernel_common` 的 `include/uapi/linux/fscrypt.h`（android11-5.4 / android12-5.4）为准。
