---
name: hermes-backup
description: 將 Hermes Agent 的所有資料（skills、config、cron、memories）壓縮。**重要**：Cron delivery 只發文字報告，備份檔需手動發送給使用者。
category: productivity
---

# Hermes Backup

將 Hermes Agent 備份到一個 zip 檔案，並透過 WhatsApp 發送給使用者。

## 備份內容

| 路徑 | 內容 |
|------|------|
| `~/.hermes/skills/` | 所有技能文件 |
| `~/.hermes/scripts/` | 自訂腳本 |
| `~/.hermes/config.yaml` | 設定檔 |
| `~/.hermes/cron/` | Cron job 資料 |
| `~/.hermes/memories/` | 長期記憶 |

## 執行步驟

### 1. 執行備份（排除 cron output）

**重要：zip `-x` pattern 只排除檔案，無法排除目錄 entry。** 不能用 `-x ".hermes/cron/output/*"` 來排除 output 目錄，因為 `.hermes/cron` 作為來源目錄會把 output 目錄 entry 直接加進 zip。

正確做法：明確指定 cron 目錄內要備份的子項目，不包含 output。

```bash
cd ~
rm -f hermes_backup.zip
zip -r hermes_backup.zip \
  .hermes/skills \
  .hermes/config.yaml \
  .hermes/cron/jobs.json \
  .hermes/memories \
  -x ".hermes/memories/*.lock"

# 空 lock 檔（.tick.lock, MEMORY.md.lock, USER.md.lock）不備份，沒有任何價值
# 注意：skills/.hub/lock.json 有內容，保留
```

這樣 `.hermes/cron/output` 整個目錄完全不會被加入 zip。

### 2. 檢查檔案大小

```bash
ls -lh hermes_backup.zip
```

### 3. 透過 WhatsApp 發送

使用 `MEDIA:/path/to/hermes_backup.zip` 格式發送檔案。

## 自動排程限制

**重要：Cron delivery 只發文字報告，不發附檔。** 備份 ZIP 檔案會生成在 `~/hermes_backup.zip`，但自動 delivery 只送文字到 WhatsApp，不含檔案。

因此排程備份需要手動發送：
- Cron 跑完後，在對話裡手動用 `MEDIA:/Users/johnchen/hermes_backup.zip` 發給使用者
- 或手動觸發 `hermes-backup` skill

## 遷移到新電腦

1. 將 `hermes_backup.zip` 複製到新電腦的 `~/` 目錄
2. 解壓縮：
   ```bash
   cd ~ && unzip -o hermes_backup.zip
   ```
3. 重啟 Hermes Agent
4. Cron jobs、技能、記憶都會完整保留

## 注意事項

- **memories 目錄名稱**：是 `memories`（複數，有 s），不是 `memory`。之前所有備份都因為寫成 `memory` 而跳過了真正的記憶檔案
- **Cron delivery 只發文字**：自動 delivery 送到 WhatsApp 的只有文字報告，**不會**自動附加 zip 檔案。備份跑完後需要在對話裡手動傳送 `MEDIA:~/hermes_backup.zip`
- **音訊快取** (`~/.hermes/audio_cache/`) 不包含在備份中，可省略
