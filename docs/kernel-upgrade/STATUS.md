# H2+ 核心升級進度

更新：2026-10-21（Asia/Taipei）。

## 目標

在 Orange Pi Zero H2+ 上升級 Linux 核心，同時保留 miZy、XR819 Wi-Fi 和
HTTPS CONNECT 代理。不是改裝完整 Armbian 系統。尚未設定 Tailscale。

## 目前裝置狀態

- 仍執行 Linux `3.4.113+`，2019-05-16 建置，ARMv7。
- miZy userspace 基於 OpenWrt Designated Driver 50015，BusyBox 1.24.2。
- U-Boot 2017.01-hyphop miZy，UART 為 115200、8N1、無流量控制。
- U-Boot 實際辨識 `Xunlong Orange Pi Zero`，512 MiB RAM。
- 晶片偵測為 H2+；舊 FEX 內的 `MX3 h3` 字串不能用來判定板型。
- SPI 為 16 MiB，目前沒有偵測到已插入的 TF 卡。
- 本次核心升級尚未改寫 SPI。先前的 Wi-Fi、DHCP 修正與代理已保存至 SPI。
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

- 現有 kernel 分割區只有 3 MiB；Armbian 現成 zImage 約 10.1 MiB，不能直接替換。
- 完整 Armbian 映像約 1.5 GB，不能放進 16 MiB SPI；它只作為驅動與核心方案
  的參考，不能視為已完成 miZy 升級。
- 舊核心模組不能跨版本沿用。XR819、mac80211、cfg80211、韌體、裝置樹和
  miZy 的 Wi-Fi 初始化流程都需要一起處理。
- 現有 FEX 不是新版核心使用的 DTB；不能只更換 kernel 檔案。
- 沒有新核心成功啟動或新版 Wi-Fi 成功連線的證據。

## 下一步

- [ ] 從 miZy 核心 fork 建立可重現的新核心建置流程，選擇維護中的版本。
- [ ] 配好 Orange Pi Zero H2+ DTB、UART、SPI、initrd、TUN 和 XR819 驅動。
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
