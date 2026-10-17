# 以 pnpm 于用户目录安装 Node CLI 并立桌面项

## 环境

- 系统：Fedora Linux 44（KDE Plasma Desktop Edition）
- 桌面：KDE Plasma
- 内核：7.2.10-200.fc44.x86_64
- 相关硬件：无特殊要求
- 前提：Node.js 22 与 npm 10 已装；以普通用户操作，无需 root。本文以 DeepSeek Harness（`dsh`）为例，实测版本为 pnpm 12.14.0、dsh 0.2.0-rc.2
- 撰写：2026-10-17

## 适用

凡 Node CLI 工具宜装于用户目录、不愿污染系统 `/usr`，且需应用菜单或桌面入口者，可用此法。此法亦免 root、便于单用户升级与卸载。

不宜用于多用户共用、需系统服务常驻、或必须自源码构建之工程。后者宜另置源码目录（如 `~/src` 或 `/opt`），再以 `pnpm install && pnpm run build` 构建，不可与此文混为一谈。

## 步骤

### 一、装 pnpm 于用户目录

npm 之 `--prefix` 可改全局安装之根，使包入 `~/.local/lib/node_modules`，命令入 `~/.local/bin`：

```bash
npm install -g pnpm --prefix "$HOME/.local" --cache /tmp/npm-cache
```

若 `~/.local/bin` 不在 `PATH`，须补之。Fedora KDE 之登录环境通常已含此目录；若不确，可在 shell 配置中加：

```bash
export PATH="$HOME/.local/bin:$PATH"
```

### 二、定 pnpm 全局目录与环境

pnpm 10 以后以 `PNPM_HOME` 定全局 bin 之根，全局包置于 `$PNPM_HOME/global`。pnpm 12 已不接受 `--global-bin-dir` 参数，故以环境变量为准：

```bash
export PNPM_HOME="$HOME/.local/share/pnpm"
export PATH="$PNPM_HOME/bin:$HOME/.local/bin:$PATH"
mkdir -p "$PNPM_HOME"
```

若在沙箱、容器或 systemd 严格环境中运行，`XDG_RUNTIME_DIR` 常指向不可写之 `/run/user/<uid>`；pnpm 12 之 store 操作锁置于其下，会报 `Read-only file system`。此时改指用户可写目录：

```bash
export XDG_RUNTIME_DIR="$HOME/.local/run"
export TMPDIR="$HOME/.local/tmp"
mkdir -p "$XDG_RUNTIME_DIR" "$TMPDIR"
chmod 700 "$XDG_RUNTIME_DIR"
```

普通桌面会话无须此段。

### 三、全局安装 CLI

```bash
pnpm add -g --store-dir "$PNPM_HOME/store" @deepseek-ai/dsh
```

`--store-dir` 宜显式给出。pnpm 12 在无项目上下文时，可能将 store 置于当前目录之 `node_modules/.pnpm-store`；固定于 `PNPM_HOME` 下，可免污染工作目录。

pnpm 10 以后默认忽略依赖之 build scripts。若包已带预编译二进制（如 `node-pty` 之 `prebuilds/linux-x64/pty.node`、`koffi` 之 `@koromix/koffi-linux-x64/linux_x64/koffi.node`），通常无碍。必要时可查其包内 `postinstall`，手动执行其脚本，或以 `pnpm approve-builds -g` 择而允之，再 `pnpm rebuild -g`。

### 四、初运行以生配置

```bash
dsh web --no-open --port 3180
```

`dsh` 默认以 `~/.dsh` 为配置根；首次运行会生成 `~/.dsh/profiles/web/`。确认其能监听后以 `Ctrl-C` 止之。

### 五、包装脚本、图标与桌面项

桌面会话之 `PATH` 未必含 `$PNPM_HOME/bin`，故以包装脚本固定环境：

```bash
cat > "$HOME/.local/bin/dsh-web" <<'EOF'
#!/bin/sh
export PATH="$HOME/.local/bin:$HOME/.local/share/pnpm/bin:/usr/local/bin:/usr/bin:/bin"
exec "$HOME/.local/bin/dsh" web
EOF
chmod 755 "$HOME/.local/bin/dsh-web"
```

图标可取自包内前端资源：

```bash
mkdir -p "$HOME/.local/share/icons/hicolor/scalable/apps"
icon=$(find "$PNPM_HOME/global" -path '*dsh-web-frontend*' -name 'favicon.svg' -print -quit)
install -m 644 "$icon" "$HOME/.local/share/icons/hicolor/scalable/apps/dsh.svg"
```

桌面项之 `Exec` 须用绝对路径；`~` 于 `.desktop` 中不展开，故以 shell 展开 `$HOME` 生成：

```bash
mkdir -p "$HOME/.local/share/applications"
cat > "$HOME/.local/share/applications/deepseek-harness.desktop" <<EOF
[Desktop Entry]
Type=Application
Version=1.0
Name=DeepSeek Harness
Comment=启动 DeepSeek Harness 网页界面
Exec=$HOME/.local/bin/dsh-web
Icon=dsh
Terminal=true
Categories=Development;
Keywords=AI;Agent;DeepSeek;Harness;dsh;
EOF
```

`Terminal=true` 因 `dsh web` 为前台服务，须留终端以观日志；关闭终端即停服务。若欲开机常驻，宜另作 systemd 用户服务。

刷新桌面数据库与图标缓存：

```bash
update-desktop-database "$HOME/.local/share/applications" 2>/dev/null || true
gtk-update-icon-cache -f -t "$HOME/.local/share/icons/hicolor" 2>/dev/null || true
```

### 六、桌面副本

```bash
install -m 755 "$HOME/.local/share/applications/deepseek-harness.desktop" "$HOME/桌面/deepseek-harness.desktop"
```

KDE Plasma 首次双击桌面项，或提示「不受信任」。右键选「允许运行」或「信任并启动」即可；应用菜单中之项不受此限。

## 验证

```bash
dsh --version
timeout 30 dsh web --no-open --port 3180
ss -ltn | grep 3180
```

前者应输出版本号；后者应见 `LISTEN ... 127.0.0.1:3180`，日志给出带 token 之 URL。桌面项点击后应自动开浏览器至该 URL。

## 疑难

| 症状 | 成因 | 对策 |
|---|---|---|
| `ERR_PNPM_GLOBAL_BIN_DIR_NOT_IN_PATH` | pnpm 要求全局 bin 目录在 `PATH` 中 | `export PATH="$PNPM_HOME/bin:$PATH"` |
| `ERR_PNPM_STORE_DIR_OPEN_OPERATION_LOCK`，附 `Read-only file system` | 锁欲置于不可写之 `XDG_RUNTIME_DIR` 或 `TMPDIR` | 改指 `~/.local/run`、`~/.local/tmp`，或于可写环境中运行 |
| `unexpected argument '--global-bin-dir'` | pnpm 12 已移除该参数 | 改用 `PNPM_HOME` 环境变量 |
| `Ignored build scripts: ...` | pnpm 10 以后默认禁 postinstall | 先查预编译二进制；必要则 `pnpm approve-builds -g` 或手动执行脚本 |
| 命令找不到 | `~/.local/bin` 或 `$PNPM_HOME/bin` 不在 `PATH` | 链入 `~/.local/bin`，或补 `PATH`；桌面项用绝对路径 |
| 桌面项点击无反应 | `Exec` 用了 `~`、缺可执行位、或 KDE 未信任 | 用绝对路径，`chmod 755`，右键信任 |
| 首次启动报 `mkdir ... ~/.dsh/profiles/...` 失败 | 配置目录不可写或父目录未建 | 建 `~/.dsh` 并确保属主为当前用户 |

## 回退

```bash
pnpm remove -g @deepseek-ai/dsh
rm -f "$HOME/.local/bin/dsh" "$HOME/.local/bin/dsh-web"
rm -f "$HOME/.local/share/applications/deepseek-harness.desktop" "$HOME/桌面/deepseek-harness.desktop"
rm -f "$HOME/.local/share/icons/hicolor/scalable/apps/dsh.svg"
rm -rf "$HOME/.dsh"
```

若不再需 pnpm，可 `npm uninstall -g pnpm --prefix "$HOME/.local"`，并删 `$PNPM_HOME`。

## 兼容性补注

- XDG 布局：命令入 `~/.local/bin`，数据入 `~/.local/share`，配置入 `~/.config` 或工具自定目录（dsh 用 `~/.dsh`）。系统级安装则宜 `/opt/<name>` 配合 `/usr/local/bin` 之链接，或以发行版包管理器为之。
- pnpm 10 至 12 之全局目录语义与 build scripts 策略有变；他版本须以 `pnpm add --help` 与 `pnpm config list` 复核。
- 本文之沙箱处置（`XDG_RUNTIME_DIR`）属宿主限制所致，普通桌面无须。
- 安装体积随依赖而异；dsh 之全局依赖约 538 MB，宜预留空间。
