# Fira Code 缺字時回退思源宋體（fontconfig 用戶規則）

## 環境

- 系統：Fedora 44
- 桌面：KDE Plasma 6.6.4（Wayland 會話）
- 內核：7.2.7-200.fc44.x86_64
- 字體：Fira Code（拉丁、符號、數字）；思源宋體（Source Han Serif SC，全字符）
- 前提：fontconfig 2.17（用戶級規則由 `/etc/fonts/conf.d/50-user.conf` 引入）
- 撰寫：2026-09-27

## 適用

凡桌面之系統字體設為不含 CJK 之西文字體（如 Fira Code），而欲以他字體（如思源宋體）補其缺字者。於 KDE、GNOME 等凡賴 fontconfig 者皆同此理；若應用自帶字體設定（如瀏覽器），則不盡依此。

## 步驟

一、察字體之真名（fontconfig 以族名匹配，非檔名）：

```bash
fc-list : family | sort -u | grep -iE "fira|serif"
```

本機所得：`Fira Code`、`思源宋體,Source Han Serif SC`。

二、立用戶規則檔 `~/.config/fontconfig/conf.d/99-fira-code-cjk-fallback.conf`：

```xml
<?xml version="1.0"?>
<!DOCTYPE fontconfig SYSTEM "urn:fontconfig:fonts.dtd">
<fontconfig>
  <!-- Fira Code 缺字（如中文）時，回退到思源宋體 Source Han Serif SC -->
  <match target="pattern">
    <test name="family" compare="contains">
      <string>Fira Code</string>
    </test>
    <edit name="family" mode="append" binding="strong">
      <string>Source Han Serif SC</string>
    </edit>
  </match>
</fontconfig>
```

說理：

- `target="pattern"` 乃於查詢之模式上動手；`mode="append"` 續於族名之後；`binding="strong"` 使其為確定之候選。
- `compare="contains"` 令 Fira Code Light、SemiBold 等變體亦蒙其澤。
- 候選之中，仍以 Fira Code 為首；遇其缺字，方取思源宋體。
- **不可**逕改 `~/.config/fontconfig/fonts.conf`——該檔為 KDE 之字體設定所管，稍改渲染參數即重寫，規則恐亡。

三、刷新：通常不必（fontconfig 每次讀取配置；`fc-cache` 只管字體目錄之緩存，與配置無涉）。

## 驗證

```bash
fc-match -s "Fira Code"                # 次位當見思源宋體
fc-match "Fira Code:charset=4e2d"      # 「中」字，當得思源宋體
```

已開之應用須重啟方循新規；Plasma 外殼（標題欄、選單）宜登出再入。

## 疑難

- `fc-match -s` 不見思源宋體：族名有誤（以 `fc-list` 覆核）、XML 有誤（`fc-match` 遇錯會警告）、或系統無 `50-user.conf`（可 `grep -r xdg /etc/fonts/fonts.conf` 察之）。
- 僅部分應用生效：應用自定字體者（瀏覽器、Electron 應用）不依 fontconfig。
- 規則消失：誤置於 `fonts.conf`（KDE 重寫之故），當移入 `conf.d/`。

## 回退

```bash
rm ~/.config/fontconfig/conf.d/99-fira-code-cjk-fallback.conf
```

## 附註

此法可推廣：欲「甲字體缺字取乙字體」者，仿此增一 `<match>`；欲易系統之 CJK 字體者，則當於 `sans-serif`／`serif` 之 `prefer` 列表著墨。系統級同類規則見 `/etc/fonts/conf.d/`（如 `65-nonlatin.conf`），然發行版更新可覆，宜仍以用戶規則為之。
