# Linux 經驗備忘

此庫收藏 Linux 使用與維護之經驗，以文言書之，供日後查考。

## 目錄之例

- 第一層為環境：`fedora/`、`debian/`、`arch/` 等；跨發行版而通用者入 `generic/`。
- 第二層為主題：`graphics/`（顯示與驅動）、`containers/`（容器與隔離）、`desktop/`（桌面與應用）、`input/`（輸入法）、`packaging/`（打包與套件）、`system/`（系統與引導）、`network/`（網路）等。
- 檔名用小寫英文與連字號（kebab-case），如 `nvidia-driver-install.md`。

## 體例

1. 文首必列「環境」：發行版與版本、桌面與會話、內核、相關硬體，及他項前提（如 Secure Boot）。
2. 次列「適用」：何時可用、何時不可用。
3. 正文以命令為主，輔以文言說其理，不作空談。
4. 末列「驗證」「疑難」「回退」。
5. 所記宜通用；過於個別者（一時之版本號、個人路徑、偶發之現象）不錄，或置於附註。
6. 若他環境之經驗可施於本機環境，於「環境」處補註其可用之版本範圍，勿另開新文。
7. 勿錄隱私：帳號、密碼、內網位址、授權序號皆不入庫。
8. 一文一事，勿相糾纏。

## 索引

| 路徑 | 主題 |
|---|---|
| `fedora/graphics/nvidia-driver-install.md` | Fedora 上安裝 NVIDIA 專有驅動（RPMFusion 與官方安裝器兩路） |
| `fedora/containers/distrobox-deb-app.md` | 以 Distrobox 隔離安裝 Debian 系 deb 應用 |
| `fedora/desktop/zed-manual-install.md` | Zed 綠色包之安置與桌面整合 |
| `fedora/input/ibus-libpinyin-double-pinyin.md` | IBus 智能拼音之雙拼設定（小鶴等） |

## 版本管理

本庫以 git 記之。每增刪文檔，宜 `git add` 與 `git commit`，訊息簡明，如「補：IBus 雙拼設定」。

```bash
cd ~/文档/linux_guidance_documents
git add -A
git commit -m "補：IBus 雙拼設定"
```
