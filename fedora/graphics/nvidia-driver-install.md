# Fedora 上安裝 NVIDIA 專有驅動

## 環境

- 系統：Fedora 44
- 桌面：KDE Plasma 6.6.4（Wayland 會話；X11 經 XWayland，`DISPLAY=:0`）
- 內核：6.19.10-300.fc44.x86_64
- 硬體：雙顯卡。NVIDIA 10de:2820（RTX 40 系筆電卡，時由 nouveau 驅動）；AMD 1002:164e（核顯，amdgpu 主管顯示）
- 前提：Secure Boot 已啟用（`mokutil --sb-state` 示 enabled）；`gcc`、`make`、`kernel-headers` 已在；`kernel-devel`、`dkms`、`akmod-nvidia` 未裝；RPMFusion 未加；`/usr/src/kernels/` 空無一物
- 前情：官方安裝器 `~/NVIDIA-Linux-x86_64-595.104.02.run` 曾行而未成，報「Unable to find the kernel source tree」；其已留 `/etc/modprobe.d/nvidia-installer-disable-nouveau.conf`（blacklist nouveau），然未重建 initramfs，nouveau 尚在載中
- 撰寫：2026-09-27

## 適用

凡 rpm 系發行版欲易 nouveau 為 NVIDIA 專有驅動者。Secure Boot 之簽名一節，凡啟用 Secure Boot 者皆須；餘者可略。他版之包名與倉址隨版本而異，策則不變。

## 風險

- Secure Boot 只載由已註冊金鑰所簽之模組，未簽者見拒（`dmesg` 常見 `Key was rejected by service`）。
- 筆電雙顯卡：主輸出或為核顯，NVIDIA 為離線之 GPU；欲其參與渲染，須 PRIME offload。外接埠之歸屬因機而異，宜實測。
- 勿中途重啟：nouveau 既入黑名單而 nvidia 尚未編成，則兩者皆不載，或落至軟體渲染；此非資料之損，可自文字終端修之。
- 內核升級後，akmod 自動重建，`.run` 則須手動重編。升級之後，宜待其編成再重啟。
- 先備一可開機之舊內核，並知自 GRUB 入文字終端之法。

## 前置檢查

```bash
lspci -nnk | grep -A 3 -E "VGA|3D|Display"
uname -r
ls /usr/src/kernels/
rpm -q gcc make kernel-headers kernel-devel dkms akmod-nvidia
mokutil --sb-state
mokutil --list-enrolled
dnf repolist
```

```bash
ls /etc/modprobe.d/ | grep -i -E "nouveau|nvidia"
cat /etc/modprobe.d/nvidia-installer-disable-nouveau.conf
lsmod | grep -E "nouveau|nvidia"
ls "/usr/lib/modules/$(uname -r)/extra/" 2>/dev/null
```

## 首選：RPMFusion 之 akmod

akmod 由 RPMFusion 供之：安裝或升內核時觸發 akmods 服務，自動為當前內核編譯並簽名，卸載交由 dnf。

### 加倉

```bash
sudo dnf install https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-44.noarch.rpm
sudo dnf install https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-44.noarch.rpm
dnf repolist | grep -i rpmfusion
```

版本號 44 依所行發行版改之。

### 先察官方倉

近歲 Fedora 與 NVIDIA 合作，官方倉或已有 nvidia-open 之 kmod，宜先查：

```bash
dnf search nvidia
dnf list 'kmod-nvidia*' 'nvidia-open*' 'akmod-nvidia*'
```

視 `Repository` 一欄：若在官方倉且有 open 模組，則優先取之；否則用 RPMFusion。

### 黑名單之取捨

官方 `.run` 曾留 `nvidia-installer-disable-nouveau.conf`。akmod 安裝時自會置其黑名單，前檔可留可刪；若疑其相擾，去之：

```bash
sudo rm -f /etc/modprobe.d/nvidia-installer-disable-nouveau.conf
```

### Secure Boot 之簽名

```bash
sudo dnf install akmods
sudo kmodgenca -a
sudo mokutil --import /etc/pki/akmods/certs/public_key.der
# 設一口令；重啟時於藍色 MOK Manager 選 Enroll MOK 並輸之
mokutil --list-new        # 察待註冊之請求
```

無 Secure Boot 者，此節可略。

### 裝驅動

```bash
sudo dnf install akmod-nvidia xorg-x11-drv-nvidia-cuda
```

### 待其編成

裝後自動觸發編譯；可促之並觀其成敗：

```bash
sudo akmods --force
journalctl -u akmods -n 50 --no-pager
ls "/usr/lib/modules/$(uname -r)/extra/"     # 望見 nvidia*.ko
```

### 驗證

```bash
lsmod | grep nvidia
nvidia-smi
glxinfo -B | grep -i -E "vendor|renderer|OpenGL"
```

Wayland 之下更須 nvidia-drm 之 modeset：

```bash
cat /proc/cmdline | tr ' ' '\n' | grep -i nvidia
# 若缺，加之：
sudo grubby --update-kernel=ALL --args="nvidia-drm.modeset=1"
sudo dracut -f
```

雙顯卡之機，可視情形設 PRIME offload（按需渲染）以省電。

## 次選：官方 .run

須裝嚴合當前內核之 `kernel-devel`，否則重蹈「Unable to find the kernel source tree」之覆：

```bash
sudo dnf install "kernel-devel-uname-r == 6.19.10-300.fc44.x86_64" kernel-headers
ls /usr/src/kernels/        # 望見同名之目錄
```

而後自圖形會話之外執行安裝器（多於 TTY，或停顯示管理員時為之）：

```bash
sudo systemctl isolate multi-user.target
sudo ~/NVIDIA-Linux-x86_64-595.104.02.run
sudo systemctl isolate graphical.target
```

Secure Boot 之下，`.run` 所製之模組未簽，須自行簽名（`/usr/src/kernels/<ver>/scripts/sign-file` 或 `kmodsign`），並以 MOK 註冊其金鑰；手續繁，且每升內核須重行。故不薦。

卸之：

```bash
sudo ~/NVIDIA-Linux-x86_64-595.104.02.run --uninstall
sudo rm -f /etc/modprobe.d/nvidia-installer-disable-nouveau.conf
sudo dracut -f
```

## 疑難

| 症 | 因 | 治 |
|---|---|---|
| 無內核源碼樹 | `kernel-devel` 缺，或與 `uname -r` 不合 | `dnf install "kernel-devel-uname-r == $(uname -r)"` |
| 重啟後兩驅皆無 | nouveau 被黑而 nvidia 未成 | 入文字終端，復 nouveau，或續編之 |
| 模組載入被拒 | Secure Boot 驗簽未過 | 完成金鑰生成與 MOK 註冊，並重建 initramfs |
| `nvidia-smi` 失效 | 內核已升而模組未重建 | `akmods --force` 並待其成，或重啟 |
| 混合顯卡黑屏 | modeset 或 PRIME 之配置 | 加 `nvidia-drm.modeset=1`；必要時以 `nomodeset` 過渡 |

日誌所在：

```bash
dmesg | grep -i -E "nvidia|nouveau"
journalctl -b -u akmods --no-pager
```

## 回退

```bash
sudo dnf remove akmod-nvidia xorg-x11-drv-nvidia-cuda xorg-x11-drv-nvidia
sudo rm -f /etc/modprobe.d/nvidia-*.conf
sudo dracut -f
sudo akmods --force
# 重建畢，重啟，nouveau 當復載
```

## 附錄：常用檢查

```bash
lspci -nnk | grep -A 3 -i nvidia
lsmod | grep -E "nvidia|nouveau"
nvidia-smi
mokutil --sb-state
sudo akmods --force && ls "/usr/lib/modules/$(uname -r)/extra/"
grep -r . /etc/modprobe.d/*nvidia* 2>/dev/null
```

## 兼容性補註

本機為 Fedora 44 之記錄。他版之別：RPMFusion 之 release 包 URL 中之版本號、`akmods` 與 `kmodgenca` 之包名或有微異；RHEL 系可用 ELRepo 之 kmod-nvidia 代之。策略（先官方後第三方，Secure Boot 必簽）不變。
