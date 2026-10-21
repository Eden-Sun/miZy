# H2+ 核心升級進度

更新：2026-10-21（Asia/Taipei）。

## 最新執行進度

Linux **6.12.113-mizy-h2plus** 已在 H2+ 實機從 RAM 啟動，四核心、512 MiB
RAM、miZy procd、原 JFFS2 設定匯入、XR819 WPA2 連線、DHCP、DNS 均已驗證。
經 VPS 反向 SSH 的 HTTPS 代理實測 `CONNECT=200 HTTPS=200`。

正式映像已寫入 SPI，kernel、initrd、DTB、boot script 均通過 `mtd verify`；
U-Boot 與 user 分割區寫入前後雜湊一致。**正式開機尚未完成**：舊 U-Boot
將 initrd 限制搬到 `0x41000000` 以下，新核心留下的低位址空間不足，停在
`Failed to allocate 0x703000 bytes below 0x41000000`。尚未進入新核心，
需要板子斷電重上電，攔截 U-Boot 後用 `setenv initrd_high 0xffffffff`
保留 initrd 在原本安全的 `0x44000000` 載入位址，再手動 bootm。
修正已加入本機 assembler；**SPI 內開機腳本尚未補上此修正**。

- 新核心 uImage：3,824,003 bytes；miZy uInitrd：7,352,384 bytes。
- SPI 已改為 4 MiB kernel，initrd 從 `0x4a0000` 開始，user 仍在 `0xe80000`。
- 最新完整備份 SHA-256：
  `67df5fe78919934b2a50008307a90666a6f18700c7c806bf70067685bb47f213`。
- 保留原 miZy userspace，沒有安裝完整 Armbian。

## 目標

在 Orange Pi Zero H2+ 上升級 Linux 核心，同時保留 miZy、XR819 Wi-Fi 和
HTTPS CONNECT 代理。不是改裝完整 Armbian 系統。尚未設定 Tailscale。

遠端代理已改採 [反向 SSH 隧道](../remote-proxy.md)，不再以升級核心作為
遠端使用代理的前提；此核心升級專案尚未完成。

## 目前裝置狀態

- SPI 已寫入 Linux 6.12.113；目前停在 U-Boot，等待斷電重啟修正。
- miZy userspace 基於 OpenWrt Designated Driver 50015，BusyBox 1.24.2。
- U-Boot 2017.01-hyphop miZy，UART 為 115200、8N1、無流量控制。
- U-Boot 實際辨識 `Xunlong Orange Pi Zero`，512 MiB RAM。
- 晶片偵測為 H2+；舊 FEX 內的 `MX3 h3` 字串不能用來判定板型。
- SPI 為 16 MiB，目前沒有偵測到已插入的 TF 卡。
- 本次核心升級已改寫 SPI，正式開機目前停在 U-Boot initrd 搬移配置錯誤。
- 2026-10-21 最後一次驗證：原系統 Wi-Fi 恢復連線，經代理測得
  `CONNECT=200 HTTPS=200`。

## 已完成

1. 備份完整 16 MiB SPI，並核對裝置與電腦上的 SHA-256 一致。
2. 保存舊 FEX，讀出開機腳本與 SPI 配置。
3. 驗證舊核心啟用 `CONFIG_KEXEC=y`，且有 `/dev/net/tun`。
4. 核對官方 OpenWrt 25.12.1 Orange Pi Zero initramfs FIT 映像，核心為
   Linux 6.12.74。
5. `kexec -l` 成功，`kexec -e` 停在 `Starting new kernel`，沒有新核心輸出。
   斷電重開後恢復原系統；不能把「成功載入」記成「成功開機」。
6. 在 U-Boot 攔截開機並確認板型、RAM 與 `loady`、`bootz` 支援。
7. 嘗試 YMODEM 傳送官方 FIT；因該映像未包含 XR819 驅動，而需求要求保留
   Wi-Fi，已取消傳輸，沒有啟動或刷寫該映像。
8. 下載並核對 Armbian community 26.11.0-trunk.104 Orange Pi Zero minimal
   映像；確認其中有 Linux 6.18.57 對應的 `xradio_wlan.ko`。
   尚未在本機板子上驗證該核心或 Wi-Fi 驅動。

## 已知限制

- 現有 kernel 分割區只有 3 MiB；新編譯 uImage 約 3.64 MiB，需要在通過
  RAM 測試後調整為 4 MiB。Armbian 現成 zImage 約 10.1 MiB，不能直接替換。
- 完整 Armbian 映像約 1.5 GB，不能放進 16 MiB SPI；它只作為驅動與核心方案
  的參考，不能視為已完成 miZy 升級。
- 舊核心模組不能跨版本沿用。XR819、mac80211、cfg80211、韌體、裝置樹和
  miZy 的 Wi-Fi 初始化流程都需要一起處理。
- 現有 FEX 不是新版核心使用的 DTB；不能只更換 kernel 檔案。
- RAM 模式下核心、Wi-Fi、代理和反向 SSH 已驗證；正式 SPI 開機仍待修正。

## 下一步

- [x] 從 miZy 核心 fork 建立固定來源版本的建置流程，編譯 Linux 6.12.113。
- [x] 編譯 Orange Pi Zero H2+ DTB、UART、SPI、initrd、TUN 和 XR819 驅動。
- [ ] 檢查 miZy rootfs/init 與新核心的相容性，計算核心、模組和 rootfs 的
      實際壓縮大小，再決定 16 MiB SPI 配置。
- [ ] 先從 U-Boot/RAM 或 TF 卡測試，不以已失敗的 legacy `kexec` 切換作為
      唯一開機方法。
- [ ] 實機驗證核心版本、Wi-Fi 驗證、DHCP、DNS、外網與 HTTPS CONNECT。
- [ ] 通過後製作可復原的 SPI 升級映像，刷寫並驗證冷開機。

## 本機資料

本機工作目錄：`/Users/m1pro/project/llm/mizy-kernel-upgrade`。

原始備份保存在同層之外的
`/Users/m1pro/project/llm/artifacts/mizy-backup-20261021/`；候選映像在
`/Users/m1pro/project/llm/artifacts/mizy-kernel-upgrade/`。這些檔案可能包含
Wi-Fi 設定或裝置金鑰，未納入 git。

## 上游與候選來源

- [miZy](https://github.com/hyphop/miZy)
- [miZy kernel](https://github.com/hyphop/miZy-linux-kernel)
- [OpenWrt 25.12.1 sunxi/cortexa7](https://downloads.openwrt.org/releases/25.12.1/targets/sunxi/cortexa7/)
- [Armbian Orange Pi Zero board config](https://github.com/armbian/build/blob/main/config/boards/orangepizero.csc)
- [Armbian community candidate](https://github.com/armbian/community/releases/tag/26.11.0-trunk.104)
