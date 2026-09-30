# Distrobox 容器內 NVIDIA Vulkan 之啟用（ICD 絕對路徑之修）

## 環境

- 宿主：Fedora 44（內核 7.2.7-200.fc44.x86_64）；KDE Plasma 6（Wayland 會話）
- 組件：rootless podman 5.8、distrobox 1.8.2.5；宿主 NVIDIA 驅動 615.71.09（開放內核模組）；nvidia-container-toolkit 1.20.1（其包不在 Fedora 官方庫，須自 NVIDIA 之倉安裝；CDI 規約生於 `/etc/cdi/nvidia.yaml`）
- 容器：distrobox 之 `ubuntu:26.04` 容器，建時帶 NVIDIA（`--nvidia`，或 `--additional-flags "--device nvidia.com/gpu=all"`）
- 相關硬體：AMD Radeon 610M（核顯，RADV，`renderD128`）＋ NVIDIA GeForce RTX 4070 Laptop（獨顯，`renderD129`）；KWin 合成器行於獨顯
- 前提：容器內 `nvidia-smi` 可用、`/dev/nvidia0` 已在；本例之應用為 GPUI 系（Zed 同門框架），其日誌足為憑證
- 撰寫：2026-09-30

## 適用

適用於：宿主庫居 `/usr/lib64`（Fedora、RHEL 系）而容器為 Debian 系（庫居 `/usr/lib/x86_64-linux-gnu`）之 distrobox 組合；容器內 **NVIDIA 之 GL 可用而 Vulkan 不可見**，遂令應用擇錯顯卡、Wayland 下 dma-buf 遭合成器拒收者。

不適用：容器全然無 NVIDIA（`nvidia-smi` 亦不可用）者——彼當先修容器之入卡；本文所治，唯在 ICD 清單之絕對路徑與容器庫位之錯配。

## 症狀

- 容器內 `nvidia-smi -L` 有卡、`/dev/nvidia0` 在、`/usr/share/vulkan/icd.d/nvidia_icd.x86_64.json` 亦在；
- 然 Vulkan 清單中無 NVIDIA。GPUI 系應用之日誌如次：

```
Found 3 GPU adapter(s):
  - AMD Radeon 610M (RADV ...) (vendor=0x1002, device=0x164e, backend=Vulkan, type=IntegratedGpu)
  - NVIDIA GeForce RTX 4070 Laptop GPU/PCIe/SSE2 (vendor=0x10de, device=0x0000, backend=Gl, type=Other)
  - llvmpipe (LLVM ...) (vendor=0x10005, device=0x0000, backend=Vulkan, type=Cpu)
Selected GPU adapter: "AMD Radeon 610M (RADV ...)" (Vulkan)
```

- 合成器行於 NVIDIA，而應用以核顯出幀，其 dma-buf 跨界被拒，遂海量刷屏：

```
ERROR wayland_backend: [destroyed object]: error 7: importing the supplied dmabufs failed
ERROR wayland_backend::sys::client_impl: Protocol error 7 on object @0
```

## 步驟

### 一、驗其虛實

```bash
# 觀 ICD 清單所指之路徑
distrobox enter <容器> -- cat /usr/share/vulkan/icd.d/nvidia_icd.x86_64.json
# 觀該路徑在容器內有無（Fedora 習俗之路）
distrobox enter <容器> -- ls -l /usr/lib64/libGLX_nvidia.so.0
# 觀真庫所在（Debian 習俗之路）與 ld 緩存所認
distrobox enter <容器> -- ls /usr/lib/x86_64-linux-gnu/ | grep -i nvidia
distrobox enter <容器> -- ldconfig -p | grep -i nvidia
```

若清單所書為絕對路徑 `/usr/lib64/libGLX_nvidia.so.0`，而該處無物、真庫卻在 `/usr/lib/x86_64-linux-gnu/`——則中此症。

按：`<容器>` 代 distrobox 容器之名（本例 `longbridge`）。RHEL 系之清單名多作 `nvidia_icd.x86_64.json`，Debian 系或作 `nvidia_icd.json`。

### 二、立鏈以補路

```bash
distrobox enter <容器> -- sudo ln -sf \
  /usr/lib/x86_64-linux-gnu/libGLX_nvidia.so.0 /usr/lib64/libGLX_nvidia.so.0
```

其理：

- 實測所見：宿主驅動庫落於**容器本土庫目錄**（本例 `/usr/lib/x86_64-linux-gnu/`），並令 ld 緩存認之，故其餘依賴可自緩存而解；
- 而 Vulkan ICD 清單乃自宿主**原樣**掛入，其內 `library_path` 為絕對路徑（Fedora 為 `/usr/lib64/...`）；容器無此路，裝載器 dlopen 撲空而**默棄**該驅動；
- GL 一路所以獨存者：glvnd 之 vendor 清單（`/usr/share/glvnd/egl_vendor.d/10_nvidia.json`）用**裸 soname**，賴緩存尋得。同源異途，此其所以一見一不見。

## 驗證

```bash
timeout 10 distrobox-enter -n <容器> -- <應用>
```

- 「Found N GPU adapter(s)」中 NVIDIA 以 `backend=Vulkan, type=DiscreteGpu, device=0x<id>` 列居首位，且 `Selected GPU ... NVIDIA`；
- 不復見 `importing the supplied dmabufs failed` 與 `Protocol error 7`；日誌由百萬行歸於數十行，即明證。

按：`<應用>` 指容器內可執行之路徑（本例 `/usr/bin/longbridge-desktop`）。`timeout` 惟殺宿主一面，容器內遺留之進程宜清：

```bash
distrobox enter <容器> -- pkill -f 'longbridg[e]'
```

模式用字元類（`[e]`），以免 `pkill` 誤中自身。

## 疑難

| 症狀 | 成因 | 對策 |
|---|---|---|
| 容器內 `nvidia-smi` 亦不可用 | NVIDIA 未入容器（CDI 或 `--nvidia` 未生效） | 查宿主 `/etc/cdi/nvidia.yaml`；或以 `--additional-flags "--device nvidia.com/gpu=all"` 重建容器 |
| Vulkan 已見 NVIDIA，應用仍擇他卡 | 應用自有擇卡之序（GPUI 之序：`ZED_DEVICE_ID` > 與合成器同卡 > 獨顯 > 核顯 > CPU） | 以 `ZED_DEVICE_ID=<四位十六進位之 PCI device id>` 指定，如 `2820`（RTX 4070 Laptop，示例）；日誌現 `ZED_DEVICE_ID filter: 0x...` 即證其支持 |
| NVIDIA 之 GL 適配器報 `device=0x0000`、`type=Other` | GL 後端不報 device id，故同卡匹配與 `ZED_DEVICE_ID` 皆失 | 屬正常；治本須使之見 Vulkan |
| 已擇同卡而 dma-buf 仍敗 | 或 `nvidia_drm` 未啟 modeset | `cat /sys/module/nvidia_drm/parameters/modeset`，宜為 `Y` |
| 急用而不求硬加速 | — | 令應用走 X11（啟動前 `unset WAYLAND_DISPLAY`）：GPU 不合時應用自退 llvmpipe 軟渲染，可用而不快 |
| 以 `VK_DRIVER_FILES` 只留軟驅以逼擇卡 | — | 曾試之，應用默然而死（零輸出），緣故未明，慎用 |

## 回退

```bash
distrobox enter <容器> -- sudo rm -f /usr/lib64/libGLX_nvidia.so.0
```

除鏈即復舊觀。又：補鏈存於容器之可寫層，重啟不丟；**`distrobox rm` 重造容器後須重行**（連同重裝應用、重新導出諸事）。

## 兼容性補註

- 根在兩系庫目錄之別：Fedora/RHEL 系庫居 `/usr/lib64`，Debian 系居 `/usr/lib/x86_64-linux-gnu`。反向之組合（Debian 宿主＋Fedora 容器）若遇同症，理亦同法，惟兩端路徑互換，須察其實書。
- 未能於他版一一驗證；凡「GL 見而 Vulkan 不見」者，皆可先察 ICD 清單之 `library_path` 與實際庫位——此法可推。
