# Zed 綠色包之安置與桌面整合

## 環境

- 系統：Fedora 44
- 桌面：KDE Plasma 6.6.4（Wayland 會話）
- 組件：Zed 1.21.0 之 Linux 綠色包（官方 tar.gz）
- 撰寫：2026-09-27

## 適用

凡自官方 tar.gz 解壓 Zed，而未執行其 `install.sh` 者。他種發行版亦可依此法，惟 `PATH` 之設與桌面快取之刷新略異。

## 安置

官方 tarball 之默認去處為 `~/.local/zed.app`，其本體在 `libexec/zed-editor`，`bin/zed` 不過啟動之器。解之：

```bash
mkdir -p ~/.local
tar -xzf zed-linux-x86_64.tar.gz -C ~/.local
ls ~/.local/zed.app        # 望見 bin/ lib/ libexec/ share/
```

其未竟之三事，須補：

```bash
mkdir -p ~/.local/bin ~/.local/share/applications
ln -sf ~/.local/zed.app/bin/zed ~/.local/bin/zed

cp ~/.local/zed.app/share/applications/dev.zed.Zed.desktop \
   ~/.local/share/applications/

sed -i -e "s|^TryExec=zed|TryExec=$HOME/.local/zed.app/bin/zed|" \
       -e "s|^Exec=zed|Exec=$HOME/.local/zed.app/bin/zed|" \
       -e "s|^Icon=zed|Icon=$HOME/.local/zed.app/share/icons/hicolor/512x512/apps/zed.png|" \
       ~/.local/share/applications/dev.zed.Zed.desktop

update-desktop-database ~/.local/share/applications
```

所以須改 `Exec` 與 `Icon` 者：包內 desktop 檔原作 `Exec=zed`、`Icon=zed`，賴 `PATH` 與圖示主題尋之；改為絕對路徑，則圖形會話不倚 shell 之 `PATH` 亦可啟動。

`~/.local/bin` 須在 `PATH` 中。多數發行版之 `~/.bashrc` 已設；若無，補之：

```bash
export PATH="$HOME/.local/bin:$PATH"
```

## 資料與設定

- 資料：`~/.local/share/zed`
- 設定：`~/.config/zed/settings.json`

## 驗證

```bash
zed --version                 # 應現 1.21.0 與 .../libexec/zed-editor 之路徑
ls ~/.local/share/applications/dev.zed.Zed.desktop
```

圖形選單中當見 Zed；若未即現，登出再入以刷新快取。

## 疑難

- `command -v zed` 無所出：`~/.local/bin` 未入 `PATH`，或符號連結未建。
- 選單無項：desktop 檔未佈署，或 `update-desktop-database` 未行。
- 圖示空白：`Icon` 未改為絕對路徑，或未重登。

## 回退

```bash
rm -f ~/.local/bin/zed ~/.local/share/applications/dev.zed.Zed.desktop
rm -rf ~/.local/zed.app ~/.local/share/zed ~/.config/zed
```

## 附註

日後升級，逕以下載之新 tarball 覆蓋 `~/.local/zed.app` 即可；符號連結與 desktop 檔不必重作，惟圖示路徑如有變更則須再改。
