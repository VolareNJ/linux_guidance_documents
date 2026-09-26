# IBus 智能拼音之雙拼設定

## 環境

- 系統：Fedora 44
- 桌面：KDE Plasma 6.6.4（Wayland 會話）
- 組件：ibus 1.5.34、ibus-libpinyin 1.16.5
- 撰寫：2026-09-27

## 適用

凡用 ibus-libpinyin（界面名曰「智能拼音」）而欲改雙拼者，含小鶴、自然碼、微軟、紫光、拼音加加諸方案。他種發行版同理，惟 gsettings 之 schema 名隨版本而異。

## 其理

雙拼非另立輸入法，乃同一引擎之一模式。其設定存於 gsettings：

- schema：`com.github.libpinyin.ibus-libpinyin.libpinyin`
- `double-pinyin`（布林）：是否啟用雙拼
- `double-pinyin-schema`（整數）：方案之序，為 MSPY=0、ZRM=1、ABC=2、ZGPY=3、PYJJ=4、XHE=5（小鶴）

## 圖形之法

```bash
/usr/libexec/ibus-setup-libpinyin
```

此器置於 `/usr/libexec`，不在 `PATH`，故須全路徑；亦可由 `ibus-setup` 進入「智能拼音」之「首選項」。

於面板中：「拼音模式」選「雙拼」，「雙拼方案」選「小鶴」。

## 命令行之法

```bash
gsettings set com.github.libpinyin.ibus-libpinyin.libpinyin double-pinyin true
gsettings set com.github.libpinyin.ibus-libpinyin.libpinyin double-pinyin-schema 5
ibus restart
```

多數即時生效；若不然，重啟 ibus，或切換一次輸入源。

## 驗證

```bash
gsettings get com.github.libpinyin.ibus-libpinyin.libpinyin double-pinyin          # 望得 true
gsettings get com.github.libpinyin.ibus-libpinyin.libpinyin double-pinyin-schema   # 望得 5
```

小鶴鍵位之驗：「中」作 `vs`（zh→v、ong→s），「国」作 `go`，故「中国」當鍵 `vsgo`。

## 疑難

- 全拼失靈：雙拼與全拼互斥，既開雙拼，即依雙拼鍵位解讀。欲復全拼，將 `double-pinyin` 設回 `false`。
- 設定不靈：確認所用輸入源為「智能拼音」（引擎名 `libpinyin`），非他種拼音表。
- 無此 schema：`gsettings list-schemas | grep -i libpinyin` 察之；若無，則未裝 ibus-libpinyin。
- 圖形面板不開：`python3` 須在，且須有圖形會話。

## 回退

```bash
gsettings set com.github.libpinyin.ibus-libpinyin.libpinyin double-pinyin false
gsettings set com.github.libpinyin.ibus-libpinyin.libpinyin double-pinyin-schema 0
ibus restart
```
