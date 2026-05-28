# OTA_Learn
记录学习OTA过程


# GEC6818 第一次本地 OTA 测试记录

## 1. 测试目标

本次测试目标是验证 GEC6818 在 SD 卡启动环境下是否可以完成最简单的本地 OTA 流程。

本阶段 OTA 不更新 U-Boot，不更新 Kernel，不更新 DTB，也不更新完整 rootfs，只做最小闭环验证：

1. 制作本地 OTA 包
2. 拷贝 OTA 包到开发板
3. 校验 OTA 包
4. 覆盖少量 rootfs 文件
5. 更新版本号
6. 重启后确认系统仍可正常启动
7. 记录 OTA 后出现的问题

本次 OTA 从：

```text
v0.5.0-gpio-led-key-pwm
```

升级到：

```text
v0.6.0-ota-sd-basic
```

---

## 2. OTA 前系统状态

当前系统运行在 SD 卡启动环境中，基础外设已经完成适配：

| 模块 | 状态 | 说明 |
| --- | --- | --- |
| LCD | 正常 | RGB 800x480 显示正常 |
| 触摸 | 正常 | I2C 触摸可用 |
| 声卡 | 正常 | ALSA 设备已适配 |
| 网口 | 正常 | 可通过 SSH 连接 |
| LED | 正常 | D7-D11 已注册到 `/sys/class/leds/` |
| 按键 | 正常 | gpio-keys 已注册 |
| 蜂鸣器 | 正常 | PWM2 beeper 已适配 |
| 启动介质 | SD 卡 | 暂未烧录 eMMC |

OTA 前版本文件：

```bash
cat /etc/gec6818-version
```

输出：

```text
v0.5.0-gpio-led-key-pwm
```

---

## 3. 第一次本地 OTA 包内容

本次 OTA 包名称：

```text
gec6818-ota-v0.6.0.tar.gz
```

OTA 包主要内容：

```text
gec6818-ota-v0.6.0/
├── manifest.txt
├── sha256sum.txt
├── rootfs
│   ├── etc
│   │   └── gec6818-version
│   └── usr
│       └── local
│           └── bin
│               └── gec6818-check.sh
└── scripts
```

其中：

```text
manifest.txt
```

用于描述 OTA 包信息。

```text
sha256sum.txt
```

用于校验 OTA 包内文件完整性。

```text
rootfs/etc/gec6818-version
```

用于更新系统版本号。

```text
rootfs/usr/local/bin/gec6818-check.sh
```

用于 OTA 后检查开发板基础状态。

---

## 4. OTA 升级脚本

开发板上的 OTA 升级脚本为：

```text
/usr/local/bin/ota-local-update.sh
```

核心逻辑如下：

```bash
#!/bin/sh
set -eu

PKG="$1"
WORKDIR="/tmp/gec6818-ota"
BACKUPDIR="/root/ota-backup"

if [ -z "$PKG" ]; then
    echo "Usage: ota-local-update.sh <ota-package.tar.gz>"
    exit 1
fi

if [ ! -f "$PKG" ]; then
    echo "ERROR: OTA package not found: $PKG"
    exit 1
fi

echo "==== CLEAN WORKDIR ===="
rm -rf "$WORKDIR"
mkdir -p "$WORKDIR"

echo "==== EXTRACT PACKAGE ===="
tar -xzf "$PKG" -C "$WORKDIR"

cd "$WORKDIR"

echo "==== CHECK MANIFEST ===="
if [ ! -f manifest.txt ]; then
    echo "ERROR: manifest.txt missing"
    exit 1
fi

cat manifest.txt

if ! grep -q '^board=gec6818$' manifest.txt; then
    echo "ERROR: board is not gec6818"
    exit 1
fi

if ! grep -q '^type=file-update$' manifest.txt; then
    echo "ERROR: unsupported OTA type"
    exit 1
fi

echo "==== CHECK SHA256 ===="
if [ ! -f sha256sum.txt ]; then
    echo "ERROR: sha256sum.txt missing"
    exit 1
fi

sha256sum -c sha256sum.txt

echo "==== BACKUP OLD FILES ===="
mkdir -p "$BACKUPDIR"

if [ -f /etc/gec6818-version ]; then
    cp -a /etc/gec6818-version "$BACKUPDIR/gec6818-version.bak"
fi

if [ -f /usr/local/bin/gec6818-check.sh ]; then
    cp -a /usr/local/bin/gec6818-check.sh "$BACKUPDIR/gec6818-check.sh.bak"
fi

echo "==== INSTALL FILES ===="
if [ ! -d rootfs ]; then
    echo "ERROR: rootfs directory missing"
    exit 1
fi

cp -a rootfs/. /

if [ -f /usr/local/bin/gec6818-check.sh ]; then
    chmod +x /usr/local/bin/gec6818-check.sh
fi

sync

echo "==== OTA DONE ===="
cat /etc/gec6818-version
```

执行 OTA：

```bash
ota-local-update.sh /root/gec6818-ota-v0.6.0.tar.gz
```

OTA 后版本变为：

```bash
cat /etc/gec6818-version
```

输出：

```text
v0.6.0-ota-sd-basic
```

说明第一次本地 OTA 的基本流程已经跑通。

---

## 5. OTA 后出现的问题

OTA 完成并重启后，发现 SSH 无法连接开发板。

现象：

```text
ssh 连接失败
```

在开发板串口终端中检查 SSH 服务状态：

```bash
systemctl status ssh
```

发现 SSH 服务启动失败。

继续执行 SSH 配置检查：

```bash
/usr/sbin/sshd -t
```

输出错误：

```text
Missing privilege separation directory: /var/run/sshd
```

说明 SSHD 启动失败的直接原因是缺少运行时目录：

```text
/var/run/sshd
```

---

## 6. 问题原因分析

SSH 服务启动时需要使用权限分离目录：

```text
/var/run/sshd
```

如果这个目录不存在，`sshd` 会直接启动失败。

本次 OTA 后 SSH 失败的根因是：

```text
/var/run/sshd 目录不存在
```

在 Linux 系统中：

```text
/var/run
```

通常是运行时目录，很多系统中它实际指向：

```text
/run
```

而 `/run` 通常是 tmpfs，也就是内存文件系统。

这类目录不是普通持久化文件目录，重启后需要由 systemd、tmpfiles 或服务脚本重新创建。

本次 OTA 使用了类似下面的方式覆盖 rootfs：

```bash
cp -a rootfs/. /
```

这个动作本身不会主动删除 `/var/run/sshd`，但是 OTA 后系统运行环境发生变化，导致 SSH 所需的运行时目录没有被正确创建。

因此 SSH 服务启动时找不到：

```text
/var/run/sshd
```

最终报错：

```text
Missing privilege separation directory: /var/run/sshd
```

总结原因：

```text
OTA 后 SSHD 所需的运行时目录没有创建，导致 ssh 服务启动失败。
```

---

## 7. 临时修复方法

通过串口登录开发板后，手动创建 SSH 运行时目录：

```bash
mkdir -p /var/run/sshd
```

然后重启 SSH 服务：

```bash
systemctl restart ssh
```

查看 SSH 服务状态：

```bash
systemctl status ssh
```

如果状态正常，应该能看到 SSH 服务已经处于运行状态。

再次检查 SSH 配置：

```bash
/usr/sbin/sshd -t
```

如果没有任何输出，说明 SSHD 配置检查通过。

然后在电脑上重新连接开发板：

```bash
ssh root@开发板IP
```

---

## 8. 如果还有 SSH 密钥问题

如果 `sshd -t` 继续报密钥相关错误，可以重新生成主机密钥：

```bash
ssh-keygen -A
```

然后再次检查：

```bash
/usr/sbin/sshd -t
```

再重启 SSH：

```bash
systemctl restart ssh
systemctl status ssh
```

一般完整修复流程为：

```bash
mkdir -p /var/run/sshd
ssh-keygen -A
/usr/sbin/sshd -t
systemctl restart ssh
systemctl status ssh
```

---

## 9. 一劳永逸的 OTA 脚本修复

为了避免下次 OTA 后 SSH 再次失败，需要在 OTA 脚本安装文件后增加 SSH 运行时目录修复。

在：

```bash
cp -a rootfs/. /
```

后面增加：

```bash
mkdir -p /var/run/sshd
chmod 0755 /var/run/sshd
```

建议修改后的 OTA 安装部分如下：

```bash
echo "==== INSTALL FILES ===="
if [ ! -d rootfs ]; then
    echo "ERROR: rootfs directory missing"
    exit 1
fi

cp -a rootfs/. /

if [ -f /usr/local/bin/gec6818-check.sh ]; then
    chmod +x /usr/local/bin/gec6818-check.sh
fi

echo "==== FIX RUNTIME DIRECTORIES ===="
mkdir -p /var/run/sshd
chmod 0755 /var/run/sshd

sync
```

这样每次 OTA 安装完成后，都会确保 SSHD 需要的目录存在。

---

## 10. 更规范的 systemd 修复方式

临时在 OTA 脚本里创建目录可以解决问题，但更规范的方式是让 systemd 在启动时自动创建 `/run/sshd`。

可以创建 tmpfiles 配置：

```bash
cat > /etc/tmpfiles.d/sshd.conf << 'EOF'
d /run/sshd 0755 root root -
EOF
```

然后立即应用：

```bash
systemd-tmpfiles --create /etc/tmpfiles.d/sshd.conf
```

检查目录：

```bash
ls -ld /run/sshd /var/run/sshd
```

如果 `/var/run` 指向 `/run`，则 `/var/run/sshd` 也会正常存在。

之后重启 SSH：

```bash
systemctl restart ssh
systemctl status ssh
```

建议 OTA 脚本中也可以加入：

```bash
if command -v systemd-tmpfiles >/dev/null 2>&1; then
    systemd-tmpfiles --create /etc/tmpfiles.d/sshd.conf || true
fi

mkdir -p /var/run/sshd
chmod 0755 /var/run/sshd
```

这样即使 `systemd-tmpfiles` 没有成功执行，也会用 `mkdir` 兜底。

---

## 11. 推荐更新后的 ota-local-update.sh

建议把 OTA 脚本更新为下面这个版本：

```bash
#!/bin/sh
set -eu                              # 遇到错误立即退出；变量未定义时报错

# ============ 参数检查 ============
if [ $# -lt 1 ]; then
    echo "Usage: ota-local-update.sh <ota-package.tar.gz>"
    exit 1
fi

# ============ 变量定义 ============
PKG="$1"                             # OTA 升级包路径（从命令行第 1 个参数传入）
WORKDIR="/tmp/gec6818-ota"           # 临时工作目录
BACKUPDIR="/root/ota-backup"         # 旧文件备份目录

# 检查是否传入了升级包路径
if [ -z "$PKG" ]; then
    echo "Usage: ota-local-update.sh <ota-package.tar.gz>"
    exit 1
fi

# 检查升级包文件是否存在
if [ ! -f "$PKG" ]; then
    echo "ERROR: OTA package not found: $PKG"
    exit 1
fi

echo "==== CLEAN WORKDIR ===="
rm -rf "$WORKDIR"                    # 清空上次残留的工作目录
mkdir -p "$WORKDIR"                  # 重新创建工作目录

echo "==== EXTRACT PACKAGE ===="
tar -xzf "$PKG" -C "$WORKDIR"        # 解压 OTA 升级包到工作目录

cd "$WORKDIR"                        # 进入工作目录

echo "==== CHECK MANIFEST ===="
# 检查清单文件是否存在
if [ ! -f manifest.txt ]; then
    echo "ERROR: manifest.txt missing"
    exit 1
fi

cat manifest.txt                     # 打印清单内容，供用户确认

# 验证目标板型号是否为 gec6818
if ! grep -q '^board=gec6818$' manifest.txt; then
    echo "ERROR: board is not gec6818"
    exit 1
fi

# 验证 OTA 升级类型是否为文件更新（file-update）
if ! grep -q '^type=file-update$' manifest.txt; then
    echo "ERROR: unsupported OTA type"
    exit 1
fi

echo "==== CHECK SHA256 ===="
# 检查校验和文件是否存在
if [ ! -f sha256sum.txt ]; then
    echo "ERROR: sha256sum.txt missing"
    exit 1
fi

# 校验所有文件的 SHA256 哈希值，确保包完整性
sha256sum -c sha256sum.txt

echo "==== BACKUP OLD FILES ===="
mkdir -p "$BACKUPDIR"                # 创建备份目录

# 备份版本信息文件
if [ -f /etc/gec6818-version ]; then
    cp -a /etc/gec6818-version "$BACKUPDIR/gec6818-version.bak"
fi

# 备份旧的检查脚本
if [ -f /usr/local/bin/gec6818-check.sh ]; then
    cp -a /usr/local/bin/gec6818-check.sh "$BACKUPDIR/gec6818-check.sh.bak"
fi

# 备份 SSH tmpfiles 配置（用于 systemd 自动创建 /run/sshd 目录）
if [ -f /etc/tmpfiles.d/sshd.conf ]; then
    cp -a /etc/tmpfiles.d/sshd.conf "$BACKUPDIR/sshd.conf.bak"
fi

echo "==== INSTALL FILES ===="
# 检查 rootfs 目录是否存在（解压后的文件系统目录）
if [ ! -d rootfs ]; then
    echo "ERROR: rootfs directory missing"
    exit 1
fi

# 将 rootfs 目录下的所有文件拷贝到根目录 /，覆盖旧文件
cp -a rootfs/. /

# 确保检查脚本有可执行权限
if [ -f /usr/local/bin/gec6818-check.sh ]; then
    chmod +x /usr/local/bin/gec6818-check.sh
fi

echo "==== FIX SSH RUNTIME DIRECTORY ===="
# 创建 tmpfiles 配置目录（如果不存在）
mkdir -p /etc/tmpfiles.d

# 写入 SSH 运行时目录的 tmpfiles 配置，确保每次启动时自动创建 /run/sshd
# 这是解决 SSH 因缺少权限分离目录而启动失败的关键步骤
cat > /etc/tmpfiles.d/sshd.conf << 'EOF'
d /run/sshd 0755 root root -
EOF

# 如果系统支持 systemd-tmpfiles，立即应用该配置创建目录
if command -v systemd-tmpfiles >/dev/null 2>&1; then
    systemd-tmpfiles --create /etc/tmpfiles.d/sshd.conf || true
fi

# 立即手动创建 SSH 运行时目录，确保当前启动不会失败
mkdir -p /var/run/sshd
chmod 0755 /var/run/sshd

echo "==== CHECK SSHD CONFIG ===="
# 用 sshd -t 验证 SSH 配置文件语法是否正确
if [ -x /usr/sbin/sshd ]; then
    /usr/sbin/sshd -t
fi

sync                                 # 同步磁盘，确保所有写入完成

echo "==== OTA DONE ===="
echo "Current version:"
cat /etc/gec6818-version             # 打印当前版本号

echo
echo "Run check script:"
# 执行升级后的检查脚本（如果存在且可执行）
if [ -x /usr/local/bin/gec6818-check.sh ]; then
    /usr/local/bin/gec6818-check.sh
fi

```

---

## 12. OTA 后必须执行的检查

每次 OTA 后都要检查以下内容。

检查版本：

```bash
cat /etc/gec6818-version
```

检查 SSHD 配置：

```bash
/usr/sbin/sshd -t
```

检查 SSH 服务：

```bash
systemctl status ssh
```

检查 SSH 运行时目录：

```bash
ls -ld /var/run/sshd
ls -ld /run/sshd
```

检查网络：

```bash
ip link
ip addr
```

检查网口日志：

```bash
dmesg | grep -Ei "eth|phy|link|stmmac|gmac"
```

检查 OTA 后基础状态：

```bash
gec6818-check.sh
```

---

## 13. 本次问题结论

本次第一次本地 OTA 流程已经成功完成，版本文件已经从：

```text
v0.5.0-gpio-led-key-pwm
```

升级到：

```text
v0.6.0-ota-sd-basic
```

OTA 后出现 SSH 无法连接问题。

直接原因：

```text
/var/run/sshd 不存在
```

报错信息：

```text
Missing privilege separation directory: /var/run/sshd
```

解决方法：

```bash
mkdir -p /var/run/sshd
systemctl restart ssh
```

长期修复：

1. 在 OTA 脚本中增加 `/var/run/sshd` 创建逻辑
2. 增加 `/etc/tmpfiles.d/sshd.conf`
3. OTA 后执行 `/usr/sbin/sshd -t` 检查
4. OTA 后重启或检查 SSH 服务

---

## 14. 本次学习到的东西

第一次本地 OTA 不能只看版本文件是否更新成功，还必须检查系统关键服务是否还正常。

尤其是以下目录属于运行时目录，OTA 时要特别小心：

```text
/run
/var/run
/tmp
/var/tmp
```

这些目录不应该被当成普通 rootfs 静态文件长期依赖。

对于 SSH 这类远程登录服务，OTA 后必须检查：

```bash
/usr/sbin/sshd -t
systemctl status ssh
```

否则系统虽然启动了，但远程连不上，后续维护会很麻烦。

本次问题说明：OTA 的第一阶段不只是“能复制文件”，还要保证升级后系统仍然可管理、可登录、可恢复。

---

## 15. 下一次 OTA 前的改进清单

下一次 OTA 前，必须完成以下改进：

1. 更新 `ota-local-update.sh`
2. 增加 `/var/run/sshd` 自动创建
3. 增加 `/etc/tmpfiles.d/sshd.conf`
4. OTA 后自动执行 `/usr/sbin/sshd -t`
5. OTA 后保留 `/root/ota-backup/`
6. OTA 包中不要随便包含 `/run`、`/var/run`、`/tmp`
7. OTA 前后都记录 `systemctl status ssh`
8. OTA 后通过串口和 SSH 双重确认系统可用

---

## 16. 当前建议

当前阶段继续使用 SD 卡启动学习 OTA 是正确的。

因为这次 OTA 后 SSH 出问题，如果是在 eMMC 正式系统上操作，恢复会更麻烦。

所以后续顺序应该是：

```text
1. 继续在 SD 卡系统上完善本地 OTA
2. 修复 SSH 运行时目录问题
3. 增加 OTA 后自检脚本
4. 确认多次 OTA 后系统仍可 SSH 登录
5. 再开始考虑 eMMC 烧录
```
```


我还把原因表述修正得更准确了一点：`cp -a rootfs/. /` 本身不一定会删除 `/var/run/sshd`，真正关键是 `/var/run`、`/run` 属于运行时目录，OTA 后 SSH 所需目录没有被正确创建，所以 `sshd` 启动失败。这样写更适合作为技术记录。
