# 分区之名：文件系统标签与挂载点路径之改

## 环境

- 系统：Fedora 44
- 桌面：KDE Plasma（Wayland）
- 内核：7.2.9-200.fc44.x86_64
- 相关硬件：内置 NVMe 之 XFS 数据分区（他种文件系统工具异而理同，见「疑难」）
- 相关软体：xfsprogs 7.1.1（`xfs_io`、`xfs_admin`）；udisks2
- 前提：分区已由 fstab 固定挂载（法见 `fedora/storage/partition-automount-run-media.md`）；有 sudo 之权
- 撰写：2026-10-12

## 适用

- 可用：欲为分区起可读之名。名分两层，各自独立：
  - **文件系统标签**（label）：udisks、Dolphin 等所见之「盘名」；
  - **挂载点路径**：盘挂于何处（如 `/run/media/<用户>/Data`）。
- 背景：格式化后无标签之分区，udisks 命名规则为——**有标签用标签，无标签用 UUID**，故初时得 `eb81928a-…` 之长名。
- 不可用：XFS 标签**至多 12 字符**；他种文件系统各有其工具与限制。

## 步骤

### 甲、文件系统标签（显示名）

读之：

```bash
xfs_io -c 'label' /run/media/<用户>/<挂载点>
```

写之，二法：

```bash
# 法一（联机，免卸载；新内核之 XFS 支持联机改标签）
sudo xfs_io -c 'label -s Data' /run/media/<用户>/<挂载点>

# 法二（离线；xfs_admin 明言 mounted 不可改，须先卸载）
sudo umount /run/media/<用户>/<挂载点>
sudo xfs_admin -L Data /dev/<设备>        # 如 /dev/nvme1n1p1
sudo mount /run/media/<用户>/<挂载点>
```

清空标签：`xfs_admin -L --`（或 `xfs_io -c 'label -c'`）。

> 长逾十二字符：`xfs_io` 直接报错；`xfs_admin` 截断且警告。宜 ASCII、无空格；中文亦可（一汉字三字节，至多四字）。

### 乙、挂载点路径（fstab 所管之盘）

```bash
# 一、卸载（窗、终端驻足其中者先关，否则报 busy）
sudo umount /run/media/<用户>/<旧挂载点>       # 旧挂载点即形如 UUID 之目录

# 二、改 fstab 与 tmpfiles 之路径（以全前缀替换，则 UUID= 之引用不致误伤）
sudo sed -i 's|/run/media/<用户>/<旧挂载点>|/run/media/<用户>/Data|g' \
  /etc/fstab /etc/tmpfiles.d/run-media-<用户>.conf

# 三、生效并重挂
sudo systemctl daemon-reload
sudo systemd-tmpfiles --create /etc/tmpfiles.d/run-media-<用户>.conf
sudo mount /run/media/<用户>/Data

# 四、除旧空目录，并验
rmdir /run/media/<用户>/<旧挂载点>
findmnt /run/media/<用户>/Data
```

## 验证

```bash
xfs_io -c 'label' /run/media/<用户>/Data    # 当显示 label = "Data"
findmnt /run/media/<用户>/Data              # 当见 rw 挂载
```

重启一次再验，以证 fstab 与 tmpfiles 之改已固。

## 疑难

- **两层各自独立**：改标签不迁挂载点（fstab 路径写死）；仅欲显示名好看者，行甲法足矣；欲路径亦雅，则甲乙并施。
- **fstab 仍以 `UUID=` 识器**：改标签于挂载无碍，此为稳妥之引；切勿改 UUID 本身（`xfs_admin -U`）。
- **显示滞后**：Dolphin/KDE 或须重登、重插方刷新；`lsblk` 之 LABEL 列亦可能因 udev 缓存滞后而暂不显示——以 `xfs_io -c 'label'` 读之方为准。
- **他种文件系统**之标签工具：NTFS 用 `ntfslabel`，ext2/3/4 用 `e2label`（或 `tune2fs -L`），FAT 用 `fatlabel`，btrfs 用 `btrfs filesystem label`。挂载点路径之改法同。
- 改路径后，fstab 中旧注释（如「某用途分区」）宜顺手更正，免日后惑。
- 联机改标签赖内核 `FS_IOC_SETFSLABEL`（新内核与 xfsprogs 皆备；若遇不支持，退用离线之法）。

## 回退

```bash
# 路径改回：依乙法逆施（以旧路径替新路径）
# 标签清空或改回：
sudo xfs_io -c 'label -c' /run/media/<用户>/Data
# 或
sudo umount /run/media/<用户>/Data && sudo xfs_admin -L -- /dev/<设备> && sudo mount /run/media/<用户>/Data
```
