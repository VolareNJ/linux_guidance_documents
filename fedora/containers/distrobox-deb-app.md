# 以 Distrobox 隔離安裝 Debian 系 deb 應用

## 環境

- 宿主：Fedora 44；KDE Plasma 6.6.4（Wayland 會話；`DISPLAY=:0` 經 XWayland）
- 組件：podman 5.8.1、distrobox 1.8.2.5、toolbox 0.3、flatpak 1.17.6
- 容器：名稱 `appbox`，基底 `docker.io/library/ubuntu:26.04`；與宿主共享家目錄，故 `~/下载` 於兩邊同路
- 實例：某統信（UOS）系商業應用之 deb，自帶 CEF 內核與大量私有庫，置於 `~/下载/example-app_1.2.3-1_amd64.deb`
- 撰寫：2026-09-27

## 適用

宿主為 rpm 系，而應用只得 deb 者；或其依賴含宿主所無之專有件；或其自帶大量庫，恐與系統相擾。單應用獨居與多應用共容器之取捨，見末節。

## 三法之較

| 法 | 益 | 損 |
|---|---|---|
| 宿主解包直跑 | 至省 | 無套件管理，依賴手補；卸載靠刪目錄 |
| Flatpak | 沙箱規範，權限可審 | 須現成打包；僅發 deb 之私有軟體不現實 |
| Distrobox + rootless Podman | 得發行版原生 dpkg／apt，與宿主整合佳 | 有空間與維護之費；顯示與縮放須理 |

本文用第三者。

## 準備

```bash
sudo dnf install -y podman distrobox
podman --version
distrobox version
```

rootless 之 podman 無常駐守護，容器以凡用戶身分運行，不必 `sudo`。

基底映像之擇：deb 屬 Debian 系，映像須 Ubuntu 或 Debian；其 glibc 不得顯著舊於 deb 之構建環境。老 deb（如 UOS 20 之流）可自 `ubuntu:22.04` 或 `debian:12` 起；新者用 `ubuntu:24.04` 或更新。先觀其 control：

```bash
dpkg-deb -I ~/下载/example-app_1.2.3-1_amd64.deb
```

## 建容器

```bash
distrobox create --name appbox --image docker.io/library/ubuntu:24.04
distrobox enter appbox
```

容器與宿主共享家目錄，故 deb 無須另掛。

## 容器內安裝

### 辨路徑

deb 之「外殼」或為目錄（由 zip 解出），真包藏於其內：

```bash
ls -l ~/下载/example-app/
file ~/下载/example-app/example-app_1.2.3-1_amd64.deb
```

若誤指目錄，dpkg 報 `archive ... is not a regular file`。

### 專有依賴之處置

此類包或倚統信之簽名校驗器 `deepin-elf-verify`，Debian 系之常倉亦無，運行實不需。宜造空殼包以滿足 apt（即 Debian `equivs` 之意）：

```bash
mkdir -p /tmp/dummy/DEBIAN
cat > /tmp/dummy/DEBIAN/control <<'EOF'
Package: deepin-elf-verify
Version: 1.1.10-1
Section: utils
Priority: optional
Architecture: all
Maintainer: local placeholder <placeholder@localhost>
Description: Dummy placeholder for deepin-elf-verify
EOF
dpkg-deb --build --root-owner-group /tmp/dummy /tmp/deepin-elf-verify_1.1.10-1_all.deb
sudo dpkg -i /tmp/deepin-elf-verify_1.1.10-1_all.deb
```

依賴名隨包而異，其法可類推。

### 安裝本體

```bash
sudo dpkg -i --force-depends ~/下载/example-app_1.2.3-1_amd64.deb
sudo apt --fix-broken install -y
```

次序要緊：先空殼，後本體；不爾，apt 見依賴斷而拒行全部。

### 補桌面庫

此類應用（尤以 CEF 為界面者）倚若干系統庫，素器 Ubuntu 多缺：

```bash
sudo apt update
sudo apt install -y libnss3 libnspr4 libgtk-3-0t64 libgbm1 libasound2t64 \
  libxkbcommon0 libxcomposite1 libxdamage1 libxfixes3 libxrandr2 libxtst6 \
  libatspi2.0-0t64 libcups2t64 libdrm2 libpango-1.0-0 libcairo2 \
  libatk1.0-0t64 libatk-bridge2.0-0t64 libgl1 libegl1 fonts-noto-cjk fontconfig
```

按：Ubuntu 24.04 以後之包名多帶 `t64` 綴；`libasound2` 為虛擬包，須指 `libasound2t64`。名若不符，apt 之提示自示正名。

## 驗證與除錯

### 迭代察缺庫

CEF 一遭只報首缺之庫，故須迭代：

```bash
cd /opt/apps/com.example.app/files
LD_LIBRARY_PATH=../lib:../lib/3rd ldd bin/mainapp | grep 'not found'
LD_LIBRARY_PATH=../lib:../lib/3rd ldd wbrowser/libcef.so | grep 'not found'
```

二者俱空，方可有成。每見 `not found`，即以 apt 補之。

### 直跑以觀其報

```bash
sh /opt/apps/com.example.app/files/bin/launcher.sh
```

「任務欄一閃即沒」者，即此腳本立退之故；其報多為缺庫，如 `libnss3.so: cannot open shared object file`。

### dpkg 之狀態

```bash
dpkg -l | grep -i com.example.app
```

`ii` 為裝妥；`rc` 為已卸而餘設定，宜 `sudo dpkg --purge com.example.app` 清之。

### sandbox

CEF 若報 sandbox，修其屬主與 setuid 位：

```bash
ls -l /opt/apps/com.example.app/files/wbrowser/chrome-sandbox
sudo chown root:root /opt/apps/com.example.app/files/wbrowser/chrome-sandbox
sudo chmod 4755 /opt/apps/com.example.app/files/wbrowser/chrome-sandbox
```

## 桌面整合

### 匯出（於容器內）

```bash
distrobox-export --app com.example.app
# 欲去「(on appbox)」之綴，可加：
distrobox-export --app com.example.app --export-label none
```

注意：distrobox 1.8 之 `distrobox-export` 無 `--container` 旗標，於宿主行之亦不合；當於容器內行之。若報 `cannot find any desktop files`，即該應用未裝成。

### 手寫 desktop 檔（可控）

於宿主佈之：

```bash
mkdir -p ~/.local/share/icons/hicolor/scalable/apps
# 圖標可自 deb 內取：opt/apps/com.example.app/entries/icons/hicolor/256x256/apps/com.example.app.svg
cp /path/to/com.example.app.svg ~/.local/share/icons/hicolor/scalable/apps/
```

`~/.local/share/applications/com.example.app.desktop`：

```ini
[Desktop Entry]
Type=Application
Name=範例應用
Exec=/usr/bin/distrobox-enter -n appbox -- /opt/apps/com.example.app/files/bin/launcher.sh
Icon=com.example.app
Terminal=false
Categories=Utility;
StartupNotify=true
```

而後：

```bash
update-desktop-database ~/.local/share/applications
desktop-file-validate ~/.local/share/applications/com.example.app.desktop
```

## 縮放

容器內之應用不繼承宿主之縮放，故字或過小。諸法如次：

```bash
export GDK_SCALE=1
export GDK_DPI_SCALE=1.25
export QT_SCALE_FACTOR=1.25
sh /opt/apps/com.example.app/files/bin/launcher.sh
```

或逕傳 CEF 之開關：

```bash
cd /opt/apps/com.example.app/files/bin
./mainapp --force-device-scale-factor=1.25
```

若其走 Wayland 而縮放不施，可逼走 X11：

```bash
unset WAYLAND_DISPLAY
```

Xft 之一路：

```bash
xrdb -query | grep -i dpi
echo 'Xft.dpi: 120' | xrdb -merge
```

KDE Wayland 之宿主，可於 `~/.config/kwinrc` 設：

```ini
[Xwayland]
Scale=1.25
```

設後須登出再入，X11 之應用方隨系統縮放。

持久之法：改宿主 desktop 檔之 `Exec` 為：

```ini
Exec=/usr/bin/distrobox-enter -n appbox -- bash -lc "unset WAYLAND_DISPLAY; export GDK_SCALE=1; export GDK_DPI_SCALE=1.25; exec /opt/apps/com.example.app/files/bin/launcher.sh"
```

## 維護與清理

```bash
distrobox list
distrobox enter appbox -- sudo apt update
distrobox rm appbox                # 除容器與其所裝
podman system df                   # 觀空間
podman system prune                # 清無用之層
du -sh ~/.local/share/containers/storage
```

## 一容器一應用，抑或共之

- 映像層跨容器共享；各容器之可寫層（其所裝之物）不共享。
- 同域相合：同發行版、依賴相容、互信之應用，可共一容器（省空間，更新一次）。
- 異域相分：需不同發行版者分；依賴有衝突者分。
- 怪者獨居：裝卸腳本不守矩、自帶整套 runtime、依賴奇特者（本實例即是），宜獨處；厭之則一刪而淨。
- 巨者獨居：各帶大型 runtime（如各帶 CEF 者）者，共之亦不共享，反生混亂。
- 口訣：同域相合，異域相分；怪者獨居，巨者獨居。

## 疑難對照

| 症 | 因 | 治 |
|---|---|---|
| `archive ... is not a regular file` | 指了目錄而非 deb 檔 | 補全至真正之 `.deb` |
| `Package 'libasound2' has no installation candidate` | 虛擬包須指名 | 用 `libasound2t64` |
| `Unsatisfied dependencies: ... deepin-elf-verify` | 專有依賴缺 | 造空殼包（見上） |
| `cannot find any desktop files` | 應用未裝成 | 重裝並驗 `dpkg -l` |
| 一閃即沒 | 缺庫 | 以 ldd 迭代補之 |
| `libnss3.so: cannot open shared object file` | 同上 | `apt install libnss3` |

## 附錄：常用命令

```bash
distrobox create --name NAME --image IMAGE
distrobox enter NAME
distrobox list
distrobox rm NAME
distrobox-export --app APP [--export-label none]
podman system df
```

## 兼容性補註

本機為 Fedora 44 之記錄。他版之別：儲存與網路之設或有微異；舊於 Ubuntu 24.04 之映像無 `t64` 綴（作 `libasound2`、`libgtk-3-0` 等）。策略不變。
