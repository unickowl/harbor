# Harbor Custom Subagents

專為 Harbor 多專案架構設計的自定義 AI subagents。

## 可用的 Subagents

### 🔍 cross-project-explorer
追蹤功能在 portal、admin、api 三個專案中的完整實作。

**自動觸發時機**：
- 當你問「這個功能如何運作」
- 當你需要理解端到端流程
- 當你想找出功能相關的所有檔案

**明確使用**：
```
Use the cross-project-explorer to analyze deposit transfer
Have the cross-project-explorer trace customer management
```

---

### 📊 api-impact-analyzer
分析 API 變更會影響哪些前端專案。

**自動觸發時機**：
- 當你提到要修改 API endpoint
- 當你問「這個 API 變更會影響什麼」
- 在規劃 API 更新時

**明確使用**：
```
Use api-impact-analyzer for /api/v2/transfers
Analyze the impact of changing Transfer schema
```

---

### 📋 contract-validator
驗證 API 契約與前端 types 的一致性。

**自動觸發時機**：
- 當你提到類型不匹配的錯誤
- 在 API 變更後
- 當你問「類型定義是否正確」

**明確使用**：
```
Use contract-validator for Transfer types
Validate Quote API contract
```

---

## 目錄結構

```
.claude/agents/
├── cross-project-explorer.md
├── api-impact-analyzer.md
├── contract-validator.md
└── README.md (本檔案)
```

## Subagent vs 主對話

### 使用 Subagent（獨立 context）

- ✅ 任務會產生大量輸出
- ✅ 想保持主對話乾淨
- ✅ 需要限制工具權限
- ✅ 可以平行處理
- ✅ 獨立的研究任務

### 使用主對話

- ✅ 需要頻繁互動
- ✅ 多階段共享重要上下文
- ✅ 快速、小範圍的變更
- ✅ 延遲很重要

---

## 如何運作

### 自動委派

Claude 會根據你的請求自動選擇合適的 subagent：

```
你：「幫我分析 deposit transfer 的完整流程」
     ↓
Claude：「這需要跨專案追蹤，啟動 cross-project-explorer」
     ↓
Subagent：獨立分析 portal、admin、api
     ↓
Claude：整理結果呈現給你
```

### 明確要求

你也可以直接要求特定的 subagent：

```
Use the api-impact-analyzer to check /api/v2/transfers
Have the contract-validator verify Transfer types
```

---

## 實用技巧

### 組合使用

```
# 場景：修改 API Endpoint
1. 先理解現況
   → Use cross-project-explorer for transfer creation

2. 評估影響
   → Use api-impact-analyzer for /api/v2/transfers

3. 驗證類型
   → Use contract-validator for Transfer
```

### 平行處理

```
# 同時研究多個模組
Research authentication, database, and API modules using separate subagents
```

### 恢復 Subagent

Subagent 完成後，如果想繼續它的工作：

```
# 第一次執行
Use cross-project-explorer to analyze authentication

# 稍後繼續
Continue that analysis and now look at authorization
```

---

## Subagent 特性

每個 subagent 都有：

- **獨立 Context Window** - 不會污染主對話
- **唯讀工具** - 只能 Read, Grep, Glob, Bash
- **Sonnet 模型** - 平衡速度與能力
- **專門系統提示** - 針對特定任務優化

---

## 管理 Subagents

### 查看所有 Subagents

在 Claude Code 中執行：

```bash
/agents
```

這會顯示：
- 內建 subagents (Explore, Plan, general-purpose)
- 你的自定義 subagents
- 每個 subagent 的配置

### 編輯 Subagent

直接編輯 markdown 檔案：

```bash
vim /home/ubuntu/Code/owlting/harbor/.claude/agents/cross-project-explorer.md
```

修改後無需重啟，Claude Code 會自動重新載入。

---

## 故障排除

### Subagent 沒有自動觸發

**原因**：描述欄位不夠明確

**解決**：明確要求
```
Use the cross-project-explorer agent to analyze deposit
```

### 輸出太簡略

**原因**：Subagent 可能簡化了輸出

**解決**：要求詳細報告
```
Use cross-project-explorer and provide the complete detailed report
```

### 想在背景執行

**解決**：明確要求
```
Run this analysis in the background using cross-project-explorer
```

或按 **Ctrl+B** 將執行中的任務移到背景。

---

## 技術細節

### Subagent 生命週期

```
1. 主對話收到請求
2. Claude 決定委派給 subagent
3. 啟動獨立的 AI 實例
4. Subagent 執行任務（獨立 context）
5. 返回結果給主對話
6. Claude 整理並呈現
```

### Context 管理

- **主對話**：你和 Claude 的對話
- **Subagent Context**：完全獨立的 context window
- **好處**：詳細輸出不會消耗主對話的 context

### 恢復機制

- 每個 subagent 執行都有獨立的 transcript
- 存放在 `~/.claude/projects/{project}/{sessionId}/subagents/`
- 可以恢復繼續之前的工作

---

## 下一步

### 擴展 Subagents

如果需要新的 subagent：

1. 創建新的 `.md` 檔案
2. 按照現有格式添加 YAML frontmatter
3. 寫清楚的 description（Claude 用來決定何時使用）
4. 定義系統提示

### 共享給團隊

這些 subagents 已經在專案的 `.claude/agents/` 目錄中，可以：

1. 簽入版本控制
2. 團隊成員 clone 後自動可用
3. 協作改進 subagent 提示

---

## 參考資源

- [Claude Code Subagents 官方文件](https://code.claude.com/docs/zh-TW/sub-agents)
- [Harbor 專案架構](../CLAUDE.md)

---

Happy coding with Harbor Subagents! 🚀
