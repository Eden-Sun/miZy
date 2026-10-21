# H2+ 核心升級進度

更新：2026-10-21（Asia/Taipei）。

## 最新 XR819 調整

2026-10-21 已保存新版 Wi-Fi 韌體 `.52.58`、上游 HT RX 格式修正 #34，以及
IRQ/輪詢競爭診斷與警告分類（只以實際 IRQ 計數變化判定，不把 TX 喚醒當 IRQ）。rootfs 寫入讀回驗證通過，其他五個分割區雜湊
不變；自動重啟已確認載入新韌體/模組並恢復原 Wi-Fi/IP。核心仍為 6.12.113。

雙向各 300 次 ping 的短樣本中，接收端從原韌體/GPIO 的 284/300 改為
新韌體/#34/GPIO 的 298/300；順序測試不能保證是因果，亦非完全無丟包。
重負載加 Flash 備份時仍有丟包及一次 TLS deadline 逾時；事後 20 次 HTTPS
代理重測全數成功。原生 SDIO 與停用 HT 均未證明能根治，所以沒有部署。
#31 也未啟用，舊版本完整備份保留。詳細數據見 kernel/mainline/XR819-IRQ.md。

## 結果

Orange Pi Zero H2+ 已從 Linux 3.4.113+ 升級至 **6.12.113-mizy-h2plus**，
寫入 16 MiB SPI，保留 miZy 使用者環境、XR819 Wi-Fi、HTTPS CONNECT 代理
及反向 SSH。沒有安裝完整 Armbian，也尚未安裝 Tailscale。

最終修正後已完成不需手動介入的軟體重啟測試：

- `uname -r`：`6.12.113-mizy-h2plus`，四核心、512 MiB RAM。
- XR819 自動連回 保存的 SSID，WPA2-PSK/CCMP，狀態 `COMPLETED`。
- 原 Wi-Fi MAC 和 DHCP 位址保留；DNS 可解析外網。
- `/dev/net/tun` 存在。
- 反向 SSH 自動重連，經 VPS 代理實測 `CONNECT=200 HTTPS=200`。
- kernel、rootfs、DTB、boot script 寫入後均通過 `mtd verify`；最後補寫
  rootfs 與 script 後再度驗證成功。U-Boot 與 user 分割區 SHA-256 維持一致。
- 最終修正後尚未另外要求實體斷電冷開機測試；以上為自動軟體重啟的結果。

## SPI 配置

| 分割區 | 起點 | 大小 |
|---|---|---|
| uboot | `0x000000` | 512 KiB |
| script | `0x080000` | 64 KiB |
| dtb | `0x090000` | 64 KiB |
| kernel | `0x0a0000` | 4 MiB |
| initrd | `0x4a0000` | `0x9e0000` bytes |
| user | `0xe80000` | 1.5 MiB |

uImage 為 3,824,003 bytes；miZy uInitrd 為 7,352,384 bytes。
UART 保持 115200/8N1，U-Boot 保持 2017.01-hyphop，沒有偵測到 TF 卡。

## 必要相容修正

- Linux 6.12.113 與 fifteenhex/xradio 固定來源、雜湊驗證及交叉編譯。
- JFFS2 啟用 LZO，SPI 使用原本 64 KiB 擦除區塊。
- 原設定在掛載 OverlayFS 前匯入 upper，使用獨立 workdir。
- Legacy squashfs initrd 使用 chroot 交接；RAM cpio bootstrap 使用 switch_root。
- 暫存 user 掛載點放在可寫的 `/mnt/user`。
- 停用舊 Wi-Fi hotplug 重啟流程，明確載入 WPA2 加密模組。
- DTB 預先配置裝置原 MAC，防止舊 U-Boot 用 ethernet1 alias 蓋掉。
- `initrd_high=0xffffffff` 保留 rootfs 在安全的 `0x44000000`，避免舊 U-Boot
  把 7 MiB rootfs 搬到 `0x41000000` 以下而空間不足。
- 停用舊 CPU/DRAM 暫存器直接寫入，改由 mainline 驅動管理。

## 備份、紀錄及重建

最新升級前完整 SPI 備份 SHA-256：
`67df5fe78919934b2a50008307a90666a6f18700c7c806bf70067685bb47f213`。

本機忽略的 `build/artifacts/` 保存升級前後備份、候選映像及實機記錄：
`spi-autoboot-final.log`、`flash-final-fix.log`。備份包含憑證與私鑰，不可上傳 git。
舊版備份另在同層 `artifacts/mizy-backup-20261021/`。

核心來源及建置流程位於 kernel 子模組的 `mainline/`，分支為
`upgrade/h2plus-linux-6.12`。詳細流程見該目錄 README。

先前 legacy kexec 未能啟動新核心；實機驗證採 U-Boot legacy uImage/uInitrd，
原 U-Boot 不支援 FIT。Armbian 只曾作為驅動方案參考。

## XR819 穩定性查核

2026-10-21 查核：現用 xradio commit 是上游最新合併版本，與 Armbian 相同。
運作約 37 分鐘仍保持連線，但日誌累計 67 次 missed interrupt。20 次 gateway
ping 無丟包，不能據此宣稱長期穩定。未合併的 ARP、HT 接收和韌體修正
尚未安裝；詳見 kernel/mainline/XR819-STATUS.md。


## XR819 修正實測

PR #31（ARP cipher）及 #34（HT RX metadata）已編譯並透過串口在 RAM 載入測試。
未寫入 Flash。短測各版本的代理均正常；加載 16 MiB 雙向 SSH 傳輸及 20 次
HTTPS 請求的兩分鐘測試中，修正版 ping 遺失 7/120，原版對照遺失 2/120。
資料 SHA-256 全部一致，兩版本 HTTPS 都為 20/20 成功。無法證明穩定性改善，
因此目前回到原版驅動，沒有永久套用。詳細數據與限制見 kernel/mainline/XR819-STATUS.md。
