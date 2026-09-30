# Distrobox 中微信 deb 之缺庫與段錯誤排查

## 環境

- 系統：Fedora 44（內核 7.2.7-200.fc44.x86_64）；KDE Plasma 6（Wayland 會話）
- 容器：distrobox `tencent`，基底 `docker.io/library/ubuntu:26.04`；另以 `ubuntu:24.04` 對照，其症相同
- 組件：rootless podman 5.8；容器由 distrobox 建，帶 userns keep-id
- 相關硬體：NVIDIA RTX 4070 Laptop（專有驅動 615.71.09）；容器內無 NVIDIA 用戶態庫
- 前提：微信官方 deb（`WeChatLinux_x86_64.deb`，示例 4.1.13.23）
- 撰寫：2026-09-27

## 適用

於容器（distrobox 等）中自 deb 裝官方微信 Linux 版，啟動即 `Segmentation fault (core dumped)`、補 `libnss3` 後仍崩者。其理亦可推於他類 deb 應用：**主二進制以 `ldd` 查之無缺，而啟動即段錯誤**者，多係 dlopen 之庫未備而程序未檢其 NULL。宿主原生安裝或 flatpak 版不在此列。

## 步驟

### 一、補齊顯性依賴

官方 deb 之 `control` 僅聲字體之賴，餘皆運行時自理，須手補：

```bash
sudo apt install -y \
  libnss3 libxkbcommon-x11-0 libxcb-cursor0 libxcb-icccm4 libxcb-image0 \
  libxcb-keysyms1 libxcb-render-util0 libxcb-shape0 libxcb-xinerama0 \
  libxcb-xfixes0 libpulse0 libasound2t64
```

其中 **`libpulse0` 最為關鍵**：微信啟動百餘毫秒時 dlopen `libpulse.so.0`，得 NULL 而不檢，遂蹈空指針之禍；此庫不在 `ldd` 之列，故尋常查驗不覺。Ubuntu 24.04 及以降，ALSA 包名為 `libasound2t64`。

### 二、若仍段錯誤，以 strace 覓其最後所尋之庫

```bash
sudo apt install -y strace
strace -f -tt -o /tmp/wx.trace wechat
grep "openat.*\.so" /tmp/wx.trace | tail -20
```

末行屢試而不得者（如 `/usr/lib/libpulse.so.0`），即元凶；以 `apt-file search libpulse.so.0` 得其包而補之。

崩之狀可以下法驗之：gdb 停於 `rip=0x0`、minidump 記 `si_code=SEGV_MAPERR, si_addr=NULL`——皆空指針召喚之徵。

### 三、對照之法（辨咎在容器抑或程序）

同一二進制於宿主直跑，若安然，則咎在容器缺庫，非程序或內核之過：

```bash
mkdir -p /tmp/wh && cd /tmp/wh
ar p ~/下载/WeChatLinux_x86_64.deb data.tar.xz | tar -xJ
LD_LIBRARY_PATH=/tmp/wh/opt/wechat timeout 20 /tmp/wh/opt/wechat/wechat
```

宿主庫齊（如 Fedora 自帶 `libpulse`），故不補而通；容器精簡，故補而後通。

## 驗證

```bash
wechat
```

無 `Segmentation fault`，登錄窗現；`ls ~/.xwechat/crashinfo/completed` 不復增 `.dmp`。

## 疑難

| 症狀 | 成因 | 對策 |
|---|---|---|
| `error while loading shared libraries: libnss3.so` | 缺 NSS | `sudo apt install libnss3` |
| 補後仍潰，或悄然而亡 | dlopen 之庫缺（`libpulse.so.0` 等），NULL 未檢 | `sudo apt install libpulse0`；以 strace 定位 |
| `WeChatAppEx: ... libasound.so.2` | 小程序子進程須 ALSA | `sudo apt install libasound2t64` |
| `driver (null)`、`failed to load driver: nvidia-drm`、`KMS: DRM_IOCTL_MODE_CREATE_DUMB failed: Permission denied` | 容器無 NVIDIA 用戶態庫，`card1` 屬宿主 gid 而未映射 | 無害，自退軟件渲染；欲用硬體加速，詳姊妹篇《Distrobox 容器內 NVIDIA Vulkan 之啟用》（`distrobox-nvidia-vulkan-icd.md`） |
| `Gtk-Message: Failed to load module "..."` | 宿主桌面之 GTK 模組 | 無害，可置之不問 |

## 回退

```bash
sudo apt remove libpulse0 libasound2t64
```

然此二庫他日或為桌面所賴，不宜輕卸。

## 兼容性補註

- 所測 `ubuntu:24.04`、`ubuntu:26.04` 精簡基底皆不預裝 `libpulse0`、`libasound2t64`；宿主 Fedora 則已具。
- 微信 deb 之依賴以實測為準（示例 4.1.13.23，2026-09 構建）；新舊版或有出入，理法不變。
- 容器中用軟件渲染即可；欲借宿主 NVIDIA，須版本相符之用戶態庫，非本文所詳。
