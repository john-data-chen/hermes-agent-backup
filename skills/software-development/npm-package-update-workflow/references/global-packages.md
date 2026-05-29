# Global npm Package Management

## npm approve-scripts 與全域套件

**`npm approve-scripts` 不支援 `-g` flag**，對全域安裝的套件直接執行會失敗：
```
npm error code EGLOBAL
npm error `npm approve-scripts` does not work for global installs
```

## 繞過方式

當全域套件（如 `opencode-ai`）有 postinstall 腳本需要核准時：

```bash
# 1. 找到 npm 全域 prefix
npm prefix -g
# -> /Users/johnchen/.local/share/fnm/node-versions/v24.16.0/installation

# 2. 在 prefix 目錄建立 package.json（第一次才需要）
cd $(npm prefix -g)
npm init -y

# 3. 建立 symlink 指向實際的 node_modules（第一次才需要）
ln -sf lib/node_modules node_modules

# 4. 執行核准
npm approve-scripts <package-name>

# 5. 清理暫存 symlink
rm node_modules
```

## 核准結果儲存位置

`npm approve-scripts` 將核准記錄寫入 prefix 目錄的 `package.json`：
```json
"allowScripts": {
    "file:../lib/node_modules/<package>": true
}
```

## 注意事項

- 此繞道只在首次核准時需要 symlink；後續同套件升級時，只要 prefix 的 `package.json` 已存在且 `allowScripts` 已記錄，應可直接運行
- 如果 symlink 比 `npm approve-scripts` call 早建好但被其他流程刪掉，需要重建
- 不需要反覆執行 `npm init -y`，它會重設 package.json 的內容
