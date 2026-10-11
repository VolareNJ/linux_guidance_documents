# onedriver：以 FUSE 按需访问 OneDrive（类 Windows「文件按需」）

## 环境

- 系统：Fedora 44
- 桌面：KDE Plasma（Wayland 会话）
- 内核：7.2.9-200.fc44.x86_64
- 相关软体：onedriver 0.15.0（COPR `jstaf/onedriver`）；fuse3、webkit2gtk 等依包自带
- 前提：挂载点所在之路径当恒存（荐置于 fstab 自动挂载之分区，见 `fedora/storage/partition-automount-run-media.md`）
- 撰写：2026-10-11

## 适用

- 可用：欲如 Windows「Files On-Demand」——目录树与元数据在，**文件内容用时方下载**。onedriver 乃 FUSE 文件系统，非同步客户端；曾开之文件离线可读，缓存存于 `~/.cache/onedriver/`。
- 不可用：数十 GB 之大文件（文件全载入内存）；需符号链接者（Microsoft 不供）；与 abraunegg/onedrive 同步客户端**不可共用同一目录**。

## 步骤

### 一、安装（Fedora，COPR）

```bash
sudo dnf copr enable jstaf/onedriver
sudo dnf install onedriver
```

若 COPR 尚无当前 Fedora 版本之构建，须依其 README 自源码编译（依赖：`golang gcc pkg-config webkit2gtk4.1-devel json-glib-devel`）。

### 二、命令行认证（要点，避一坑，见「疑难」）

```bash
onedriver --auth-only --no-browser /run/media/<用户>/<UUID>/OneDrive
```

依其提示：浏览器开所给 URL → 登入并允之 → 授权毕得空白页，**复制其地址栏之 URL** → 黏回终端回车。成则 auth tokens 落于 `~/.cache/onedriver/<转义之挂载点>/auth_tokens.json`。

（若终端设有 http(s)_proxy 之类环境变量而网络直连，认证前可 `unset` 之，免其作梗。）

### 三、试挂载（前台，观其效；Ctrl-C 即卸）

```bash
onedriver /run/media/<用户>/<UUID>/OneDrive
```

### 四、立 systemd 用户服务，登入即自挂

```bash
systemctl --user enable --now "onedriver@$(systemd-escape --path /run/media/<用户>/<UUID>/OneDrive).service"
```

观其日志：

```bash
journalctl --user -u 'onedriver@*' --since today
```

## 验证

```bash
findmnt -t fuse.onedriver
```

当见挂载点与 `fuse.onedriver` 之类型。入目录 `ls`，所览者元数据也；`cat` 或开某文件，其内容方始下载。

## 疑难

- **首启认证弹窗崩溃**（KDE Wayland 下常见）：日志见 `Error 71 (Protocol error) dispatching to Wayland display`，其前有 kwin 之 `error in client communication`。此乃内置 WebKitGTK 与 Wayland 之隙，与挂载点、账号皆无涉。**对策**：如上二节以 `--no-browser` 于终端认证，根本不启窗口。若日后启动仍遇同类崩溃，可为服务加 X11 回退：

    ```bash
    mkdir -p ~/.config/systemd/user/onedriver@.service.d
    printf '[Service]\nEnvironment=GDK_BACKEND=x11\n' > ~/.config/systemd/user/onedriver@.service.d/x11.conf
    systemctl --user daemon-reload
    ```

    （令 WebKit 走 XWayland，功能不减。）
- **全量下载之触发**：文件管理器作缩略图、备份/索引工具、`find`、`rsync`、`grep -r` 遍历目录者，皆会诱发目录内文件之下载——按需之代价。慎以大工具遍历挂载点。
- **挂载点须恒存**：onedriver 服务于登入后启动，其时 fstab 之盘早已挂好，无碍；然若挂载点缺（盘未挂），服务即败。故先固定盘之挂载。
- 服务单元模板为 `/usr/lib/systemd/user/onedriver@.service`，其 `ExecStopPost` 自带 `fusermount3 -uz` 以清理。

## 回退

```bash
systemctl --user disable --now "onedriver@$(systemd-escape --path <挂载点>).service"
fusermount3 -uz <挂载点>
onedriver -w          # 清空该挂载点之全部本地缓存与 auth tokens
sudo dnf remove onedriver
```
