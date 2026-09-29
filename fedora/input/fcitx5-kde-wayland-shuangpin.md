# fcitx5 於 KDE Wayland 之配置與小鶴雙拼

## 環境

- 系統：Fedora 44
- 桌面：KDE Plasma 6.7.5（Wayland 會話）
- 組件：fcitx5 5.1.22、kwin 6.7.5、imsettings 1.8.11、fcitx5-chinese-addons（5.1 系列）
- 撰寫：2026-09-29

## 適用

凡 KDE Plasma Wayland 會話中用 fcitx5 而欲輸入中文（全拼或雙拼，含小鶴）者。X11 會話與 GNOME 之理各異，不可照搬。他發行版之 KDE 同理，惟包名不同；無 imsettings 者，環境變數當自設。

## 其理

裝 `fcitx5` 而中文不出者，其因有三，須並治之：

1. `fcitx5` 僅為框架，中文引擎在 `fcitx5-chinese-addons`（提供「拼音」與「雙拼」兩輸入法）。未裝則無可切之中文；所見「中文」唯鍵盤佈局（如 cn-altgr-pinyin），故只得拉丁字母。此最易誤認。
2. KDE Wayland 之輸入法樞機在 kwin：`kwinrc` 之 `[Wayland] InputMethod` 記輸入法之 desktop 文件名（`org.fcitx.Fcitx5.desktop`），kwin 據以啟動 fcitx5，並使之為 Wayland 原生輸入法後端（zwp_input_method 協議）。不設，則 Wayland 原生程式無輸入法可用。
3. 環境變數由 imsettings 於會話啟動時統籌（`/etc/xdg/plasma-workspace/env/xinput.sh`）：Wayland 下 `GTK_IM_MODULE`、`QT_IM_MODULE` 一概清除（意在交桌面協議），`XMODIFIERS` 則取自 `~/.config/imsettings/xinputrc`；無此檔者按 locale 推之，zh_CN 而無配置者落至 none.conf，得 `@im=none`。故 XWayland 程式（Wine 等）之輸入，須令 xinputrc 指向 fcitx5 之配置。

又：雙拼非另立引擎，而為獨立輸入法（名 `shuangpin`，屬 addon `pinyin`），其方案（`ShuangpinProfile`）存於 `~/.config/fcitx5/conf/pinyin.conf`（輸入法配置默認即寫 addon 之配置）。全拼（`pinyin`）與之並存，切換用之。

## 步驟

一、裝包（`fcitx5-gtk`、`fcitx5-qt` 為 GTK/Qt 之 immodule，以備 immodule 路徑之需）：

```bash
sudo dnf install fcitx5 fcitx5-chinese-addons fcitx5-configtool fcitx5-gtk fcitx5-qt
```

二、令 kwin 掌輸入法：

```bash
kwriteconfig6 --file kwinrc --group Wayland --key InputMethod org.fcitx.Fcitx5.desktop
```

三、令 `XMODIFIERS` 及於 XWayland 程式：

```bash
ln -sf /etc/X11/xinit/xinput.d/fcitx5.conf ~/.config/imsettings/xinputrc
```

四、兜底二事：systemd 用戶會話設 `XMODIFIERS`，並備自啟（防 kwin 未起之時）：

```bash
mkdir -p ~/.config/environment.d
echo 'XMODIFIERS=@im=fcitx5' > ~/.config/environment.d/fcitx5.conf
mkdir -p ~/.config/autostart
cp /usr/share/applications/org.fcitx.Fcitx5.desktop ~/.config/autostart/
```

`environment.d` 之文件名不可用 `imsettings*.conf`，以免為 xinput.sh 所刪。

五、加中文輸入法於 `~/.config/fcitx5/profile`，於 `[Groups/0]` 諸項之後依序添之：

```ini
[Groups/0/Items/2]
# Name
Name=shuangpin
# Layout
# Layout=

[Groups/0/Items/3]
# Name
Name=pinyin
# Layout
# Layout=
```

六、定小鶴方案：

```bash
mkdir -p ~/.config/fcitx5/conf
printf '[Pinyin]\nShuangpinProfile=Xiaohe\n' > ~/.config/fcitx5/conf/pinyin.conf
```

七、註銷重入（或重啟），諸設定方生效。

## 驗證

```bash
pgrep -a fcitx5        # 望見 fcitx5 進程（多為 kwin 所育）
fcitx5-diagnose        # 察環境變數與模組
```

於 Kate、Firefox 等程式試敲：按 `Ctrl+Space` 啟用 fcitx5，循環至「雙拼」（循環序即 profile 中 Items 之序）；`Shift` 切中英。小鶴鍵位之驗：「中」作 `vs`，「国」作 `go`，故「中国」當鍵 `vsgo`。

## 疑難

- 既設 kwinrc 而 Wayland 程式仍無輸入法：於「系統設置 → 鍵盤 → 虛擬鍵盤」重選 Fcitx 5（KDE 自寫同値，可規範格式）。
- 某程式獨不通：Chromium 系或須 `--enable-wayland-ime`，否則宜以 X11 運行（ozone）；X11 程式則仰 `XMODIFIERS`（見步驟三、四）。
- 未裝 `fcitx5-chinese-addons` 而先啟 fcitx5：profile 中 `shuangpin`、`pinyin` 等無效條目將為 fcitx5 所清除，須於裝包後重寫或於 fcitx5-configtool 中重加。故裝包宜先於設定。
- 全拼與雙拼互斥：所用者為哪一輸入法，即以何鍵位解讀；切換輸入法即可改用另一。
- 察 fcitx5 是否為 kwin 所育（後端之頼）：`ps -e -o pid,ppid,comm | grep fcitx5` 觀其父進程。

## 回退

```bash
kwriteconfig6 --file kwinrc --group Wayland --key InputMethod --delete
rm -f ~/.config/imsettings/xinputrc
rm -f ~/.config/environment.d/fcitx5.conf
rm -f ~/.config/autostart/org.fcitx.Fcitx5.desktop
```

欲改用 IBus 者，見《IBus 智能拼音之雙拼設定》（`fedora/input/ibus-libpinyin-double-pinyin.md`）。
