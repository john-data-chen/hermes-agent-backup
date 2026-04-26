---
name: npm-package-update-workflow
description: 更新 npm packages 到最新版並驗證（lint/test/build），自動建立 commit 和 PR
category: software-development
---

# NPM Package Update Workflow

將專案的所有 npm packages 更新到最新版，執行完整驗證流程，最後 commit 並建立 PR。

## 支援的專案

| 專案 | 路徑 | 類型 | Cron Job ID |
|------|------|------|-------------|
| next-dnd-starter-kit | ~/projects/next-dnd-starter-kit/ | 單一 Next.js | 6612902dca66 |
| turborepo-starter-kit | ~/projects/turborepo-starter-kit/ | Turborepo monorepo (api/web/ui) | bfddf898b8cf |

## 適用場景
- 定期維護專案依賴
- 確保 dependencies 保持最新

## 前置需求
- Node.js 24+ (pnpm)
- gh CLI（用於自動建立 PR，如未安裝則需手動建立）
- 必要時需要 MongoDB 執行測試

## 關鍵技術發現（實驗證明）

**`pnpm update --latest` 無視所有 semver 約束。**
- `^24.12.2`、`~24.12.2`、`pnpm.overrides`、`catalog` 中的版本，全部無效
- `--latest` 模式會直接升級到 registry 上的最新版本，包括跨 major 版本
- 這是 pnpm 的設計行為，非 bug
- 解決方案：手動還原 pnpm-workspace.yaml 中 @types/node 版本後執行 `pnpm install`

## 工作流程

### 1. 切換到專案目錄
```bash
cd ~/projects/{project-name}
```

### 2. 環境準備
- 單一專案（如 next-dnd-starter-kit）：根目錄的 `.env`, `.env.test`
- Monorepo（如 turborepo-starter-kit）：各 app 的 `.env`
- 檢查並從 `env.example` 建立必要的環境變數檔案
- 生成新的 secret（如果 AUTH_SECRET 或 JWT_SECRET 是 placeholder）

### 3. 更新 Packages

**單一專案（next-dnd-starter-kit）和 Turborepo 根目錄：**
```bash
pnpm update --latest
```

**Turborepo monorepo 專用（api/web/ui/mobile）：**
- 更新 `apps/api`、`apps/web`、`packages/ui` 三個 workspace：
  ```bash
  pnpm --filter turborepo-starter-kit-api --filter turborepo-starter-kit-web --filter @repo/ui update --latest
  ```
- 更新 mobile app：
  ```bash
  pnpm mobile:expo:upgrade
  ```

> 注意：`pnpm update --latest` 在根目錄執行時只會更新根目錄的直接 dependencies，不會更新 workspace 子目錄的 packages。用 `--filter` 指定目標 workspace 才能正確更新。

### 4. 修正 @types/node 版本（防止升級到 25.x）
`pnpm update --latest` 會無視 semver 約束將 `@types/node` 升級到 25.x。還原方式：

```bash
# 回復 pnpm-workspace.yaml catalog 中的 @types/node 版本
# 注意：macOS 的 sed -i 對複雜字元會失敗，使用 Python 較可靠
python3 -c "
import re
with open('pnpm-workspace.yaml', 'r') as f:
    content = f.read()
# 如果 @types/node 版本不是 24.12.2 就改回 24.12.2
content = re.sub(r'(\"@types/node\":\\s*\")[^\"]+(\")', r'\g<1>24.12.2\g<2>', content)
with open('pnpm-workspace.yaml', 'w') as f:
    f.write(content)
"

# 確認 package.json 保持 catalog: 引用（不應被修改）
# \"@types/node\": \"catalog:\"

# 砍掉 lockfile 並重新執行 pnpm install 讓 workspace 解析並鎖定正確版本
rm pnpm-lock.yaml
pnpm install
```

> **單一專案（如 next-dnd-starter-kit）** 不需要這個流程，直接用 `pnpm add -D @types/node@24.12.2` 即可。

### 4b. 修正 catalog 版本（砍 lockfile 法）

有時候只改 `pnpm-workspace.yaml` 再執行 `pnpm install`，pnpm 會說 "Lockfile is up to date" 不予更新。這是因為 lockfile 已經是一致的，什麼都沒變。這種情況必須砍掉 lockfile 讓它重新產生：

```bash
rm pnpm-lock.yaml
pnpm install
```

驗證 lockfile 中 `@types/node` 的 specifier 是否為正確版本。

> **為什麼不用 `pnpm install --force`**：pnpm 的 `--force` 在 lockfile 已解析過的情況下仍可能跳過更新。砍掉 lockfile 是最乾淨的方式。

### 4c. Mobile app 更新（turborepo-starter-kit 專用）

Mobile app 使用 `pnpm mobile:expo:upgrade` 更新（這會更新 apps/mobile）：
```bash
pnpm mobile:expo:upgrade
```
這是 turborepo 的特殊流程：
- 使用 `turbo run expo:upgrade --filter=mobile` 透過 turborepo 執行
- 會呼叫 `npx expo install expo@latest && npx expo install --fix`
- Expo SDK 會自動調整 react/react-native 等版本到相容版本
- 執行後需檢查 peer dependency 警告，並確保 lockfile 有更新

### 5. 檢查變更並通知
```bash
git status
```
- 如果有變更：執行驗證流程
- 如果沒有變更：**一定通知使用者**「✅ [專案名] 檢查完畢，無新 package 更新」，不得靜默結束

### 6. 驗證流程
```bash
# 單一專案
pnpm lint && pnpm test && pnpm build

# Turborepo monorepo
pnpm turbo run lint && pnpm turbo run test && pnpm turbo run build
```

### 7. Commit 並 Push
```bash
git checkout -b feat/john/update-packages-$(date +%Y%m%d)

# 單一專案
git add package.json pnpm-lock.yaml

# Turborepo monorepo（需含 pnpm-workspace.yaml）
git add package.json pnpm-lock.yaml pnpm-workspace.yaml

git commit -m "chore(deps): update all packages to latest versions"
git push -u origin HEAD
```

### 8. 建立 PR
```bash
gh pr create \
  --title "chore(deps): update all packages to latest versions" \
  --body "## Summary
Updates all npm packages to their latest versions.

## Testing
- \`pnpm lint\` - passed
- \`pnpm test\` - passed
- \`pnpm build\` - successful"
```

## 各專案設定

### next-dnd-starter-kit
- **路徑**: ~/projects/next-dnd-starter-kit/
- **類型**: 單一 Next.js 專案
- **環境檔案**: `.env`, `.env.test` (root)
- **Secret**: `AUTH_SECRET`
- **驗證**: `pnpm lint && pnpm test && pnpm build`
- **Cron**: 每週六 06:50

### turborepo-starter-kit
- **路徑**: ~/projects/turborepo-starter-kit/
- **類型**: Turborepo monorepo
- **環境檔案**: `apps/api/.env`, `apps/web/.env`
- **Secret**: `JWT_SECRET` (in apps/api/.env)
- **驗證**: `pnpm turbo run lint && pnpm turbo run test && pnpm turbo run build`
- **Cron**: 每週六 06:55
- **特殊**: mobile app 有 Expo 更新，需執行 `pnpm mobile:expo:upgrade`

## 關鍵發現：pnpm update --latest 的行為

**`pnpm update --latest` 無視所有 semver 約束**，包括：
- `^`、`~` 版本的 semver 範圍
- `pnpm.overrides`
- `pnpm-workspace.yaml` catalog 中的版本

**實測結果：**
- `"@types/node": "^24.12.2"` → `pnpm update --latest` → 升級到 25.6.0
- `"@types/node": "~24.12.2"` → `pnpm update --latest` → 升級到 25.6.0
- `pnpm.overrides: "@types/node": "^24.12.2"` → `pnpm update --latest` → 升級到 25.6.0
- catalog 中 `"@types/node": "24.12.2"` → `pnpm update --latest` → 升級到 25.6.0
- **root `package.json` 的 `devDependencies.@types/node` 無視 `-D` 需用 `-wD`**

**對策：** 每次 `pnpm update --latest` 後執行以下命令還原版本（sed 在 macOS 可能需要不同語法，建議用 python3）：

## 疑難排解

### 驗證失敗時：確認是否為既有问题
```bash
git stash  # 暫存更新變更
pnpm turbo run test  # 只跑 test，避免 build 失敗干擾判斷
# 觀察結果：
#   - 如果 test 仍然失敗 → 既有问题，恢復變更後仍可 commit+PR
#   - 如果 test 通過 → 本次更新引發的問題，需進一步調查
git stash pop  # 恢復變更
```

### Vercel build 失敗：ERR_PNPM_LOCKFILE_CONFIG_MISMATCH
如果 Vercel 報這個錯誤，表示 lockfile 和 workspace 配置不同步。
**解決方法：**
```bash
rm pnpm-lock.yaml && pnpm install
git add pnpm-lock.yaml && git commit -m "chore: regenerate lockfile" && git push
```
然後在 Vercel dashboard redeploy。

## 注意事項
- `.env` 和 `.env.test` 包含敏感資訊，已在 `.gitignore` 中
- `AUTH_SECRET` / `JWT_SECRET` 必須是有效的 random string（使用 `openssl rand -base64 32` 生成）
- Turborepo 使用 `pnpm overrides` 固定特定版本（如 React），`pnpm update --latest` 不會覆蓋這些
- 如果 gh CLI 未安裝，需要手動建立 PR（瀏覽 `https://github.com/{owner}/{repo}/compare/{branch}` 建立）

## 驗證成功標準
- lint: 0 warnings, 0 errors
- test: all tests passed
- build: successful

**注意：** turborepo-starter-kit 的 web tests（rolldown JSX 解析錯誤）和 mobile build（global.css 匯入問題）是**既有的問題**，更新前就存在。驗證失敗時需先用 `git stash && pnpm turbo run test` 確認是否為既有问题。如下一節的「驗證失敗時」流程。

## 管理 Cron Jobs
```bash
# 列出所有 jobs
cronjob(action='list')

# 暫停 job
cronjob(action='pause', job_id='...')

# 恢復 job
cronjob(action='resume', job_id='...')

# 刪除 job
cronjob(action='remove', job_id='...')
```
