# Proxmox VE 安装 U 盘制作指南

## 目录
- [简介](#简介)
- [macOS 系统](#macos-系统)
  - [前提条件](#macos-前提条件)
  - [步骤概览](#macos-步骤概览)
  - [详细步骤](#macos-详细步骤)
    - [1. 识别 U 盘](#1-识别-u-盘)
    - [2. 转换 ISO 镜像](#2-转换-iso-镜像)
    - [3. 卸载 U 盘](#3-卸载-u-盘)
    - [4. 写入镜像](#4-写入镜像)
    - [5. 弹出 U 盘](#5-弹出-u-盘)
  - [验证安装](#macos-验证安装)
  - [常见问题](#macos-常见问题)
- [Windows 系统](#windows-系统)
  - [前提条件](#windows-前提条件)
  - [步骤概览](#windows-步骤概览)
  - [详细步骤](#windows-详细步骤) 
  - [验证安装](#windows-验证安装)
  - [常见问题](#windows-常见问题)
- [Linux 系统](#linux-系统)
  - [前提条件](#linux-前提条件)
  - [步骤概览](#linux-步骤概览)
  - [详细步骤](#linux-详细步骤)
  - [验证安装](#linux-验证安装)
  - [常见问题](#linux-常见问题)
- [参考资料](#参考资料)

## 简介

本指南详细介绍了如何在不同操作系统下将 Proxmox VE ISO 镜像写入 U 盘，制作可启动的 Proxmox VE 安装媒介。Proxmox VE 是一个开源的服务器虚拟化环境，基于 Debian Linux 开发，集成了 KVM 虚拟化和 LXC 容器技术。

## macOS 系统

### macOS 前提条件

- macOS 操作系统（适用于所有版本）
- 管理员权限
- Proxmox VE ISO 镜像文件（本例使用 Proxmox VE 8.4-1.iso）
- 容量至少 4GB 的 U 盘（建议 8GB 以上）
- 稳定的互联网连接（如需下载 ISO）

### macOS 步骤概览

1. 识别系统中的 U 盘设备
2. 转换 ISO 为 macOS 可写入的 DMG 格式
3. 卸载 U 盘
4. 使用 dd 命令写入镜像
5. 安全弹出 U 盘

### macOS 详细步骤

#### 1. 识别 U 盘

首先插入 U 盘，然后在终端中使用以下命令识别 U 盘的设备号：

```bash
diskutil list
```

输出示例：
```
/dev/disk0 (internal, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      GUID_partition_scheme                        *500.3 GB   disk0
   ...

/dev/disk5 (external, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      GUID_partition_scheme                        *64.0 GB    disk5
   ...
```

在此例中，U 盘被识别为 `/dev/disk5`。请记下您的 U 盘设备号，后续步骤中将使用。

> ⚠️ **警告**：确保正确识别 U 盘设备号，以免误操作其他磁盘。外部存储设备通常以 `external` 标识。

#### 2. 转换 ISO 镜像

macOS 需要将 ISO 文件转换为 DMG 格式才能写入 U 盘。使用以下命令：

```bash
cd ~/Downloads  # 假设 ISO 文件在 Downloads 目录
hdiutil convert "Proxmox VE 8.4-1.iso" -format UDRW -o Proxmox-VE_8.4-1.dmg
```

此命令会在同一目录下创建一个 `Proxmox-VE_8.4-1.dmg` 文件。

#### 3. 卸载 U 盘

在写入镜像前，需要卸载 U 盘的所有分区：

```bash
diskutil unmountDisk /dev/disk5  # 使用您的 U 盘设备号
```

成功卸载后，您将看到 `Unmount of all volumes on disk5 was successful` 消息。

#### 4. 写入镜像

使用 `dd` 命令将转换后的 DMG 文件写入 U 盘。此命令需要管理员权限：

```bash
sudo dd if=Proxmox-VE_8.4-1.dmg of=/dev/rdisk5 bs=1m status=progress
```

> ⚠️ **注意**：
> - 使用 `rdisk` 而非 `disk` 可显著提高写入速度
> - 请将 `disk5` 替换为您的 U 盘设备号
> - 根据 U 盘速度和大小，此过程可能需要几分钟到几十分钟不等

写入完成后，您将看到类似以下输出：
```
1499+1 records in
1499+1 records out
1571895296 bytes transferred in 10.411651 secs (150974643 bytes/sec)
```

#### 5. 弹出 U 盘

完成写入后，安全弹出 U 盘：

```bash
diskutil eject /dev/disk5  # 使用您的 U 盘设备号
```

成功弹出后，您可以物理移除 U 盘。

### macOS 验证安装

要验证 U 盘是否正确写入，请重新插入 U 盘，系统应自动挂载分区。然后：

1. 检查挂载的分区：
   ```bash
   mount | grep PVE
   ```

2. 浏览内容验证文件完整性：
   ```bash
   ls -la /Volumes/PVE
   ls -la /Volumes/PVE/boot
   ls -la /Volumes/PVE/proxmox
   ```

3. 检查启动配置文件：
   ```bash
   head -20 /Volumes/PVE/boot/grub/grub.cfg
   cat /Volumes/PVE/boot/grub/grub.cfg | grep menuentry
   ```

成功写入的 U 盘应包含以下关键文件和目录：
- `pve-base.squashfs` 和 `pve-installer.squashfs`（系统核心文件）
- `boot` 目录（含 `linux26` 内核和 `initrd.img`）
- `grub.cfg` 配置文件，含 "Install Proxmox VE" 等启动选项
- `proxmox/packages` 目录（安装所需软件包）

### macOS 常见问题

#### 问题：无法卸载分区
**解决方案**：关闭所有使用 U 盘的应用程序，然后重试卸载命令。如问题持续，可尝试重启计算机。

#### 问题：写入过程中出现 "Resource busy" 错误
**解决方案**：确保已完全卸载 U 盘所有分区。重启电脑可解决顽固的资源占用问题。

#### 问题：dd 命令报错 "Permission denied"
**解决方案**：确保使用 `sudo` 运行 dd 命令，并输入正确的管理员密码。

#### 问题：写入速度非常慢
**解决方案**：确保使用 `/dev/rdisk` 而非 `/dev/disk`。r 前缀可显著提高写入性能。

## Windows 系统

### Windows 前提条件

- Windows 7/8/10/11 操作系统
- 管理员权限
- Proxmox VE ISO 镜像文件（本例使用 Proxmox VE 8.4-1.iso）
- 容量至少 4GB 的 U 盘（建议 8GB 以上）
- Rufus 工具（免费开源的 USB 启动盘制作工具）
- 稳定的互联网连接（如需下载 ISO 和 Rufus）

### Windows 步骤概览

1. 下载并安装 Rufus 工具
2. 插入 U 盘
3. 使用 Rufus 将 ISO 镜像写入 U 盘
4. 等待写入完成并安全移除 U 盘

### Windows 详细步骤

#### 1. 下载 Rufus

访问 [Rufus 官方网站](https://rufus.ie/) 下载最新版本的 Rufus。有两个版本可供选择：
- 安装版（需要安装到系统）
- 便携版（无需安装，下载后直接运行）

建议选择便携版（portable），下载完成后直接运行即可。

#### 2. 准备 U 盘

将 U 盘插入电脑的 USB 接口，确保 U 盘中没有重要数据，因为制作过程会格式化 U 盘上的所有数据。

#### 3. 运行 Rufus 并配置

1. 以管理员身份运行 Rufus
2. 在设备选项中，选择您的 U 盘
3. 在"引导类型选择"中，点击"选择"按钮，找到并选择 Proxmox VE ISO 文件
4. 确保分区方案设置为"MBR"
5. 目标系统类型选择"BIOS 或 UEFI"
6. 文件系统选择"Large FAT32"（通常是自动选择的最佳选项）
7. 集群大小保持默认

#### 4. 开始制作过程

1. 确认所有设置正确后，点击"开始"按钮
2. Rufus 会警告您 U 盘上的所有数据将被删除，点击"确定"继续
3. 如果提示下载额外文件，请允许 Rufus 下载
4. 等待写入过程完成，这可能需要几分钟时间

#### 5. 完成制作

当操作完成后，Rufus 会显示"就绪"状态。现在您可以：
1. 点击"关闭"按钮退出 Rufus
2. 安全移除 U 盘（在系统托盘中点击"安全删除硬件"图标）

### Windows 验证安装

Windows 不会自动挂载 Linux 分区，所以无法直接在 Windows 中查看 U 盘内的 Linux 分区内容。但您可以：

1. 检查 U 盘是否包含可见的 FAT32 分区，并查看是否有以下文件：
   - `boot` 目录
   - `EFI` 目录（用于 UEFI 启动）
   - 其他 Proxmox 相关文件

2. 最可靠的验证方法是使用此 U 盘尝试启动服务器：
   - 将 U 盘插入目标服务器
   - 在服务器 BIOS/UEFI 中设置从 USB 设备启动
   - 检查是否成功进入 Proxmox VE 安装界面

### Windows 常见问题

#### 问题：Rufus 无法识别 U 盘
**解决方案**：
- 尝试使用不同的 USB 接口
- 检查 U 盘是否正常工作
- 在 Windows 磁盘管理中检查 U 盘是否被识别

#### 问题：写入过程中出现错误
**解决方案**：
- 确保 ISO 文件完整，尝试重新下载
- 检查 U 盘是否有物理损坏
- 尝试使用管理员权限运行 Rufus

#### 问题：创建的 U 盘无法启动
**解决方案**：
- 确保目标服务器支持 USB 启动
- 在 BIOS/UEFI 设置中正确配置启动顺序
- 尝试在 Rufus 中使用不同的分区方案（例如尝试 GPT 代替 MBR）

## Linux 系统

### Linux 前提条件

- 任何 Linux 发行版（Ubuntu, Debian, Fedora, CentOS 等）
- root 权限或 sudo 访问权限
- Proxmox VE ISO 镜像文件（本例使用 Proxmox VE 8.4-1.iso）
- 容量至少 4GB 的 U 盘（建议 8GB 以上）
- 稳定的互联网连接（如需下载 ISO）

### Linux 步骤概览

1. 识别 U 盘设备
2. 卸载 U 盘分区
3. 使用 dd 命令直接写入 ISO 镜像
4. 同步文件系统并安全移除 U 盘

### Linux 详细步骤

#### 1. 识别 U 盘设备

插入 U 盘后，通过以下命令确定 U 盘的设备标识：

```bash
lsblk
```

或者：

```bash
sudo fdisk -l
```

输出示例：
```
sda      8:0    0 500.1G  0 disk
├─sda1   8:1    0   512M  0 part /boot/efi
├─sda2   8:2    0   244M  0 part /boot
└─sda3   8:3    0 499.4G  0 part /

sdb      8:16   1  14.9G  0 disk
└─sdb1   8:17   1  14.9G  0 part
```

在此例中，`sdb` 是 U 盘（注意其大小和外部设备特性）。请记下您的 U 盘设备标识符（例如 `/dev/sdb`），后续步骤中将使用。

> ⚠️ **警告**：确保正确识别 U 盘设备，以免误操作写入到系统硬盘。使用设备大小、是否可移动等特征确认。

#### 2. 卸载 U 盘分区

在写入前必须卸载 U 盘上的所有分区，防止数据损坏：

```bash
sudo umount /dev/sdb1   # 卸载第一个分区，如有多个分区，分别卸载
sudo umount /dev/sdb2   # 如果存在其他分区
```

或者使用此命令卸载设备的所有分区：

```bash
sudo umount /dev/sdb*
```

#### 3. 写入 ISO 镜像

使用 `dd` 命令将 ISO 文件直接写入 U 盘：

```bash
sudo dd if=/path/to/Proxmox-VE_8.4-1.iso of=/dev/sdb bs=4M status=progress oflag=sync
```

请替换：
- `/path/to/Proxmox-VE_8.4-1.iso` 为 ISO 文件的实际路径
- `/dev/sdb` 为您的 U 盘设备标识符

> ⚠️ **注意**：
> - `bs=4M` 设置块大小，提高写入速度
> - `status=progress` 显示写入进度
> - `oflag=sync` 确保数据实际写入设备
> - 写入过程可能需要几分钟到几十分钟，取决于 U 盘速度

#### 4. 完成写入

写入完成后，运行以下命令确保所有数据都已写入 U 盘：

```bash
sync
```

现在您可以安全移除 U 盘：

```bash
sudo eject /dev/sdb
```

### Linux 验证安装

为了验证 U 盘是否正确写入，您可以：

1. 重新插入 U 盘，Linux 通常会自动挂载分区
2. 查看挂载的分区：
   ```bash
   mount | grep -i proxmox
   ```
   或
   ```bash
   lsblk -f
   ```

3. 浏览 U 盘内容以验证关键文件：
   ```bash
   ls -la /media/$USER/PVE/        # 路径可能因发行版不同而异
   ls -la /media/$USER/PVE/boot/
   ls -la /media/$USER/PVE/proxmox/
   ```

4. 查看启动配置：
   ```bash
   cat /media/$USER/PVE/boot/grub/grub.cfg | grep menuentry
   ```

### Linux 常见问题

#### 问题：dd 命令完成但 U 盘无法启动
**解决方案**：
- 确保写入了正确的设备（比如 `/dev/sdb` 而不是 `/dev/sdb1`）
- 检查 ISO 文件是否完整，可通过校验和验证
- 在写入后使用 `sync` 命令确保所有数据已写入

#### 问题：权限被拒绝
**解决方案**：
- 确保使用 `sudo` 运行 dd 命令
- 检查用户是否有足够权限

#### 问题：无法卸载 U 盘分区
**解决方案**：
- 关闭所有使用 U 盘的应用程序
- 使用 `lsof | grep /dev/sdb` 查找使用 U 盘的进程
- 如果问题持续，可尝试 `sudo fuser -km /dev/sdb*` 终止使用设备的进程

## 参考资料

- [Proxmox VE 官方安装媒介准备指南](https://pve.proxmox.com/wiki/Prepare_Installation_Media)
- [Proxmox VE 官网](https://www.proxmox.com/proxmox-ve)
- [Rufus 官网](https://rufus.ie/)
- [Linux dd 命令参考](https://man7.org/linux/man-pages/man1/dd.1.html)
- [macOS dd 命令参考](https://ss64.com/osx/dd.html) 