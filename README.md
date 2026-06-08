# GEC6818 OTA Stage 1 学习记录

这是一次基于 GEC6818 / S5P6818 开发板的 OTA 升级实验记录。

本阶段目标不是实现完整商用 OTA，而是先跑通一个最小可用流程：

```text
HTTP 下载固件 → sha256 校验 → 备份旧文件 → 更新 boot 分区 → 重启验证
```

本次最终结果：

- 成功通过局域网 HTTP 服务下载 `Image` 和 DTB
- 成功校验 `sha256`
- 成功将旧 `Image` 和旧 DTB 备份到 p3 分区
- 成功将新 `Image` 和新 DTB 写入 p1 boot 分区
- 更新后 hash 与服务器端一致

---

## 1. 环境说明

开发板：

```text
GEC6818 / Smart6818
SoC: Samsung S5P6818
System: Ubuntu 20.04 LTS arm64 minimal
Kernel: Linux 4.4.172-s5p6818
```

启动结构：

```text
U-Boot
  ↓
p1 boot 分区
  ↓
Image + ramdisk.img + DTB
  ↓
kernel
  ↓
p2 / p3 OverlayFS rootfs
```

eMMC 分区：

| 分区 | 作用 |
|---|---|
| `/dev/mmcblk0p1` | boot 分区，存放 `Image`、`ramdisk.img`、DTB |
| `/dev/mmcblk0p2` | rootfs lowerdir，基础系统 |
| `/dev/mmcblk0p3` | userdata，存放 overlay 和 OTA 备份 |

本阶段只更新：

```text
Image
s5p6818-gec6818-rev01.dtb
```

暂时不更新：

```text
ramdisk.img
rootfs
```

原因是 `ramdisk.img` 和 rootfs 影响早期启动，风险更高。Stage 1 先验证最基础的 boot 文件远程更新流程。

---

## 2. OTA 固件目录

Windows 主机上准备固件目录：

```text
ota-firmware/
├── manifest.txt
├── sha256sums.txt
├── Image
└── s5p6818-gec6818-rev01.dtb
```

`manifest.txt` 示例：

```bash
VERSION=2026.06.08-stage1
BOARD=WC-gec6818
UPDATE_IMAGE=1
UPDATE_DTB=1
UPDATE_RAMDISK=0
UPDATE_ROOTFS=0
IMAGE_FILE=Image
DTB_FILE=s5p6818-gec6818-rev01.dtb
```

`sha256sums.txt` 示例：

```bash
2d92254d576f9b64bd238d2882c2be41a2f91aa52c34f20e3802e3fd376ee49b  manifest.txt
8f7720d599cf30971a8ccec40e7a646e55e5f22a0c593fb8e8e072b9576a5ff5  Image
604728b6baa568df5de139bb523793b915c5f3ca6b74283ee1b876b4d91d8077  s5p6818-gec6818-rev01.dtb
```

---

## 3. Windows 端启动 HTTP 服务

进入固件目录：

```powershell
cd C:\ota-firmware
```

启动 HTTP 服务：

```powershell
python -m http.server 8000 --bind 0.0.0.0
```

如果开发板访问不到，需要放行 Windows 防火墙：

```powershell
netsh advfirewall firewall add rule name="OTA HTTP 8000" dir=in action=allow protocol=TCP localport=8000
```

在开发板上测试：

```bash
wget http://192.168.88.1:8000/manifest.txt
```

能正常下载说明 HTTP 服务可用。

---

## 4. OTA 脚本设计 (在最底下贴出)

设备端脚本位置：

```bash
/usr/sbin/ota_stage1_update.sh
```

脚本主要做这些事：

```text
1. 接收 OTA 服务器地址
2. 下载 manifest.txt
3. 下载 sha256sums.txt
4. 根据 manifest 下载 Image / DTB
5. 执行 sha256 校验
6. 挂载 p1 boot 分区
7. 挂载 p3 userdata 分区
8. 把旧 Image / DTB 备份到 p3
9. 把新 Image / DTB 写入 p1
10. sync 后提示重启
```

本次采用的关键策略：

```text
旧文件备份到 p3，不备份到 p1
```

备份路径：

```bash
/mnt/ota-data/ota-backup/boot/时间戳/
```

例如：

```bash
/mnt/ota-data/ota-backup/boot/20260608-115046/
```

这样可以避免 boot 分区空间被备份文件占满。

---

## 5. 执行 OTA 前检查

先清理可能残留的挂载点：

```bash
sudo -s

umount /tmp/bootpart 2>/dev/null || true
umount /mnt/ota-data 2>/dev/null || true
umount /mnt/p3 2>/dev/null || true
umount /mnt/ota-boot 2>/dev/null || true
```

确认没有残留挂载：

```bash
mount | grep -E 'mmcblk0p1|mmcblk0p3|ota-boot|ota-data|tmp/bootpart|mnt/p3'
```

如果没有输出，说明挂载点干净。

检查 p1 空间：

```bash
mkdir -p /mnt/ota-boot
mount /dev/mmcblk0p1 /mnt/ota-boot
df -h /mnt/ota-boot
umount /mnt/ota-boot
```

本次遇到的问题是：

```text
lsblk 显示 p1 是 64M
但 df 显示文件系统只有 23M，并且已经 100% 使用
```

解决方法：

```bash
resize2fs /dev/mmcblk0p1
```

扩容后：

```text
/dev/mmcblk0p1   60M   23M   35M  40%
```

---

## 6. 执行 OTA

执行：

```bash
/usr/sbin/ota_stage1_update.sh http://192.168.88.1:8000
```

关键成功输出：

```text
[OTA-STAGE1] sha256 OK
[OTA-STAGE1] Old Image backed up to: /mnt/ota-data/ota-backup/boot/20260608-115046/Image
[OTA-STAGE1] Old DTB backed up to: /mnt/ota-data/ota-backup/boot/20260608-115046/s5p6818-gec6818-rev01.dtb
[OTA-STAGE1] Image updated: Image
[OTA-STAGE1] DTB updated: s5p6818-gec6818-rev01.dtb
[OTA-STAGE1] OTA Stage 1 finished successfully
```

---

## 7. 重启前验证

挂载 p1：

```bash
mkdir -p /mnt/ota-boot
mount /dev/mmcblk0p1 /mnt/ota-boot
```

检查 hash：

```bash
sha256sum /mnt/ota-boot/Image /mnt/ota-boot/s5p6818-gec6818-rev01.dtb
```

正确结果：

```text
8f7720d599cf30971a8ccec40e7a646e55e5f22a0c593fb8e8e072b9576a5ff5  /mnt/ota-boot/Image
604728b6baa568df5de139bb523793b915c5f3ca6b74283ee1b876b4d91d8077  /mnt/ota-boot/s5p6818-gec6818-rev01.dtb
```

检查 p3 备份：

```bash
mkdir -p /mnt/ota-data
mount /dev/mmcblk0p3 /mnt/ota-data

find /mnt/ota-data/ota-backup/boot -maxdepth 3 -type f -ls 2>/dev/null
```

应该能看到：

```text
/mnt/ota-data/ota-backup/boot/20260608-115046/Image
/mnt/ota-data/ota-backup/boot/20260608-115046/s5p6818-gec6818-rev01.dtb
```

确认无误后重启：

```bash
sync
umount /mnt/ota-boot 2>/dev/null || true
umount /mnt/ota-data 2>/dev/null || true
sync
reboot
```

---

## 8. 重启后验证

查看内核：

```bash
uname -a
```

查看启动参数：

```bash
cat /proc/cmdline
```

确认 overlay：

```bash
mount | grep overlay
```

再次验证 boot 文件：

```bash
sudo -s
mkdir -p /mnt/boot
mount /dev/mmcblk0p1 /mnt/boot
sha256sum /mnt/boot/Image /mnt/boot/s5p6818-gec6818-rev01.dtb
umount /mnt/boot
```

如果 hash 仍然一致，说明新 `Image` 和 DTB 已经稳定写入。

---

## 9. 本次踩坑

### WSL2 HTTP 服务无法被开发板访问

一开始在 WSL2 里启动：

```bash
python3 -m http.server 8000
```

开发板访问 Windows 主机 IP 时一直卡住。

原因是 WSL2 有自己的虚拟网络，局域网设备不一定能直接访问 WSL2 内部端口。

最终改成 Windows 宿主机直接启动：

```powershell
python -m http.server 8000 --bind 0.0.0.0
```

问题解决。

---

### p1 分区是 64M，但文件系统只有 23M

`lsblk` 显示：

```text
mmcblk0p1   64M
```

但 `df -h` 显示：

```text
/dev/mmcblk0p1   23M   23M     0 100%
```

说明分区本身是 64M，但 ext4 文件系统没有扩展到完整大小。

执行：

```bash
resize2fs /dev/mmcblk0p1
```

扩容后 p1 可用空间恢复正常。

---

### p1 / p3 有残留挂载点

曾经出现 p1 挂载在 `/tmp/bootpart`，p3 同时挂载在 `/mnt/p3` 和 `/mnt/ota-data` 的情况。

OTA 前需要统一清理：

```bash
umount /tmp/bootpart 2>/dev/null || true
umount /mnt/ota-data 2>/dev/null || true
umount /mnt/p3 2>/dev/null || true
umount /mnt/ota-boot 2>/dev/null || true
```

避免 OTA 脚本挂载时状态混乱。

---

### 备份不能放在 p1

最初想把旧 `Image` 备份到 p1，但 p1 是 boot 分区，空间很小。

如果同时存在：

```text
旧 Image
新 Image.new
旧 Image.bak
```

空间很容易爆掉。

最终方案是：

```text
p1 只放启动文件
p3 存放 OTA 备份
```

---

## 10. 当前结论

Stage 1 OTA 已经跑通：

```text
Windows HTTP 固件服务
  ↓
开发板 wget 下载 manifest / sha256 / Image / DTB
  ↓
sha256 校验
  ↓
旧 Image / DTB 备份到 p3
  ↓
新 Image / DTB 写入 p1
  ↓
重启验证
```

这是一个最小可用 OTA 闭环。

后续可以继续扩展：

```text
Stage 2: U-Boot active_slot 双启动
Stage 2: bootcount 自动回滚
Stage 3: ramdisk 改造
Stage 3: rootfs A/B 或 overlay 增量升级
```
---

## 11. 脚本(这个是直接一键终端执行)
```bash
cat > /usr/sbin/ota_stage1_update.sh <<'EOF'
#!/bin/sh
#
# GEC6818 OTA Stage 1
#
# 当前版本：只建议升级 Image + DTB
#
# 关键策略：
#   1. 从 HTTP 下载 manifest.txt / sha256sums.txt / Image / DTB
#   2. 先 sha256 校验
#   3. 把旧 Image / 旧 DTB 备份到 p3，而不是备份到 p1
#   4. p1 只保留当前启动文件，避免 64MB boot 分区空间爆掉
#
# 用法：
#   sudo /usr/sbin/ota_stage1_update.sh http://192.168.88.1:8000
#

set -eu

BASE_URL="${1:-}"
WORKDIR="/tmp/ota-stage1"

BOOT_DEV="/dev/mmcblk0p1"
DATA_DEV="/dev/mmcblk0p3"

BOOT_MNT="/mnt/ota-boot"
DATA_MNT="/mnt/ota-data"

BOOT_IMAGE_NAME="Image"
BOOT_DTB_NAME="s5p6818-gec6818-rev01.dtb"
BOOT_RAMDISK_NAME="ramdisk.img"

log() {
    echo "[OTA-STAGE1] $*"
}

die() {
    echo "[OTA-STAGE1][ERROR] $*" >&2
    exit 1
}

need_cmd() {
    command -v "$1" >/dev/null 2>&1 || die "missing command: $1"
}

get_manifest_value() {
    key="$1"
    grep "^${key}=" "$WORKDIR/manifest.txt" | head -n 1 | cut -d= -f2-
}

download_file() {
    file="$1"
    url="${BASE_URL}/${file}"

    log "Downloading $url"
    wget -T 30 -t 2 -O "$WORKDIR/$file" "$url" || die "download failed: $url"
}

cleanup() {
    set +e
    mountpoint -q "$BOOT_MNT" 2>/dev/null && umount "$BOOT_MNT"
    mountpoint -q "$DATA_MNT" 2>/dev/null && umount "$DATA_MNT"
}
trap cleanup EXIT INT TERM

if [ -z "$BASE_URL" ]; then
    echo "Usage:"
    echo "  $0 http://192.168.88.1:8000"
    exit 1
fi

BASE_URL="${BASE_URL%/}"

need_cmd wget
need_cmd sha256sum
need_cmd mount
need_cmd umount
need_cmd cp
need_cmd mv
need_cmd rm
need_cmd sync
need_cmd df
need_cmd ls

log "Base URL: $BASE_URL"

rm -rf "$WORKDIR"
mkdir -p "$WORKDIR"

download_file "manifest.txt"
download_file "sha256sums.txt"

VERSION="$(get_manifest_value VERSION || echo unknown)"
BOARD="$(get_manifest_value BOARD || echo unknown)"

UPDATE_IMAGE="$(get_manifest_value UPDATE_IMAGE || echo 0)"
UPDATE_DTB="$(get_manifest_value UPDATE_DTB || echo 0)"
UPDATE_RAMDISK="$(get_manifest_value UPDATE_RAMDISK || echo 0)"
UPDATE_ROOTFS="$(get_manifest_value UPDATE_ROOTFS || echo 0)"

IMAGE_FILE="$(get_manifest_value IMAGE_FILE || echo Image)"
DTB_FILE="$(get_manifest_value DTB_FILE || echo s5p6818-gec6818-rev01.dtb)"
RAMDISK_FILE="$(get_manifest_value RAMDISK_FILE || echo ramdisk.img)"
ROOTFS_TAR="$(get_manifest_value ROOTFS_TAR || echo rootfs-update.tar.gz)"

log "Version: $VERSION"
log "Board: $BOARD"
log "UPDATE_IMAGE=$UPDATE_IMAGE"
log "UPDATE_DTB=$UPDATE_DTB"
log "UPDATE_RAMDISK=$UPDATE_RAMDISK"
log "UPDATE_ROOTFS=$UPDATE_ROOTFS"

if [ "$BOARD" != "WC-gec6818" ]; then
    die "board mismatch: expected WC-gec6818, got $BOARD"
fi

[ "$UPDATE_IMAGE" = "1" ] && download_file "$IMAGE_FILE"
[ "$UPDATE_DTB" = "1" ] && download_file "$DTB_FILE"
[ "$UPDATE_RAMDISK" = "1" ] && download_file "$RAMDISK_FILE"
[ "$UPDATE_ROOTFS" = "1" ] && download_file "$ROOTFS_TAR"

log "Checking sha256..."
(
    cd "$WORKDIR"
    sha256sum -c sha256sums.txt
) || die "sha256 check failed"

log "sha256 OK"

mkdir -p "$BOOT_MNT"
mkdir -p "$DATA_MNT"

if [ "$UPDATE_IMAGE" = "1" ] || [ "$UPDATE_DTB" = "1" ] || [ "$UPDATE_RAMDISK" = "1" ]; then
    log "Mounting boot partition: $BOOT_DEV"
    mount "$BOOT_DEV" "$BOOT_MNT"

    log "Mounting data partition for backup: $DATA_DEV"
    mount "$DATA_DEV" "$DATA_MNT"

    TS="$(date +%Y%m%d-%H%M%S 2>/dev/null || echo backup)"
    BACKUP_DIR="$DATA_MNT/ota-backup/boot/$TS"
    mkdir -p "$BACKUP_DIR"

    log "Boot partition before update:"
    df -h "$BOOT_MNT" || true
    ls -lh "$BOOT_MNT" || true

    # 清理上次失败可能留下的临时文件。
    rm -f "$BOOT_MNT"/*.new
    rm -f "$BOOT_MNT"/.new
    rm -f "$BOOT_MNT"/Image.bak.*
    rm -f "$BOOT_MNT"/s5p6818-gec6818-rev01.dtb.bak.*
    rm -f "$BOOT_MNT"/ramdisk.img.bak.*
    sync

    # 把旧文件备份到 p3，不再占用 p1 空间。
    if [ "$UPDATE_IMAGE" = "1" ] && [ -f "$BOOT_MNT/$BOOT_IMAGE_NAME" ]; then
        cp -a "$BOOT_MNT/$BOOT_IMAGE_NAME" "$BACKUP_DIR/$BOOT_IMAGE_NAME"
        log "Old Image backed up to: $BACKUP_DIR/$BOOT_IMAGE_NAME"
    fi

    if [ "$UPDATE_DTB" = "1" ] && [ -f "$BOOT_MNT/$BOOT_DTB_NAME" ]; then
        cp -a "$BOOT_MNT/$BOOT_DTB_NAME" "$BACKUP_DIR/$BOOT_DTB_NAME"
        log "Old DTB backed up to: $BACKUP_DIR/$BOOT_DTB_NAME"
    fi

    if [ "$UPDATE_RAMDISK" = "1" ] && [ -f "$BOOT_MNT/$BOOT_RAMDISK_NAME" ]; then
        cp -a "$BOOT_MNT/$BOOT_RAMDISK_NAME" "$BACKUP_DIR/$BOOT_RAMDISK_NAME"
        log "Old ramdisk backed up to: $BACKUP_DIR/$BOOT_RAMDISK_NAME"
    fi

    sync

    # 更新 Image。
    # 注意：这里仍然先写 Image.new，再 mv 成 Image。
    # 这样至少避免写到一半直接覆盖原文件。
    if [ "$UPDATE_IMAGE" = "1" ]; then
        cp "$WORKDIR/$IMAGE_FILE" "$BOOT_MNT/$BOOT_IMAGE_NAME.new"
        sync
        mv "$BOOT_MNT/$BOOT_IMAGE_NAME.new" "$BOOT_MNT/$BOOT_IMAGE_NAME"
        sync
        log "Image updated: $BOOT_IMAGE_NAME"
    fi

    # 更新 DTB。
    if [ "$UPDATE_DTB" = "1" ]; then
        cp "$WORKDIR/$DTB_FILE" "$BOOT_MNT/$BOOT_DTB_NAME.new"
        sync
        mv "$BOOT_MNT/$BOOT_DTB_NAME.new" "$BOOT_MNT/$BOOT_DTB_NAME"
        sync
        log "DTB updated: $BOOT_DTB_NAME"
    fi

    # 当前阶段不建议升级 ramdisk。
    if [ "$UPDATE_RAMDISK" = "1" ]; then
        cp "$WORKDIR/$RAMDISK_FILE" "$BOOT_MNT/$BOOT_RAMDISK_NAME.new"
        sync
        mv "$BOOT_MNT/$BOOT_RAMDISK_NAME.new" "$BOOT_MNT/$BOOT_RAMDISK_NAME"
        sync
        log "ramdisk updated: $BOOT_RAMDISK_NAME"
    fi

    log "Boot partition after update:"
    df -h "$BOOT_MNT" || true
    ls -lh "$BOOT_MNT" || true

    log "Backup saved at:"
    echo "$BACKUP_DIR"
    ls -lh "$BACKUP_DIR" || true

    sync
    umount "$BOOT_MNT"
    umount "$DATA_MNT"
fi

if [ "$UPDATE_ROOTFS" = "1" ]; then
    die "rootfs update is not enabled in this simplified Stage 1 script yet"
fi

log "OTA Stage 1 finished successfully"
log "Image/DTB updated. Run: sync && reboot"
EOF

chmod +x /usr/sbin/ota_stage1_update.sh

```

---
