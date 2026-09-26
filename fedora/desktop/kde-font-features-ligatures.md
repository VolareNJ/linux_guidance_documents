# KDE 字體特性欄與 Fira Code 連字

## 環境

- 系統：Fedora 44
- 桌面：KDE Plasma 6.6.4（Wayland 會話）
- 內核：7.2.7-200.fc44.x86_64
- 組件：Qt 6.10.2；KDE Frameworks 6.25（KConfig、KWidgetsAddons）；Konsole 25.12.3
- 字體：Fira Code
- 撰寫：2026-09-27

## 適用

凡於 KDE Plasma 之「系統設定 → 文本與字體 → 字體」中，欲為字體開啟 OpenType 特性（如編程連字所用之 `calt`、`liga`）者。此法於諸 Qt 應用一併生效，不必逐個應用而設。

## 原委

編程連字出於 OpenType 之 GSUB 特性：Fira Code 之連字多由 `calt`（contextual alternates）供之，部分由 `liga`（standard ligatures）。此二者不在 fontconfig 之權內——fontconfig 只司字體匹配與渲染參數，不通特性——故當自 KDE 之字體設定入手。

KDE 之字體對話框（KFontChooser）藏有「字體特性」欄，其值隨 QFont 存入 `kdeglobals`（`[General]` 之 `font=`、`fixed=` 等），凡讀此設定之 Qt 應用（KDE 桌面、Konsole、KWrite/Kate、Dolphin 等）皆用之。

## 步驟

一、系統設定 → 文本與字體 → 字體。

二、於欲改之字體項（「常規」「固定寬度」等）右側點字體按鈕，開字體對話框。

三、於對話框下方「字體特性」（Font features，提示為 Comma separated list of font features (e.g., liga, calt)）填入：

```
liga, calt
```

其語法（據 KFontChooser 之 `slotFeaturesChanged()`）：逗號分隔；每項或為四字元標籤（如 `liga`、`calt`），即啟用之（值為 1）；或為 `標籤=數值`（如 `dlig=1`、`liga=0`）——0 為關、1 為開，更大者取該特性之第 n 種替代字形；空欄即清除一切特性。標籤逾四字元而又無「=」者，將被忽略。

四、確定，復按「應用」。

五、六項字體皆欲如是者，須逐項為之。**「調整所有字體…」不可代勞**——據 kcm_fonts 之 `applyFontDiff()`，其所傳者惟大小、家族、樣式，不含特性。

## Konsole 之別調

Konsole 之連字，另繫於其自身之渲染設定——終端逐格排列，非合併成串則無從形變。須開「複雜文字排版」（Complex Text Layout）中之「單詞模式」與「ASCII characters」（即 `WordMode` 與 `WordModeAscii`）。

無圖形時可逕書之。

`~/.local/share/konsole/Default.profile`：

```ini
[General]
Name=Default
Parent=FALLBACK/

[Appearance]
WordMode=true
WordModeAscii=true
```

`~/.config/konsolerc`：

```ini
[Desktop Entry]
DefaultProfile=Default.profile
```

說理：profile 檔必含 `Name=`，否則 Konsole 棄之；`Parent=FALLBACK/` 者，承內置默認（字體仍取系統等寬字體，即 `kdeglobals` 之 `fixed`）。

## 驗證

- 重開字體對話框：特性欄當仍見 `liga, calt`（若已空，則未存住）。
- `grep '^font=' ~/.config/kdeglobals`：設特性後其值有變（本機 KConfig 之庫引用 `QDataStream` 以存取 QFont，含特性者或以二進制編碼存之，明據可覆按）。
- 新開一 Qt 應用（KWrite 等），書 `<=`、`!=`、`=>` 觀其形變。
- Konsole：開新視窗，`echo '<='`。

## 疑難

- 已開之應用不隨：KDE 於應用後發 `refreshFonts` 訊號，多數應用即更；少數須重啟。
- 非 Qt 應用（Firefox、GTK 應用）不與焉——它們不讀 `kdeglobals` 之字體。
- Konsole 無連字：其單詞模式未開（見上）。
- 特性欄所填標籤須為四字元者，誤填將被默然忽略。

## 回退

於特性欄清空，或逐項重設字體。動 `kdeglobals` 前宜備份：

```bash
cp ~/.config/kdeglobals ~/.config/kdeglobals.bak
```

## 附註

- 「字體特性」欄繫於 KDE Frameworks 6（本機 6.25，隨 Fedora 44 之 Plasma 6.6.4）；KDE 5 無之。
- 系統設定之「字體」頁面本身無此欄，須入字體對話框方見。
- 此文所記與 Zed 之 `font_features`（`{"liga": true, "calt": true}`）不同法：兩者各依其應用之機制。
