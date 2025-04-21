# Ubuntu 系统第二块 SSD 挂载配置指南

本文档介绍如何在 Ubuntu 系统中挂载额外的 SSD 硬盘作为数据盘。

## 准备工作

在开始之前，确保你有以下条件：
- 已安装的 Ubuntu 系统
- 一个额外的 SSD 硬盘，已正确连接到系统
- 管理员权限（sudo）

## 步骤一：确认 SSD 硬盘状态

首先，我们需要确认系统识别了两块 SSD 硬盘。使用以下命令列出所有存储设备：

```bash
lsblk
```

或者，获取更详细的信息：

```bash
sudo fdisk -l
```

你应该能看到类似下面的输出，显示多个 nvme 设备，例如 `nvme0n1` 和 `nvme1n1`。

## 步骤二：创建分区

以下步骤使用 `fdisk` 工具创建新分区：

1. 打开 fdisk 工具处理目标硬盘（假设是 `/dev/nvme0n1`）：

```bash
sudo fdisk /dev/nvme0n1
```

2. 在 fdisk 交互界面中输入以下命令：
   - 输入 `n` 创建新分区
   - 输入 `p` 选择主分区
   - 输入 `1` 设置分区号
   - 对于起始扇区和结束扇区，直接按 Enter 使用默认值（使用整个磁盘）
   - 输入 `w` 保存更改并退出

3. 验证分区创建成功：

```bash
sudo fdisk -l /dev/nvme0n1
```

应该能看到新创建的分区（例如 `/dev/nvme0n1p1`）。

## 步骤三：格式化分区

使用 ext4 文件系统格式化新创建的分区：

```bash
sudo mkfs.ext4 /dev/nvme0n1p1
```

## 步骤四：创建挂载点

创建要挂载 SSD 的目录：

```bash
sudo mkdir -p /data
```

## 步骤五：临时挂载测试

先临时挂载以测试是否正常工作：

```bash
sudo mount /dev/nvme0n1p1 /data
```

验证挂载成功：

```bash
df -h | grep /data
```

## 步骤六：设置权限

设置适当的所有权和权限，让普通用户可以访问：

```bash
sudo chown -R $USER:$USER /data
```

## 步骤七：配置开机自动挂载

编辑 `/etc/fstab` 文件，添加以下行：

```bash
sudo bash -c 'echo "/dev/nvme0n1p1 /data ext4 defaults 0 2" >> /etc/fstab'
```

## 步骤八：测试挂载点

创建测试文件以验证权限设置正确：

```bash
touch /data/test_file
```

验证完成后，可以删除测试文件：

```bash
rm /data/test_file
```

## 注意事项

1. 挂载过程中会创建 `lost+found` 目录，这是 ext4 文件系统用于文件系统恢复的特殊目录，不应删除它。
2. 如果系统重启后挂载失败，可能需要检查 UUID，并在 `/etc/fstab` 中使用 UUID 而不是设备名。
3. 在进行任何分区或格式化操作前，确保没有重要数据需要备份。

## 故障排除

### 挂载失败

如果挂载失败，可以尝试以下步骤：

1. 检查分区是否存在：`sudo fdisk -l`
2. 检查文件系统是否完好：`sudo fsck /dev/nvme0n1p1`
3. 如果使用 UUID 挂载，确认 UUID 是否正确：`sudo blkid /dev/nvme0n1p1`

### 权限问题

如果无法写入挂载点，可能是权限问题：

```bash
sudo chmod 777 /data
```

注意：在生产环境中应使用更严格的权限设置。

## 结论

按照上述步骤，你应该已经成功将额外的 SSD 硬盘挂载为数据盘，并设置为开机自动挂载。现在你可以使用 `/data` 目录存储大量数据，充分利用额外的 SSD 存储空间。 