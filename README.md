# kishimc4sheng

Xiaomi Pad 6S Pro（`xiaomi,sheng` / SM8550）**kShimC 开机菜单双启动**的构建工作流记录。
不是上游项目仓库，只是把“从干净环境产出可刷 `boot_a` 镜像”的每一步、每个坑写清楚，方便复现。

## 从哪看起

- 构建全流程 → **[构建工作流-build-workflow.md](构建工作流-build-workflow.md)**（本仓库主页的 README 也是同一份内容）

## 相关仓库

| 仓库 | 作用 |
|---|---|
| [tc1911/kShimC4sheng](https://github.com/tc1911/kShimC4sheng) | 上游：kShimC boot menu（GPL-3.0），含 `scripts/`、`packer/`、`prebuilt/` |
| [sunflower2333/kShimC](https://github.com/sunflower2333/kShimC) | kShimC runtime 的原始上游 |
| [Project-Aloha/DualBootKernelPatcher](https://github.com/Project-Aloha/DualBootKernelPatcher) | 打包器 `MultiBootKernelPatcher` 来源 |
| [ianchb/sm8550-mainline](https://github.com/ianchb/sm8550-mainline) | sheng 的 mainline 内核（内核 / DTB / deb 包来源） |
| [code002-2/Xiaomi-pad-6s-pro-Linux](https://github.com/code002-2/Xiaomi-pad-6s-pro-Linux) | 双系统分区布局与安装文档参考 |

## 许可

- 本文档内容为设备适配与构建笔记。
- 涉及的上游组件（kShimC runtime、MultiBootKernelPatcher 等）均为 **GPL-3.0**，二进制与源码版权归各自作者；本仓库不分发原厂内核、boot 分区 dump、DTBO 等厂商专有内容。

---

# kShimC4sheng 构建工作流 / Build workflow

小米平板 6S Pro（`xiaomi,sheng` / SM8550）开机菜单双启动的完整构建流程。
本文件只讲“怎么从一个干净环境产出可刷的 `boot_a` 镜像”；原理、排障与实机记录属于作者本机笔记，未随本仓库发布。

- 上游项目（菜单与打包器本体，GPL-3.0）：<https://github.com/tc1911/kShimC4sheng>
- 内核/固件来源：<https://github.com/ianchb/sm8550-mainline>
- 设备：Xiaomi Pad 6S Pro 12.4（`xiaomi,sheng` / SM8550），BL 已解锁，固定 A 槽

- 上游仓库：`github.com/tc1911/kShimC4sheng`（GPL-3.0）；下文出现的 `~/AIWORK/...` 都是作者本机路径，按自己的目录替换即可
- 当前基线：`v0.0.6`（commit `7aafa11`），runtime sha256 `4194554f…`
- 最近的产物：`out/boot_kishim_20261109-1845.img`，96,956,416 B，sha256 `9790a57229c088ecc3bf155f25fbba5cd00d861a1faed0d51b765e36987ff70f`

---

## 0. 一图流（推荐路径）

```
原厂 Android boot.img ──┐
                        ├─► prepare.sh ──► payload/ ──┐
Linux Image + DTB ──────┘                             ├─► build.sh ──► boot_a.img ──► dd / fastboot
prebuilt/sheng-runtime.bin（4 MiB）──────────────────┘
```

只有要改菜单/配置时才需要重建 runtime（第 4 节），日常换内核只走第 1–3 节，**不需要交叉编译器**。

---

## 1. 准备输入

| 输入 | 要求 | 本机现成文件 |
|---|---|---|
| 原厂 Android boot | v4 boot 镜像或整分区 dump，**内核必须是 raw ARM64 Image** | `~/AIWORK/.tmp/stock_base.img`（46,682,112 B，由 `/WoaManager/boot.img` 解 gzip 后重建） |
| Linux 内核 | raw `Image`，或 `Image.gz`（脚本内部解压） | `/boot/Image.gz` 或 ianchb release 里的 `Image` |
| DTB | sheng 设备树 | `/boot/sm8550-xiaomi-sheng.dtb`，或 `boot_sheng_dualboot_plymouth.img` 尾部附加的 FDT |
| runtime | 固定 4 MiB | `prebuilt/sheng-runtime.bin` |
| initrd | 可选 | 通常没有（7.3.0 内核自带内置 initramfs） |

### 1.1 原厂内核是 gzip 时怎么处理（必踩）

`/WoaManager/boot.img` 的 kernel 段是 **gzip 单成员 + 可能附带尾部数据**，直接喂给 packer 会报
`stock kernel is gzip-compressed; decompress to a raw Image`。正确做法：

```bash
cd ~/AIWORK/项目-projects/kShimC4sheng
python3 - <<'PY'
import sys, zlib
sys.path.insert(0, 'packer')
import bootimg
d = open('/WoaManager/boot.img', 'rb').read()
h = bootimg.parse(d)                       # 返回 dict
seg = d[h['page_size']:h['page_size'] + h['kernel_size']]
z = zlib.decompressobj(31)                 # 只解第一个 gzip 成员
raw = z.decompress(seg) + z.flush()        # 尾部残留忽略，别用 gzip.decompress
assert raw[:2] == b'MZ' and raw[0x38:0x3C] == b'ARM\x64'
open('/home/username/AIWORK/.tmp/stock_base.img', 'wb').write(bootimg.build(h, raw))
PY
```

> `packer/extract.py` 与 `packer/make_boot.py` 内部也是 `zlib.decompressobj(16 + MAX_WBITS)` 只解第一个成员，
> 所以也可以改用 `--boot`/`--linux-image` 让脚本自己解；但 **stock 的 base image 必须自己先转 raw**。

### 1.2 Linux 载荷：从社区 boot 镜像里拆

`boot_sheng_dualboot_plymouth.img` 是 v0 + gzip 内核 + 尾部附加 DTB：

```bash
python3 - <<'PY'
import zlib
d = open('/home/username/AIWORK/.tmp/rel730/boot_sheng_dualboot_plymouth.img','rb').read()
ks = int.from_bytes(d[8:12],'little')
z = zlib.decompressobj(31)
img = z.decompress(d[4096:4096+ks]) + z.flush()
open('/home/username/AIWORK/.tmp/rel730/Image','wb').write(img)
open('/home/username/AIWORK/.tmp/rel730/dtb_from_img.dtb','wb').write(z.unused_data)  # FDT 魔数 d00dfeed
PY
```

---

## 2. 提取载荷（prepare.sh）

```bash
scripts/prepare.sh --stock-boot /path/stock_base.img \
                   --linux-image /path/Image \
                   --dtb /path/sheng.dtb \
                   [--initrd /path/initrd.img] \
                   --out out/payload
```

产出（payload 目录固定布局）：

| 文件 | 必需 | 说明 |
|---|---|---|
| `stock-boot.img` | 是 | 原厂 boot（拿到的是 raw 内核） |
| `linux-Image` | 是 | raw ARM64 Image |
| `linux.dtb` | 视情况 | 有 DTB 就带 |
| `linux-initrd.img` | 否 | 存在则自动打进 manifest |

也可以用 `--distro-boot <linux-boot.img>` 一次给出 `Image.gz + DTB [+ initrd]`。

---

## 3. 打包（build.sh → packer/make_boot.py）

```bash
scripts/build.sh --payload out/payload \
                 --runtime prebuilt/sheng-runtime.bin \
                 --out out/boot_a.img \
                 --default android --timeout-ms 8000 \
                 --bootargs 'root=PARTLABEL=linux rw rootwait quiet splash plymouth.ignore-serial-consoles fbcon=map:1 udmabuf.size_limit_mb=256' \
                 --partition-size 201326592
```

内部调用 `packer/make_boot.py`，它会：

1. 写 MBKP 配置（`[Image-00] Android` / `[Image-01] Linux` / `[Image-02] sheng-dtb` / `[Image-03] initrd` / `[Pack]`）；
2. 调 `build/packer/MultiBootKernelPatcher`（缺则用 cmake 现场编译）产出 packed 内核；
3. 调 `packer/bootimg.build()` 把 boot 头几何沿用原厂、内核段换成 packed blob；
4. 跑 `verify()` 做结构自检。

### 3.1 verify() 会检查什么（失败原因都在这里）

| 检查 | 失败提示 |
|---|---|
| `manifest.image_size == len(packed)` | `manifest image_size ... != packed size` |
| manifest 备份的原厂头 5 个 u64 与 stock 一致 | `base image backup ... != stock header` |
| stock 头槽 `0x20/0x28/0x30` 必须为 0 | `stock kernel header slots ... are not zero` |
| 条目顺序 `Android, Linux[, sheng-dtb][, initrd], Shim` | `manifest entries ... != expected` |
| 条目 0 是 `Linux` 且带 `BASE_IMAGE|EXECUTABLE` | `entry 0 is not a Linux base image` |
| 条目 1 带 `HAS_DTB` 且 `dtb_entry == 2` | `entry 1 has no DTB reference` |
| 条目 2 载荷以 FDT 魔数开头、大小一致 | `entry 2 is not an FDT` |
| 最后一条是 `Shim`，offset 等于 `shim_offset`，size 等于 runtime 大小 | `last entry is not an executable Shim` |
| 各条目不越过 manifest | `entry ... overlaps manifest` |

### 3.2 实际产物结构（2026-11-09 那一版）

```
manifest v0.1  default=0(Android)  timeout=8000ms  shim_offset=0x2d30000
  Android        Linux  BASE_IMAGE|EXECUTABLE   off=0x0        46,676,480 B
  Linux          Linux  EXECUTABLE|HAS_DTB      off=0x3200000  43,747,840 B
  sheng-dtb      DTB                            off=0x5c50000    153,100 B
  Attached Shim  Shim   EXECUTABLE              off=0x2d30000   4,194,304 B
文件内偏移 = packed 偏移 + 0x1000（boot 头一页）
内核槽 96,950,104 B，分区 201,326,592 B，余 104,370,176 B
```

`build.sh` 会在打包后检查「boot 头 + packed ≤ 分区 − 64 KiB」，超限直接失败。

---

## 4. （可选）重建 runtime

只有改了菜单/语言/配置才需要，日常换内核跳过。

```bash
scripts/fetch_deps.sh          # 拉 CrDK/picolibc/lvgl/libfdt（走 GitHub 镜像，需要网络）
scripts/fetch_toolchain.sh     # Arch 系自动解包 aarch64 交叉工具链；Debian 用系统 aarch64-linux-gnu-gcc
export PATH=~/AIWORK/.venv/kshim/bin:$PATH   # genconfig（kconfiglib）
nice -n 19 -c scripts/build_runtime.sh       # 从 config/sheng_defconfig 构建 4 MiB runtime
```

产出必须**正好 4,194,304 字节**，否则 `make-boot.sh` 会拒绝。

---

## 5. 一键路径（推荐日常）

上面的 1–3 步已封装成：

```bash
~/AIWORK/项目-projects/kishimc-local/重建双启动镜像-rebuild-dualboot-image.sh            # 只构建
~/AIWORK/项目-projects/kishimc-local/重建双启动镜像-rebuild-dualboot-image.sh --flash --yes   # 构建+备份+刷入+读回校验
```

它自动处理：找 stock base（默认 `/WoaManager/boot.img`）→ gzip 解压 → 重建 base →
找 Linux 内核与 DTB → 调 `make-boot.sh` → 容量检查 → 写同名 `.txt` 版本标记 →（`--flash`）备份旧分区、
`dd bs=4M conv=fsync` 写入、按块读回比 sha256、写 `last-slot.txt` 与 `flash-history.log`。

实测整链约 2 秒，同输入两次构建 sha256 一致。

---

## 6. 验证与刷入

```bash
make test                      # 纯 Python 回归（合成镜像，不需要设备）
scripts/test_boot.sh           # fastboot boot，一次性加载不写分区
scripts/backup.sh              # adb 备份 boot/dtbo/init_boot/vendor_boot
scripts/flash.sh -y            # fastboot flash boot_a
```

本机没有 fastboot，实际刷入走 `dd`：

```bash
sudo dd if=out/boot_kishim_XXXX.img of=/dev/disk/by-partlabel/boot_a bs=4M conv=fsync
sudo dd if=/dev/disk/by-partlabel/boot_a of=/tmp/rb.img bs=4M   # 读回
head -c <镜像大小> /tmp/rb.img | sha256sum                       # 应与源文件一致
```

---

## 7. 常见失败与处理

| 症状 | 原因 | 处理 |
|---|---|---|
| `stock kernel is gzip-compressed` | base image 还是 gzip | 按 1.1 先解成 raw |
| `cannot satisfy Linux Image alignment` | 镜像里有个别条目太小（合成测试时遇到过） | 用接近真实大小的载荷 |
| `Failed to start media pipeline: -32` | 与构建无关（相机 RDI 宽度限制） | — |
| `image ... exceeds pad target` | 载荷超分区 | 减小内核/去掉 initrd |
| `manifest entries ... != expected` | payload 目录混入旧文件 | 清空 `out/payload` 重跑 |
| `base image backup ... != stock header` | stock 内核头 0x20/0x28/0x30 非零 | 从工厂 boot 重新取内核 |

---

## 8. 固定参数速查

| 参数 | 值 | 来源 |
|---|---|---|
| boot 分区大小 | 201,326,592 B（192 MiB） | `config/sheng.env` |
| page size | 4096 | 同上 |
| 默认菜单项 | `android`，超时 8000 ms | 同上 |
| 默认 bootargs | `root=PARTLABEL=linux rw rootwait` | 同上（实际构建会覆盖成带 `splash`/`udmabuf` 的完整串） |
| runtime 大小 | 4,194,304 B | 硬校验 |
| 对齐 | Linux/Android 2 MiB（`Align=21`），DTB 8 B（`Align=3`） | `make_boot.py` 生成配置 |
| Shim 位置 | packed `0x2d30000`（文件内 `0x2d31000`） | 由 base image 大小决定 |
