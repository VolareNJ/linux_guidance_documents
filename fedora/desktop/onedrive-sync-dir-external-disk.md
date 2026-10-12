# onedrive for linux：同步目录设于独立数据盘，及挂载守卫

## 环境

- 系统：Fedora 44
- 桌面：KDE Plasma（Wayland）
- 内核：7.2.9-200.fc44.x86_64
- 相关软体：onedrive（abraunegg/onedrive）2.5.11；systemd 用户服务
- 前提：数据盘由 fstab 固定挂载（见 `fedora/storage/partition-automount-run-media.md`）；有 sudo 之权
- 撰写：2026-10-12

## 适用

- 可用：欲将 onedrive for linux 之 `sync_dir`（同步目录）置于系统所挂之独立盘（如数据盘），并防「盘未挂载而同步误落空目录」之险。
- 须知：此客户端为**全量镜像同步**，非按需挂载（与 onedriver 之别，见 `fedora/desktop/onedriver-files-on-demand.md`）；盘上所得乃云端整份副本。
- 不宜：以 NFS、CIFS、SMB、Samba 等**网络挂载点**为 `sync_dir`——inotify 失效，monitor 模式之实时侦测无从谈起，官方明言宜慎。

## 背景

onedrive for linux 无「挂载点」之说，其对应者为 `sync_dir`，默认 `~/OneDrive`；改之**必须** `--resync`（官方明文）。其险在于：盘若未挂载，`sync_dir` 路径即不存在，客户端便径于空挂载点之下创建之而开始同步——数据遂误落。故官方备有「挂载守卫」：`check_nomount` 配 `.nosync` 文件。

## 步骤

### 一、配置（`~/.config/onedrive/config`）

```
sync_dir = "/run/media/<用户>/Data/OneDrive"
check_nomount = "true"
```

（`<用户>` 与盘名依实况替换；`check_nomount` 默认 false，须显式启用。）

### 二、预埋 `.nosync`（**须于盘未挂载时为之**）

其理：`.nosync` 须落于**挂载点目录本体**（被盘所遮盖之底层）；盘挂载时所写者乃盘面，非底层。官方 TIP 云：「Create a `.nosync` file in your mount point *before* you mount your disk.」

```bash
sudo umount /run/media/<用户>/Data          # 若有窗口、终端驻足其中，先关之
touch /run/media/<用户>/Data/.nosync        # 落于挂载点底层
mkdir -p /run/media/<用户>/Data/OneDrive
touch /run/media/<用户>/Data/OneDrive/.nosync
sudo mount /run/media/<用户>/Data
mkdir -p /run/media/<用户>/Data/OneDrive    # 盘上真身（挂载后创建）
```

> 官方于配置文档言「in your mount point」，而 CLI 帮助言「in the syncdir root」，表述稍异；故挂载点与同步目录两处**兼埋**，以求必中。两枚皆隐于底层，盘挂载时不可见，盘一卸即现形，客户端见之则中止同步。

### 三、验证配置

```bash
onedrive --display-config      # 当见 sync_dir 指向盘中
```

### 四、初次同步

```bash
onedrive --sync --resync       # 会示警告并问 [Y/N]，输 y 以继续
```

（欲先观计划，可加 `--dry-run`。）

### 五、常驻服务（可选）

```bash
systemctl --user enable --now onedrive.service
```

单元在 `/usr/lib/systemd/user/onedrive.service`，其 ExecStart 为 `onedrive --monitor`（登录后延十五秒启动）；若需 resync，则以 126 退出而不自转——故**先手动 resync 毕，再启服务**。

## 验证

- `onedrive --display-config` 见新 `sync_dir` 无误；
- resync 之后，云端文件渐现于 `/run/media/<用户>/Data/OneDrive`；
- 盘未挂载时启动同步（或服务），当见因 `.nosync` 而中止之讯，数据无恙。

## 疑难

- **resync 乃重操作**：弃本地状态、依云端重建，官方戒勿日常用之；然改 `sync_dir`（或 sync_list 等）后即为必须。大库尤耗内存（约每十万对象 1 GiB）。运行前会示警告与 [Y/N]。
- **运行中勿删 `sync_dir` 内之物**：本地之删会被认作本意，而波及云端。
- **动盘之前先停同步**：`systemctl --user stop onedrive.service`（若用服务），或先了结前台进程，再行卸载、断电。
- **次序无忧**：fstab 之盘于开机早期即挂，先于用户服务（登入后启动）；守卫复为万一之备。
- **勿与 onedriver 混用同目录**：一者同步、一者按需挂载，混则必乱。

## 回退

```bash
systemctl --user disable --now onedrive.service     # 停常驻
# 将 config 之 sync_dir 改回（或去 check_nomount），再：
onedrive --sync --resync

# 撤守卫（须在盘未挂载时）：
sudo umount /run/media/<用户>/Data
rm /run/media/<用户>/Data/.nosync
rm /run/media/<用户>/Data/OneDrive/.nosync
sudo mount /run/media/<用户>/Data
```
