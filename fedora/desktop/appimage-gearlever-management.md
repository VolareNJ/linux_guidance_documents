# 以 Gear Lever 代管 AppImage（Flatpak 安装与 CLI 整合）

## 环境

- 系统：Fedora Linux 44（KDE Plasma Desktop Edition）
- 桌面：KDE Plasma（Wayland 会话）
- 内核：7.2.9-200.fc44.x86_64
- 相关硬件：不涉
- 前提：Flatpak 已装（1.18.4）、Flathub 仓库已配；受管之 AppImage 皆 type-2（辨型见 `fedora/desktop/appimage-desktop-entry.md` 之补注）
- 撰写：2026-10-11

## 适用

凡欲以图形工具统一代管 AppImage——整合菜单与图标、集中存放、查检更新、移除——皆可参此。Gear Lever 以 Flatpak 分发，故凡可装 Flatpak 之发行版皆宜。若不便装 Flatpak，或仅欲手工整合少数 AppImage，参 `fedora/desktop/appimage-desktop-entry.md` 之手工法。

同类工具亦有 AppImageLauncher，然其久未更新（末次发布在 2022 年），于新发行版兼容有虞；Gear Lever 则活跃维护，故取之。

## 步骤

### 一、安装

系统级（默认，需 root 权限）：

```bash
flatpak install flathub it.mijorus.gearlever
```

亦可加 `--user` 装于用户域。装毕，菜单即见「Gear Lever」。

### 二、整合既有 AppImage

其备命令行入口，可免开图形界面：

```bash
flatpak run it.mijorus.gearlever --integrate <AppImage 路径> --yes
```

`<AppImage 路径>` 为占位符，须易为实际路径（如 `~/Downloads/example.AppImage`），勿照抄字面；`--yes` 免交互之问，不加则逐一询名称、版本与更新源。同一命令可反复施于多个文件。

可设别名以求简便（其帮助文本亦如此建议）：

```bash
alias gearlever='flatpak run it.mijorus.gearlever'
```

常用子命令一览：

| 子命令 | 作用 |
|---|---|
| `--integrate <file> [--yes] [--replace]` | 整合入管（`--replace` 于同名冲突时以今者替旧者） |
| `--list-installed [--json]` | 列已代管者 |
| `--update <file>` / `--update --all -y` | 查检并更新 |
| `--remove <file> [--delete]` | 移除（默认入回收站，`--delete` 则直删） |
| `--remove-all` | 尽数移除 |

### 三、整合之实况（目录布局）

整合非仅生成桌面项，实为**迁移**：

- AppImage 本體迁往 `~/AppImages`（取自 GSettings 之 `appimages-default-folder`，默认即此；`move-appimage-on-integration` 默认为 true）。文件名改取包内桌面之 Name：小写、空格易下划线、去特殊字符，如 `comfyui.AppImage` → `comfy_desktop.appimage`；中文名则存中文，如 `哔哩下载姬.appimage`。
- 图标解出，藏于 `~/AppImages/.icons/`，桌面项之 `Icon` 书绝对路径指之。
- 桌面文书书于 `~/.local/share/applications/`，其 `TryExec`、`Exec` 皆指新径，且令 `Exec=env DESKTOPINTEGRATION=1 ...`。
- 其借 `flatpak-spawn --host` 于宿主代跑 `update-desktop-database`，故菜单随刷。

### 四、清理旧手写桌面项

若旧有手书之 desktop 文书（其 `Exec` 指 `~/.local/bin/` 等旧处），本體既迁，文书遂成空壳，点之无应。先察其内容：

```bash
cat ~/.local/share/applications/<旧文书>.desktop
```

若 `Exec` 尚指旧径而文件已迁者，删之；若恰同名而已被 Gear Lever 覆写为新径者，留之：

```bash
rm ~/.local/share/applications/<旧文书>.desktop
update-desktop-database ~/.local/share/applications
```

## 验证

```bash
flatpak run it.mijorus.gearlever --list-installed
ls ~/AppImages
```

前者列其名、版本、更新源与路径；后者见所迁之 AppImage 与 `.icons` 目录。

## 疑难

| 症状 | 成因 | 对策 |
|---|---|---|
| `update-desktop-database` 报 `invalid MIME type` | 包内桌面文书带 `MimeType=;` 空值，Gear Lever 原样抄录 | 删该空行，或忽略（无害） |
| `desktop-file-validate` 报 Exec 含 reserved character `'` | 包名含中文（非 ASCII）时，Gear Lever 以 `shlex` 引路径，被裹单引号 | 启动无碍（KDE、GNOME 之解析器皆认单引号）；欲严格合规，易为双引号 |
| CLI 偶报 `Error writing credentials to socket` | 与既有实例通讯之握手，CLI 早退故管断 | 无害，命令照常 |
| 菜单未即见新项 | KDE 之 sycoca 缓存未刷 | `kbuildsycoca6 --noincremental`，或注销重登 |
| 手写文书点之无应 | 本體已迁，`Exec` 空指 | 删之，统一由 Gear Lever 管 |
| 菜单中同名两枚 | 手写文书与 Gear Lever 所生者并存 | 删手写者 |

## 回退

```bash
flatpak run it.mijorus.gearlever --remove <file>            # 入回收站
flatpak run it.mijorus.gearlever --remove <file> --delete   # 直删
flatpak run it.mijorus.gearlever --remove-all
```

卸 Gear Lever 本體（已管之 AppImage 宜先移除）：

```bash
flatpak uninstall it.mijorus.gearlever
```

## 兼容性补注

- 整合后文件既迁，凡他处脚本或文书引用旧径者，皆须改指 `~/AppImages/`。
- 更新检测赖包内嵌更新信息或自定义更新源；无者，列表示 `UpdatesNotAvailable`，仍可手动重下新包，以 `--integrate --replace` 替之。
- Gear Lever 之 Flatpak 沙箱申请 `host` 全文件系统访问与 `org.freedesktop.Flatpak` 通讯权，方能代跑宿主命令、书文书于家目录——故其能管家目录之整合，非仅沙箱之内。
