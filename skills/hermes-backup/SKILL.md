---
name: hermes-backup
description: 將 Hermes Agent 的所有資料（skills、scripts、config、cron、memories）壓縮並透過 WhatsApp 發送給使用者
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

### 1. 執行備份指令

```bash
cd ~ && zip -r hermes_backup.zip .hermes/skills .hermes/config.yaml .hermes/cron .hermes/memories
```

排除不必要的目錄（如 `scripts` 如果是空的、`audio_cache` 等）：
```bash
cd ~ && zip -r hermes_backup.zip .hermes/skills .hermes/config.yaml .hermes/cron .hermes/memories
```

### 2. 檢查檔案大小

```bash
ls -lh hermes_backup.zip
```

### 3. 透過 WhatsApp 發送

使用 `MEDIA:/path/to/hermes_backup.zip` 格式發送檔案。

## 遷移到新電腦

1. 將 `hermes_backup.zip` 複製到新電腦的 `~/` 目錄
2. 解壓縮：
   ```bash
   cd ~ && unzip -o hermes_backup.zip
   ```
3. 重啟 Hermes Agent
4. Cron jobs、技能、記憶都會完整保留

## 注意事項

- **音訊快取** (`~/.hermes/audio_cache/`) 不包含在備份中，可省略
- 如果 `scripts` 或 `memories` 目錄不存在，zip 可能會發出警告但仍正常運作
- 備份檔案建議妥善保管，含有敏感設定資訊
