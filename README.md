# Pixel 9a (LineageOS 23) 字體美化指南：上古黑體 (Shanggu Sans)

這份文檔記錄了如何在 **Pixel 9a (tegu)** 從刷機、Root 到字體美化的全過程：**LineageOS 23 (Android 16)** 正確 Root 並替換系統字體為 **[上古黑體 (Shanggu Sans)](https://github.com/GuiWonder/Shanggu)**。

由於 Android 16 的分區變動和字體渲染引擎更新，傳統的 Root 和換字體方法會導致 **Bootloop (卡開機)**。本文包含了詳細的避坑指南和救磚方案。

## ⚠️ 核心摘要 (TL;DR)

1. **Root 坑**：Pixel 9a 的 Ramdisk 在 `init_boot` 分區，**不要** Patch `boot.img`，否則無效或軟磚。
2. **字體坑**：Android 16 強制要求 Variable Font (VF)。CFI 模組需要關閉自動修正 (`Correction=0`) 才能正常開機。
3. **救命指令**：卡 Logo 時使用 `adb wait-for-device shell magisk --remove-modules`。

---
## Part 0: LineageOS 安裝與解鎖陷阱

請參考 [LineageOS 官方指南](https://wiki.lineageos.org/devices/tegu/) 進行安裝，但有一個**官方沒寫清楚的巨大心態考驗**：

### ⚠️ 關於 `fastboot flashing unlock`
當你執行完解鎖 Bootloader 指令並在手機上確認後，手機會重啟並強制清除數據 (Wipe Data)。

* **現象**：手機會**一直卡在 Google Logo**，無法進入系統，也無法再次啟用 ADB。
* **對策**：**不要慌！這是正常現象。**
    * `fastboot flashing unlock` 解鎖後無法進入原廠系統沒關係。
    * 直接長按 `電源` + `音量下` 強制進入 Fastboot Mode。
    * 繼續按照官方教程刷入 Recovery (`fastboot flash boot ...` 等)，完全不影響後續刷機步驟。

---

## Part 1: 正確的 Root 方式 (Magisk)

### ❌ 錯誤示範 (我踩過的坑)

一開始我按照舊習慣，提取了 LineageOS 的 `boot.img`，用 Magisk App 修補後刷入 (`fastboot flash boot ...`)。

* **後果**：手機顯示 **沒有 Root**，或者直接 **Boot Failure**。
* **原因**：Pixel 9 系列將 Ramdisk 移動到了 `init_boot.img`，單純動 `boot.img` 只會破壞核心簽名。

### ✅ 正確步驟

在 LineageOS Recovery 下直接 Sideload Magisk 安裝包，讓腳本自動識別分區。

1. 下載 Magisk APK，將副檔名改為 `.zip` (例：`Magisk-v28.0.zip`)。
2. 重啟進入 Lineage Recovery。
3. 選擇 `Apply Update` -> `Apply from ADB`。
4. 執行：
```bash
adb sideload Magisk-v28.0.zip
```


5. 觀察終端機輸出，確認出現 `- Patching init_boot image` 字樣。
6. 重啟系統，安裝 Magisk App 並修復環境。

### 🚑 救磚方法 (如果你已經刷壞了 boot)

如果不幸刷壞了 `boot.img` 導致無法開機：

1. 進入 Fastboot Mode。
2. 刷回原廠核心文件：
```bash
fastboot flash boot boot.img
fastboot flash dtbo dtbo.img
fastboot flash vendor_kernel_boot vendor_kernel_boot.img
```

---

## Part 2: 字體替換 (上古黑體 + CFI)

我們使用 **Custom Font Installer (CFI)** 模組。由於 Android 16 對 `fonts.xml` 結構極其敏感，必須使用 **Variable Font (可變字體)** 版本並手動修改配置。

### 1. 準備字體文件

我們選用 **[上古黑體 (Shanggu Sans)](https://github.com/GuiWonder/Shanggu)** 的可變字體版本，以獲得舊字形 (Old Glyph) 的顯示效果。

* **下載源**：[Shanggu Sans Releases](https://github.com/GuiWonder/Shanggu/releases)
* **文件準備**：
* `ShangguSans-VF.ttf` (黑體，用於介面主體)
* `ShangguSerif-VF.ttf` (明體，用於搭配/斜體位)


### 2. 部署文件 (ADB Push)

在手機中建立 `/sdcard/OhMyFont/CFI/` 目錄。我們將黑體 (Sans) 映射為主字體，將明體 (Serif) 映射為斜體位 (這是一種個人風格選擇)。

執行以下命令：

```bash
# 建立目錄
adb shell mkdir -p /sdcard/OhMyFont/CFI

# 1. 部署主體 (Sans-Serif) -> ss.ttf
adb push ShangguSans-VF_TTFs/ShangguSans-VF.ttf /sdcard/OhMyFont/CFI/ss.ttf

# 2. 部署等寬字體 (Monospace) -> ms.ttf
# 這裡我們也用上古黑體覆蓋，保持風格統一
adb push ShangguSans-VF_TTFs/ShangguSans-VF.ttf /sdcard/OhMyFont/CFI/ms.ttf

# 3. 部署斜體 (Italic) -> ssi.ttf
# 這裡我選擇用上古明體 (Serif) 來作為斜體顯示，創造獨特視覺效果
adb push ShangguSerif-VF_TTFs/ShangguSerif-VF.ttf /sdcard/OhMyFont/CFI/ssi.ttf

# 4. 部署等寬斜體 -> msi.ttf
adb push ShangguSerif-VF_TTFs/ShangguSerif-VF.ttf /sdcard/OhMyFont/CFI/msi.ttf
```

### 3. 配置 Config (關鍵防磚設定 ⚠️)

默認的 CFI 配置在 Android 16 上會導致 Bootloop。必須建立一個 `config.cfg` 並推送到 `/sdcard/OhMyFont/`。

**config.cfg 內容：**
參見 [config.cfg 文件](./config.cfg) 的示例。我做了偷懶的設定，將所有字體位都指向了上古黑體的可變字體版本，並且關閉了自動修正功能。

**推送配置：**

```bash
adb push config.cfg /sdcard/OhMyFont/
```

### 4. 刷入模組

1. 進入 Magisk App -> Modules。
2. 選擇 `Install from storage` -> 刷入 CFI 的 ZIP 包。
3. **重啟手機**。
---

## Part 3: 故障排除 (Troubleshooting)

### Q: 刷入後卡在 Google Logo (Bootloop) 怎麼辦？

不要慌，不需要清除數據。這是因為字體 XML 導致 SystemUI 崩潰。

**解決方案：**

1. 保持手機連接電腦。
2. 在 Logo 轉圈時，終端機輸入：
```bash
adb wait-for-device shell magisk --remove-modules
```


3. 手機會自動重啟並移除所有模組，恢復正常。

### Q: 系統字體變了，但小紅書/Firefox 沒變？

這是 App 內置字體策略導致的。

* **解法**：安裝 **LSPosed** (Zygisk) + **AOSPMods**。
* 在 AOSPMods 中強制覆蓋 System UI 和特定 App 的字體設置。

---

## Credits

* **LineageOS**: For the clean Android 16 base.
* **GuiWonder**: For the amazing [Shanggu Sans (上古黑體)](https://github.com/GuiWonder/Shanggu).
* **Magisk**: For systemless root.
* **CFI**: For the Magisk module that enables custom fonts.