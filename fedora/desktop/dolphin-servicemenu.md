# Dolphin 自訂服務選單（右鍵以指定程式開啟）

## 環境

- 系統：Fedora 44（KDE Plasma Desktop Edition）
- 桌面：KDE Plasma 6.7.5（Wayland 會話）；Dolphin 26.08.1
- 內核：7.2.7-200.fc44.x86_64
- 組件：KDE Frameworks 6（KService/KIO）；所開之程式以 Zed 1.21.0（裝於 `~/.local`）為例
- 前提：程式已裝，且知其執行檔之絕對路徑；`kbuildsycoca6` 可用
- 撰寫：2026-09-29

## 適用

凡欲於 Dolphin 右鍵選單為檔案與資料夾增「以某程式開啟」之項者。KIO 系檔案管理器（Konqueror 等）同理；他應用惟易 `Exec`、`Name`、`Icon` 而已。

於 GNOME/Nautilus 不用此法。程式之本體安置（如 Zed）見 `zed-manual-install.md`，本文惟論選單之整合。

## 步驟

### 一、建服務選單檔

置於 `~/.local/share/kio/servicemenus/`（個人，宜）或 `/usr/share/kio/servicemenus/`（系統，屬 root）：

```bash
mkdir -p ~/.local/share/kio/servicemenus
$EDITOR ~/.local/share/kio/servicemenus/zed-open.desktop
```

其文如左。`<user>` 換作實際用戶名（可 `echo $HOME` 得之）——desktop 檔不展開 `~` 與環境變數，故 `Exec`、`Icon` 須書絕對路徑：

```ini
[Desktop Entry]
Type=Service
ServiceTypes=KonqPopupMenu/Plugin
MimeType=all/allfiles;inode/directory;
Actions=openWithZed;
X-KDE-Priority=TopLevel

[Desktop Action openWithZed]
Name=Open with Zed
Name[zh_CN]=用 Zed 打开
Icon=/home/<user>/.local/zed.app/share/icons/hicolor/512x512/apps/zed.png
Exec=/home/<user>/.local/zed.app/bin/zed %F
```

逐句之義：

- `MimeType=all/allfiles;inode/directory;`——`all/allfiles` 為諸檔案，`inode/directory` 為資料夾；並列之則兩者皆見。
- `X-KDE-Priority=TopLevel`——列於右鍵主選單，不藏於子選單。
- `%F`——所選項目之路徑串列，可多選（`%f` 祇取其一）。
- `Exec` 用絕對路徑——圖形會話之 `PATH` 未必含 `~/.local/bin`（`~/.bashrc` 之設不入圖形會話），不可賴之。

### 二、設可執行位（緊要）

```bash
chmod +x ~/.local/share/kio/servicemenus/zed-open.desktop
```

說理：KDE 於讀用服務選單檔時有安全檢查——**非 root 所屬者，須有可執行位**，否則 Dolphin 拒之，報「您沒有此文件的執行權限」。系統目錄之選單屬 root，不受此限；個人目錄所建者必設之。未設 x 者，或見項於選單，而點之即敗，並於日誌留「not owned by root and executable flag not set」之語。

### 三、刷新快取

```bash
kbuildsycoca6 --noincremental
```

Dolphin 通常自察其變；若不現，重啟 Dolphin（`kquitapp6 dolphin`）或登出再入。

## 驗證

右鍵任一檔案或資料夾，當見「用 Zed 打开」並攜其圖示；點之即開。

若疑不成，察日誌：

```bash
journalctl --user -b --no-pager | grep -i servicemenus
```

見 `denied, not owned by root and executable flag not set` 者，即 +x 未設之故。

## 疑難

| 症狀 | 成因 | 對策 |
|---|---|---|
| 「您沒有此文件的執行權限」 | 選單檔無 x 位（KDE 安全檢查） | `chmod +x` 該檔，而後刷新快取 |
| 選單中無此項 | 快取未刷新；或 `MimeType` 未中對象 | `kbuildsycoca6 --noincremental`；驗 `MimeType` 含 `all/allfiles` 與 `inode/directory` |
| 點之而無反應 | `Exec` 用相對名，而圖形會話之 `PATH` 無之 | `Exec` 改絕對路徑 |
| 圖示空白 | `Icon` 之名不在圖示主題中 | `Icon` 改絕對路徑（png 之路徑） |

## 回退

```bash
rm ~/.local/share/kio/servicemenus/zed-open.desktop
kbuildsycoca6 --noincremental
```

## 兼容性補註

- 此文所據為 Plasma 6.7.5 與 Dolphin 26.08.1。「非 root 須 +x」之檢查見於較新之 KDE Frameworks 6；舊版或寬鬆，然設之無害。
- 服務選單之徑 KDE Frameworks 5、6 皆同；Plasma 5 之刷新器為 `kbuildsycoca5`。
- 桌面整合之他法：於 `dev.zed.Zed.desktop` 之 `MimeType` 添 `inode/directory`，可令 Zed 現於「開啟方式」子選單；然官方註解有警告，或致 Zed 反成默認檔案管理器，故未採。
