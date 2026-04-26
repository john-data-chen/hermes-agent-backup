Project-specific Docker isolation setup for open333crm: docker-compose.dev.yml for read-write mode, docker-sh for rw, docker-sh.ro for readonly. Named volumes for node_modules. Do NOT modify original docker-compose.yml.
§
Concurrent agents modifying same files cause race conditions — files get overwritten by sibling agents mid-session. Use python3 for atomic writes instead of patch tool when sibling agents are active.
§
turborepo-starter-kit workspace update workflow: use `pnpm --filter apps/api --filter apps/web --filter packages/ui update --latest` to update specific workspaces from root instead of cd into each
§
Hermes User Profile/Memory 儲存在 `~/.hermes/memories/` (有 s)，不是 `~/.hermes/memory`。備份時路徑要正確。
§
Hermes Backup 注意事項：備份路徑是 `.hermes/memories/`（有 s），不是 `memory`。Cron job 的 `deliver: origin` 只發文字報告，不發附檔。備份跑完後需手動透過 WhatsApp 發送 ~/hermes_backup.zip 給約翰。
§
config.yaml 已驗證無 API keys 或 tokens，只有 WHATSAPP_HOME_CHANNEL (只是 chat ID，無安全風險)
§
約翰喜歡公開分享他的 agent workflow，但會先手動檢查確認無敏感資訊洩漏