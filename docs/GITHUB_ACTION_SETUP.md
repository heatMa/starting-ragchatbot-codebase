# Claude GitHub Actions 配置指南

本文档记录如何在 GitHub 仓库中配置 Claude AI 自动回复功能。

## 快速配置（推荐方案）

由于 `anthropics/claude-code-action` 是私有 GitHub App，无法直接安装，我们使用**简化版 workflow**，直接调用 Anthropic API。

### 步骤 1: 配置 API Key

1. 访问 https://console.anthropic.com/settings/keys 获取 API Key
2. 确保账户有可用额度（可在 https://console.anthropic.com/settings/plans 查看）
3. 在仓库中添加 Secret：
   - 打开 https://github.com/{username}/{repo}/settings/secrets/actions
   - 点击 **New repository secret**
   - Name: `ANTHROPIC_API_KEY`
   - Value: 你的 Anthropic API Key (格式: `sk-ant-api03-...`)

### 步骤 2: 创建 Workflow 文件

创建 `.github/workflows/simple-claude.yml`：

```yaml
name: Simple Claude Response

on:
  issue_comment:
    types: [created]
  issues:
    types: [opened]
  pull_request:
    types: [opened]

jobs:
  respond:
    if: contains(github.event.comment.body, '@claude') || contains(github.event.issue.body, '@claude') || contains(github.event.pull_request.body, '@claude')
    runs-on: ubuntu-latest
    permissions:
      issues: write
      pull-requests: write
    steps:
      - name: Call Claude API
        id: claude
        run: |
          PROMPT='${{ github.event.comment.body || github.event.issue.body || github.event.pull_request.body }}'
          ISSUE_NUMBER='${{ github.event.issue.number || github.event.pull_request.number }}'

          # Escape special characters for JSON
          ESCAPED_PROMPT=$(echo "$PROMPT" | sed 's/"/\\"/g' | sed 's/\n/\\n/g' | tr -d '\r')

          RESPONSE=$(curl -s -X POST https://api.anthropic.com/v1/messages \
            -H "Content-Type: application/json" \
            -H "x-api-key: ${{ secrets.ANTHROPIC_API_KEY }}" \
            -H "anthropic-version: 2023-06-01" \
            -d "{
              \"model\": \"claude-3-5-sonnet-20241022\",
              \"max_tokens\": 2048,
              \"messages\": [{\"role\": \"user\", \"content\": \"$ESCAPED_PROMPT\"}]
            }")

          echo "response=$RESPONSE" >> $GITHUB_OUTPUT
          echo "API Response: $RESPONSE"

      - name: Post response as comment
        uses: actions/github-script@v7
        with:
          script: |
            const response = JSON.parse(`${{ steps.claude.outputs.response }}`);
            let reply;

            if (response.error) {
              reply = `**Error:** ${response.error.message}`;
            } else {
              reply = response.content?.[0]?.text || 'Sorry, I could not process your request.';
            }

            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: ${{ github.event.issue.number || github.event.pull_request.number }},
              body: reply
            });
```

### 步骤 3: 提交并测试

```bash
git add .github/workflows/simple-claude.yml
git commit -m "Add Claude AI response workflow"
git push
```

测试方法：
1. 创建一个 Issue 或 PR
2. 在评论或描述中包含 `@claude`
3. 查看 Actions 运行结果：https://github.com/{username}/{repo}/actions

## 常见问题

### 问题 1: `/install-github-app` 命令失败

**错误信息:**
```
Couldn't install GitHub App: Failed to access repository xxx
```

**原因:** `anthropics/claude-code-action` 是私有 GitHub App，不开放公开安装。

**解决方案:** 使用本文档的简化版 workflow，无需安装 GitHub App。

### 问题 2: API Key 余额不足

**错误信息:**
```json
{
  "type": "error",
  "error": {
    "type": "invalid_request_error",
    "message": "Your credit balance is too low to access the Anthropic API."
  }
}
```

**解决方案:**
1. 访问 https://console.anthropic.com/settings/plans
2. 点击 "Add credits" 充值（最低 $5）

### 问题 3: Workflow 没有触发

**检查清单:**
- [ ] 确认文件路径是 `.github/workflows/simple-claude.yml`
- [ ] 确认已推送到正确的分支（通常是 `main`）
- [ ] 确认 Issue/PR 内容包含 `@claude` 关键字
- [ ] 检查 Actions 是否启用：https://github.com/{username}/{repo}/settings/actions

### 问题 4: 权限错误

**错误信息:**
```
Error: Resource not accessible by integration
```

**解决方案:** 确保 workflow 中有正确的权限声明：
```yaml
permissions:
  issues: write
  pull-requests: write
```

## 历史尝试记录（供参考）

以下是我们尝试过的方法，但都因为各种原因失败：

### 尝试 1: 直接安装 GitHub App
```bash
/install-github-app
```
**失败原因:** App 是私有的，无法通过公开渠道安装。

### 尝试 2: 使用 GitHub CLI 扩展
```bash
gh extension install anthropics/gh-claude
```
**失败原因:** 扩展不存在。

### 尝试 3: 手动创建 GitHub App
**方案:** 创建自定义 GitHub App 并配置权限
**放弃原因:** 过于复杂，需要额外的 APP_ID 和 APP_PRIVATE_KEY 配置。

### 尝试 4: 简化版 workflow（当前方案）
**方案:** 直接使用 `curl` 调用 Anthropic API
**优点:** 简单、无需额外配置、直接有效
**缺点:** 功能较基础（不能修改代码、创建 PR 等）

## 高级功能（可选）

如果需要代码修改、自动创建 PR 等高级功能，需要：

1. **创建自定义 GitHub App:**
   - 访问 https://github.com/settings/apps/new
   - 设置权限：Contents (read/write), Issues (read/write), Pull requests (read/write)
   - 生成并下载 Private Key
   - 安装 App 到仓库

2. **添加 Secrets:**
   - `APP_ID`: 你的 App ID
   - `APP_PRIVATE_KEY`: Private Key 内容

3. **使用完整版 workflow:**
   - 参考 https://github.com/anthropics/claude-code-action
   - 配置 `github_token` 使用 App 生成的 token

## 参考链接

- [Anthropic API 文档](https://docs.anthropic.com/en/api/getting-started)
- [GitHub Actions 文档](https://docs.github.com/en/actions)
- [Claude Code Action 仓库](https://github.com/anthropics/claude-code-action)
