# 分区开机自动挂载于 /run/media/<用户>/ 之下（tmpfiles.d 加 fstab）

## 环境

- 系统：Fedora 44
- 桌面：KDE Plasma（Wayland 会话）
- 内核：7.2.9-200.fc44.x86_64
- 相关硬件：内置第二块 NVMe 之数据分区，文件系统为 XFS（他种文件系统同理）
- 前提：有 sudo 之权；若还欲于其上挂 FUSE 文件系统（如 onedriver），另见 `fedora/desktop/onedriver-files-on-demand.md`
- 撰写：2026-10-11

## 适用

- 可用：固定之数据分区，欲**开机自动挂载**，且挂载点取其惯用路径 `/run/media/<用户>/<UUID>`（或他处），使路径恒存，供脚本、容器及 FUSE 挂载点引用。
- 不可用／不必用：优盘、移动硬盘等可移动设备——彼等本由 udisks 与桌面按需自动挂载，强入 fstab 反致扞格。

## 背景

`/run/media` 为 tmpfs，重启即空。其中 `<用户>` 之子目录，平日由 systemd-logind（或 udisks）于用户登录时创建；而 fstab 之挂载行于**系统启动早期**即行，此时该子目录尚未有其主。故须以 `tmpfiles.d` 之条目预建目录，fstab 方不至于"挂载点不存在"而败。二者相合，方得稳当。

## 步骤

### 一、查明分区之 UUID 与文件系统

```bash
lsblk -f
```

本文以如下值为示例，实际须按己之输出替换：

- UUID：`eb81928a-e206-4913-a1ff-3a8fda824ff2`
- 文件系统：`xfs`
- 挂载点：`/run/media/<用户>/eb81928a-e206-4913-a1ff-3a8fda824ff2`

### 二、写 tmpfiles 条目

以 root 建 `/etc/tmpfiles.d/run-media-<用户>.conf`，凡二行：首行建父目录，次行建挂载点本体（`<用户>` 须替换为实际用户名）：

```
d /run/media/<用户> 0750 <用户> <用户> -
d /run/media/<用户>/eb81928a-e206-4913-a1ff-3a8fda824ff2 0755 <用户> <用户> -
```

### 三、备份而后增 fstab 一行

```bash
sudo cp /etc/fstab /etc/fstab.bak-$(date +%F)
```

于 `/etc/fstab` 末增：

```
UUID=eb81928a-e206-4913-a1ff-3a8fda824ff2 /run/media/<用户>/eb81928a-e206-4913-a1ff-3a8fda824ff2 xfs defaults,nofail,x-systemd.device-timeout=15s 0 0
```

- `nofail`：设备不在（未装、未通电）时不得阻断启动；
- `x-systemd.device-timeout=15s`：等候设备至多十五秒；
- 末二列以 `0 0` 为宜（XFS 不列开机 fsck 之序）。

### 四、令其即刻生效

```bash
sudo systemctl daemon-reload
sudo systemd-tmpfiles --create /etc/tmpfiles.d/run-media-<用户>.conf
sudo umount /run/media/<用户>/eb81928a-e206-4913-a1ff-3a8fda824ff2   # 若旧挂载在
sudo mount  /run/media/<用户>/eb81928a-e206-4913-a1ff-3a8fda824ff2
```

## 验证

```bash
findmnt /run/media/<用户>/eb81928a-e206-4913-a1ff-3a8fda824ff2
```

当见 `rw` 之选项（若仍见 `ro`，则旧挂载未除，或文件系统有恙，见下节）。重启之后再验一次，以证"开机自动"。

## 疑难

- **分区已被 udisks 以只读（ro）挂载**：常见于桌面文件管理器点击挂载之后。先 `sudo umount <挂载点>`，再依第四节重挂；或径行重启，fstab 自会接管。fstab 既管此设备，udisks 与桌面之后便不再插手。
- **`/run/media` 之语义**：本为临时挂载点（tmpfs），厝固定分区于此，非系统之惯例（惯例为 `/mnt` 或自定义）。然路径既已约定俗成，用之亦无碍，惟须如上以 tmpfiles 兜其目录。
- **其上复挂 FUSE（如 onedriver）**：须先盘后 FUSE；卸载则反之（先 FUSE 后盘），否则盘报 busy。
- 欲"用时方挂"（access 时方挂），可于 fstab 选项加 `x-systemd.automount`，则开机不实挂、访问时方挂之。

## 回退

```bash
sudo cp /etc/fstab.bak-<日期> /etc/fstab        # 或手动删去相应行
sudo rm /etc/tmpfiles.d/run-media-<用户>.conf
sudo systemctl daemon-reload
sudo umount /run/media/<用户>/eb81928a-e206-4913-a1ff-3a8fda824ff2
```
