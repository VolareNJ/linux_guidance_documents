# Qualcomm WCN7850（NCM865）於 CN 管域下 5GHz 全隱之修（本地補丁 ath12k 模組）

## 環境

- 系統：Fedora Linux 44（KDE Plasma Desktop Edition）
- 桌面：KDE Plasma（Wayland）
- 內核：7.2.7-200.fc44.x86_64；凡 6.16 至 7.2 之穩定分支皆有此疾
- 相關硬體：Qualcomm WCN785x Wi-Fi 7（FastConnect 7800；Foxconn NCM865 模組）；PCI 17cb:1107，子系 105b:e0f7；驅動 ath12k_wifi7_pci
- 前提：Secure Boot 已啟（自編模組須以 MOK 簽名方得載入）；已裝與運行內核相符之 kernel-devel
- 撰寫：2026-10-04

## 適用

- 疾狀：Wi-Fi 掃描唯見 2.4GHz；而 `iw phy` 明明示 5GHz 頻道俱在（此最惑人）。開機之初或暫見 5GHz，一連 2.4GHz，旋即隱沒。
- 成因：內核 6.16 起之提交 `0d777aa2ca77`；於「單 pdev」晶片（WCN7850 即然）且管域無 6GHz 規則（CN 即然）時，`ath12k_regd_update()` 計算所得 `ar->freq_range` 僅餘 2402–2482 MHz，發往固件之頻道表只十三個 2.4GHz 信道，故 5GHz 不可見。
- 上游補丁（Shenghan Gao，2026-07-15，題 "fix frequency range for single-pdev devices"）至 2026-10 尚未入 mainline；故暫以本地補丁模組代之。
- 不適用：非 WCN7850、或管域含 6GHz 規則、或別有他因者。又 CN 本無 6GHz，勿以 6GHz 缺失為病。

## 步驟

### 一、驗明正身

```bash
iw reg get | sed -n '/phy#0/,$p'          # 見 (self-managed) country CN
iw phy phy0 info | grep -E '5180|5745'    # 5GHz 頻道在冊，未 disabled
nmcli -f SSID,CHAN,FREQ dev wifi list     # 唯見 2.4GHz —— 乃本疾之徵
```

### 二、取源碼（僅 ath12k 子樹）與 kernel-devel

```bash
KV=$(uname -r)                            # 例：7.2.7-200.fc44.x86_64
mkdir -p ~/ath12k-fix
curl -fL -o ~/ath12k-fix/linux.tar.xz \
  https://mirrors.tuna.tsinghua.edu.cn/kernel/v7.x/linux-7.2.7.tar.xz
tar -xJf ~/ath12k-fix/linux.tar.xz -C ~/ath12k-fix linux-7.2.7/drivers/net/wireless/ath/ath12k
sudo dnf install "kernel-devel-$KV"
```

### 三、補 `reg.c` 三處（`ath12k_regd_update()` 內）

```diff
- phy_id = ar->pdev->cap.band[WMI_HOST_WLAN_2GHZ_CAP].phy_id;
+ phy_id = ar->pdev->cap.band[NL80211_BAND_2GHZ].phy_id;

- if (supported_bands & WMI_HOST_WLAN_5GHZ_CAP && !ar->supports_6ghz) {
+ if (supported_bands & WMI_HOST_WLAN_5GHZ_CAP &&
+     (!ar->supports_6ghz || ab->hw_params->single_pdev_only)) {

- phy_id = ar->pdev->cap.band[WMI_HOST_WLAN_5GHZ_CAP].phy_id;
+ phy_id = ar->pdev->cap.band[NL80211_BAND_5GHZ].phy_id;
```

### 四、編譯

```bash
SRC=~/ath12k-fix/linux-7.2.7/drivers/net/wireless/ath/ath12k
make -C /lib/modules/$(uname -r)/build M=$SRC modules -j$(nproc)
# 警語「Skipping BTF generation …」無害
```

### 五、簽名（Secure Boot 之必須）

```bash
MOK=~/ath12k-fix/mok
mkdir -p $MOK
openssl req -x509 -new -nodes -utf8 -sha512 -days 36500 -batch \
  -outform PEM -out $MOK/mok.pem -keyout $MOK/mok.key -subj '/CN=ath12k-5ghz-local-fix/'
openssl x509 -outform DER -in $MOK/mok.pem -out $MOK/mok.der
SF=/usr/src/kernels/$(uname -r)/scripts/sign-file
$SF sha512 $MOK/mok.key $MOK/mok.pem $SRC/ath12k.ko
$SF sha512 $MOK/mok.key $MOK/mok.pem $SRC/wifi7/ath12k_wifi7.ko
modinfo $SRC/ath12k.ko | grep signer        # 應見 ath12k-5ghz-local-fix
```

### 六、先註 MOK，後裝模組（免中途無線俱斷）

```bash
sudo mokutil --import $MOK/mok.der          # 設一次性密碼，記之
sudo reboot
# 重啟見藍屏 MokManager：Enroll MOK → Continue → 輸密碼；
# 其後之單（reboot / enroll key from disk / enroll hash from disk）擇 reboot 即可
```

### 七、安裝並重編索引（此處有坑，見「疑難」）

```bash
KV=$(uname -r)
sudo install -D -m0644 $SRC/ath12k.ko             /lib/modules/$KV/extra/ath12k/ath12k.ko
sudo install -D -m0644 $SRC/wifi7/ath12k_wifi7.ko /lib/modules/$KV/extra/ath12k/ath12k_wifi7.ko
sudo depmod -a "$KV"
modprobe --show-depends ath12k | tail -1    # 望其指 extra/；若仍指 kernel/，則：
printf 'override ath12k * extra/ath12k\noverride ath12k_wifi7 * extra/ath12k\n' \
  | sudo tee /etc/depmod.d/ath12k-fix.conf
sudo depmod -a "$KV"
modprobe --show-depends ath12k | tail -1    # 應見 extra/ath12k/ath12k.ko
sudo reboot
```

### 八、驗證

```bash
modinfo ath12k | grep -E 'filename|signer'  # extra/ 與 ath12k-5ghz-local-fix
cat /sys/module/ath12k/taint                # O —— 外掛模組確已載
nmcli -f SSID,CHAN,FREQ,SIGNAL dev wifi list
# 5GHz（5xxx MHz）現矣；再連 2.4GHz 而重掃，5GHz 仍在 —— 此步最要（驗 11d 更新後不復隱）
```

## 疑難

| 症狀 | 成因 | 對策 |
|---|---|---|
| 掃描唯 2.4GHz，`iw phy` 卻見 5GHz 頻道 | 本疾：freq_range 只餘 2.4GHz | 施補丁模組（如上） |
| 模組已裝，`modinfo ath12k` 仍指 `kernel/…` | depmod 未以 `extra/` 為先（Fedora 44 實遇） | `/etc/depmod.d` 加 override 並 `depmod -a` |
| 重啟後無線全無、日誌見 key rejected | MOK 未註，簽名不被信 | 再重啟，藍屏 `Enroll MOK` |
| 開機初見 5GHz，連 2.4GHz 後即隱 | 殘影；連網觸發 11d 監管更新而復發 | 屬本疾；補丁後自癒 |
| `depmod` 後 `modules.dep` 不見 `extra/ath12k/ath12k.ko` | 索引取捨之坑（同上） | override，並以 `modprobe --show-depends` 驗之 |

## 回退

```bash
KV=$(uname -r)
sudo rm -f /lib/modules/$KV/extra/ath12k/ath12k.ko /lib/modules/$KV/extra/ath12k/ath12k_wifi7.ko
sudo rm -f /etc/depmod.d/ath12k-fix.conf
sudo depmod -a "$KV"
sudo reboot          # 復用原廠模組
```

## 兼容性補註

- 上游補丁：[PATCH ath-current] wifi: ath12k: fix frequency range for single-pdev devices（Shenghan Gao，2026-07-15；patchwork 狀態 new）。其併入內核或 Fedora 回移之日，即可撤本模組。
- 他發行版：Debian/Ubuntu/Mint 等有同疾（6.16+ 內核 + WCN7850 + 無 6GHz 規則之管域）；修法一般無異（源碼 + 對應 headers + MOK 簽名）。Ubuntu 有以更新 linux-firmware 板檔救之之案，惟本疾非板檔之咎（本機 `board-2.bin` md5 `4b0cb2416db49e39c8c7e3302067d22e` 已屬新版而病如故）。
- 內核升級後，本地模組僅繫於舊版 modules 目錄，須按新核重編重簽，或待上游修復。
