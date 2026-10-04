# AppImage 之桌面項集成與辨誤（LocalSend 為例）

## 環境

- 系統：Fedora Linux 44（KDE Plasma Desktop Edition）
- 桌面：KDE Plasma（`XDG_CURRENT_DESKTOP=KDE`）
- 內核：7.2.7-200.fc44.x86_64
- 相關硬體：不涉
- 前提：運行 AppImage 須 FUSE 與 `/dev/fuse`；本機 fuse3 已裝
- 撰寫：2026-10-04

## 適用

凡以 AppImage 分發之程序，欲入桌面選單而啟動；或軟連結「打不開」；或啟動器報「沒有 Exec 字段」者，皆可參此。所涉軟連結、inode 與 desktop 入口之規範，跨發行版通用；FUSE 及他項異同見「兼容性補註」。

## 步驟

### 一、先辨軟連結：勿令自指

符號連結（soft link）有「連結名」與「標的」二者，其式為 `ln -s <標的> <連結名>`。若誤書為 `ln -s localsend localsend`（名與標的同），或於相對標的處誤解其基準（相對標的乃相對於**連結所在目錄**，非當前目錄），即得自環：

```text
~/.local/share/localsend -> localsend
```

此連結解析必敗，狀若「打不開」。辨之：

```bash
readlink ~/.local/share/localsend        # 示其所指
readlink -f ~/.local/share/localsend     # 正解者示其最終實體；循環者無輸出而非零退出
file ~/.local/share/localsend            # 應為 symbolic link to <實體>；自環者云 broken
```

正解為以絕對路徑重建：

```bash
ln -sfn ~/.local/bin/localsend ~/.local/share/localsend
```

然須思其位：可執行檔之連結宜置 `~/.local/bin`（在 PATH 中者），`~/.local/share` 本非其地也。

### 二、再辨 desktop 文件：必須為文本

`~/.local/share/applications/*.desktop` 乃桌面入口文件，其為**純文本**，別於程序本體。若誤以 `ln`（無 `-s`，成硬連結）或 `cp` 將 AppImage 置於此處，則解析器逐行為之，滿紙密文，無 `[Desktop Entry]` 組頭、無 `Exec` 鍵可讀，啟動器遂報「沒有 Exec 字段」。辨之：

```bash
file ~/.local/share/applications/localsend.desktop
# 云 "ELF ... executable" 者，即誤置二進位；
# 云 "text" 者，方為 desktop 文件。
ls -l ~/.local/share/applications/localsend.desktop
# 觀第二列（連結數）；大於 1 者或為硬連結，可與本體對 inode 而辨：
stat --printf='inode=%i links=%h %n\n' \
  ~/.local/share/applications/localsend.desktop ~/.local/bin/localsend
```

正解為另書文本（`<user>` 為您的用戶名；`Exec` 為必需鍵，須書絕對路徑）：

```bash
rm ~/.local/share/applications/localsend.desktop   # 刪去誤置者，無損本體
cat > ~/.local/share/applications/localsend.desktop <<'EOF'
[Desktop Entry]
Type=Application
Name=LocalSend
Comment=Share files with nearby devices
Exec=/home/<user>/.local/bin/localsend
Icon=localsend
Terminal=false
Categories=Network;FileTransfer;
EOF
chmod 644 ~/.local/share/applications/localsend.desktop
```

`Icon` 之名於系統無對應圖示者，至多示預設圖，不妨啟動。

### 三、驗證與刷新

```bash
desktop-file-validate ~/.local/share/applications/localsend.desktop
update-desktop-database ~/.local/share/applications
```

前者無輸出即通過；後者刷新資料庫，令選單即見。

### 四、圖示之補

`Icon=` 所書之名（或絕對路徑），須確有對應之圖示，否則選單示預設圖。先辨之：

```bash
find ~/.local/share/icons /usr/share/icons -iname '*localsend*' 2>/dev/null
```

AppImage 內多自帶圖示，可解出而裝之（圖示名須與 `Icon=` 相符）：

```bash
mkdir -p /tmp/ls-icon && cd /tmp/ls-icon
~/.local/bin/localsend --appimage-extract 'usr/share/icons/*'
for s in 32 128 256; do
  png="squashfs-root/usr/share/icons/hicolor/${s}x${s}/apps/localsend.png"
  xdg-icon-resource install --novendor --size "$s" "$png" localsend
done
kbuildsycoca6
```

尺寸以 AppImage 內實有者為準；`~/.local/share/icons/hicolor/` 之骨幹 `xdg-icon-resource` 自會補之，`index.theme` 無需自備（系統者可疊加）。GTK 系程序可另以 `gtk-update-icon-cache -f -t ~/.local/share/icons/hicolor` 刷之。

若選單仍示預設圖（主題查找不效），則改用絕對路徑之法——`Icon=` 逕書圖示檔之絕對路徑，不賴主題查找，最為直截：

```bash
install -Dm644 ~/.local/share/icons/hicolor/256x256/apps/localsend.png ~/.local/share/icons/localsend.png
# desktop 文件：Icon=/home/<user>/.local/share/icons/localsend.png
kbuildsycoca6
```

## 驗證

- `desktop-file-validate` 無輸出；
- `readlink -f` 對所建連結皆能示出實體；
- 應用選單中見 LocalSend 及其圖示，點之可啟。

## 疑難

| 症狀 | 成因 | 對策 |
|---|---|---|
| 軟連結打不開，`cat` 云「Too many levels of symbolic links」 | 連結自環：`ln -s` 名、標的同，或相對路徑基準誤解 | `readlink -f` 察之；以 `ln -sfn <絕對標的> <連結名>` 重建 |
| 啟動器云「沒有 Exec 字段」等 | `.desktop` 非文本，實為二進位之硬連結或副本 | `file` 辨之；刪之，另書文本 desktop 文件 |
| 運行云 `fuse: device not found` 或 `Cannot mount AppImage` | FUSE 未備：未裝 fuse3、未載 fuse 模組，或 `/dev/fuse` 缺失 | `sudo modprobe fuse`；裝 fuse3；或以 `--appimage-extract-and-run` 運行 |
| 選單中不見其項 | 資料庫未刷新，或會話未重入 | `update-desktop-database`；註銷重登 |
| 選單中無圖示，或示預設圖 | `Icon=` 之名於圖示主題查無對應，或主題查找不效 | 自 AppImage 解出圖示，以 `xdg-icon-resource` 裝之；不效則改用絕對路徑（詳步驟四），末以 `kbuildsycoca6` 刷之 |

## 回退

```bash
rm ~/.local/share/applications/localsend.desktop
rm ~/.local/share/localsend   # 若曾建此軟連結
update-desktop-database ~/.local/share/applications
```

若曾裝圖示，並刪之：

```bash
rm ~/.local/share/icons/hicolor/{32x32,128x128,256x256}/apps/localsend.png
rm ~/.local/share/icons/localsend.png
```

若所刪為硬連結（連結數大於 1），刪之不動本體；本體自在 `~/.local/bin/localsend`。

## 兼容性補註

- desktop entry 之規範（freedesktop.org）各發行版同；報錯文字因啟動器而異，其理一也。
- FUSE 套件名：Fedora 與 Debian 系皆 `fuse3`；上古老 AppImage 或須 `fuse`（fuse2）。
- 辨 AppImage type 2：`od -An -tx1 -N 12 <file>`，於偏移 8–10 位元組見 `41 49 02`（即 `AI\x02`）。
- 容器、沙箱中所見之 FUSE 報錯（`/dev/fuse` 不可見）未必反映桌面實況，須於桌面會話復驗。
