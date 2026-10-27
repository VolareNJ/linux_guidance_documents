# TeX Live 中文排版之部署（Fedora，XeLaTeX）

## 环境

- 系统：Fedora release 44 (Forty Four)，dnf 5
- 桌面：KDE Plasma，Wayland 会话
- 内核：7.2.11-200.fc44.x86_64
- 相关硬件：无特殊要求
- 前提：Secure Boot 已启用（与本流程无涉）；需 sudo 权限
- 撰写：2026-10-27

## 适用

Fedora 41–44 之 dnf 安装，已具 `texlive-collection-basic`、`texlive-collection-latex`、`texlive-xetex`，而欲以 XeLaTeX 排中文者。若已装 `texlive-collection-langchinese`，可径入「验证」。不适用于以 `install-tl` 安装之上游 TeX Live，其套件管理与此异。

## 步骤

### 一、察其现状

```bash
rpm -q texlive-collection-basic texlive-collection-latex texlive-xetex
kpsewhich ctexart.cls xeCJK.sty zhnumber.sty cjkpunct.sty
```

`kpsewhich` 无输出者，即缺其件。

### 二、装中文支持

最小之装：

```bash
sudo dnf install texlive-ctex texlive-xecjk texlive-zhnumber texlive-cjkpunct
```

或装全中文集合：

```bash
sudo dnf install texlive-collection-langchinese
```

Fedora 之 TeX Live 分拆为多包，无单一 `texlive` 总包；`texlive-ctex` 会依赖 xeCJK、zhnumber、fandol、cjk 等，dnf 自带入。全集合含中日韩，体积较大。若用 `fontset=none` 自定字体，可省 fandol，然依赖仍由 dnf 决定。

### 三、备字体

```bash
fc-list :lang=zh family | sort -u
fc-match 'Source Han Serif SC'
```

本例用思源宋体（`Source Han Serif SC`，本机为 `/usr/local/share/fonts/s/SourceHanSerif.ttc`）。若无，可装：

```bash
sudo dnf install adobe-source-han-serif-cn-fonts
```

或：

```bash
sudo dnf install google-noto-serif-cjk-fonts
```

字体族名以 `fc-list` 所示为准：CN 版多作 `Source Han Serif CN`，Noto 版作 `Noto Serif CJK SC`。

### 四、作最小测试

```bash
cat > test-ctex.tex <<'TEST_EOF'
\documentclass[fontset=none]{ctexart}
\usepackage{fontspec}
\setmainfont{Liberation Serif}
\setCJKmainfont{Source Han Serif SC}
\begin{document}
中文测试：宋体与西文混排 ABC 123。
\end{document}
TEST_EOF
xelatex test-ctex.tex
```

`fontset=none` 者，不用 ctex 内建字体集，而自定之；若用默认 fandol，须装 `texlive-fandol`。`Liberation Serif` 为系统常见之 TrueType 字体，与 Times 度量相容。

### 五、验证

```bash
pdfinfo test-ctex.pdf | grep Pages
pdffonts test-ctex.pdf
pdftotext test-ctex.pdf - | head -3
xelatex --version | head -1
```

预期：一页；`pdffonts` 见 `SourceHanSerif` 与 `LiberationSerif`，`emb` 皆为 `yes`；文字可提取。

## 疑难

- `! LaTeX Error: File 'ctexart.cls' not found.`：未装 `texlive-ctex`。
- `xdvipdfmx:fatal: Cannot proceed without the font: .../NimbusRoman-Regular.t1`：`\setmainfont` 指 Type1 字体，xdvipdfmx 不能嵌之。改用 TrueType/OpenType，如 `Liberation Serif`。
- `Package fontspec Error: The font "..." cannot be found`：字体名不符或未装。以 `fc-match '名'` 核之；TTC 内之 Bold、Light 等面，先只用族名，勿径指面名。
- `Missing character: There is no ... in font ...`：字体缺字，或文中有零宽字符（U+200B）。去之，或换更全之 CJK 字体。
- 中文断行不当、标点居行首：未用 ctex 或 xeCJK 之 CJK 排版。用 `ctexart`，或于导言区加 `\usepackage{xeCJK}`。
- `! Package ctex Error: CTeX fontset 'fandol' is unavailable`：装 `texlive-fandol`，或用 `fontset=none` 自定字体。
- 字体已装而仍报无：`fc-cache -fv` 更新缓存，重编。
- `xelatex: command not found`：装 `texlive-xetex`。

## 回退

```bash
sudo dnf history
sudo dnf history undo <ID>
```

或只除新装之件：

```bash
sudo dnf remove texlive-ctex texlive-xecjk texlive-zhnumber texlive-cjkpunct
```

若另装字体，`sudo dnf remove adobe-source-han-serif-cn-fonts`，复以 `fc-cache -fv`。移除 TeX 套件不影响既有 PDF。

## 附：latexmk

```bash
sudo dnf install latexmk
latexmk -xelatex test-ctex.tex
latexmk -c
```

`latexmk` 于 Fedora 为独立包，非 texlive 子包；自动多次编译，适于有目录、交叉引用之文稿。

## 兼容性补注

Fedora 41–44 之套件名同；TeX Live 2025/2026 之 ctex 版本略有差异，命令不变。
