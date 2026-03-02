# GitHub Actions 配置指南

由于子模块 `xcc-src` 是私有仓库，需要配置 Personal Access Token (PAT) 才能让 GitHub Actions 访问。

**推荐使用 Fine-grained tokens**，它提供更精细的权限控制，更加安全。

## 配置步骤（推荐方式）

### 1. 创建 Fine-grained Personal Access Token

1. 访问 GitHub Token 设置: https://github.com/settings/tokens?type=beta
2. 点击 "Generate new token"
3. 配置 Token：
   - **Token name**: `xcc-release-actions`
   - **Expiration**: 选择合适的过期时间（建议 90 天或 1 年）
   - **Description**: `Access private submodule for xcc-release CI/CD`
   
4. **Repository access** 部分：
   - 选择 "Only select repositories"
   - 在下拉列表中选择：
     - ✅ `wshon/xcc-release`
     - ✅ `wshon/xcc-src`

5. **Permissions** 部分（Repository permissions）：
   - **Contents**: `Read-only` ✅
   - **Metadata**: `Read-only` ✅（自动选中）

6. 点击 "Generate token"
7. **立即复制生成的 token**（只显示一次，格式类似 `github_pat_xxx`）

### 2. 添加 Secret 到 xcc-release 仓库

1. 访问 xcc-release 仓库: https://github.com/wshon/xcc-release
2. 点击 "Settings" 标签页
3. 左侧菜单选择 "Secrets and variables" → "Actions"
4. 点击 "New repository secret"
5. 配置 Secret：
   - **Name**: `GH_PAT`（必须是这个名称）
   - **Secret**: 粘贴刚才复制的 token
6. 点击 "Add secret"

### 3. 验证配置

配置完成后，测试是否正常工作：

1. 访问仓库的 "Actions" 标签页
2. 选择 "Manual Build" 工作流
3. 点击 "Run workflow" → "Run workflow"
4. 等待工作流运行
5. 查看运行日志中的 "Checkout repository" 步骤
6. 确认子模块能正常克隆，没有 "Repository not found" 错误

## 经典 Token 方式（备选）

如果需要使用经典 Token：

1. 访问: https://github.com/settings/tokens
2. 点击 "Generate new token" → "Generate new token (classic)"
3. 设置：
   - **Note**: `xcc-release-actions`
   - **Expiration**: 选择合适的过期时间
   - **Select scopes**: 勾选 `repo` (Full control of private repositories)
4. 生成并复制 token
5. 按照上述步骤 2 添加到仓库 Secret

## 安全最佳实践

### Fine-grained Token 的优势

- ✅ 只能访问指定的仓库（xcc-release 和 xcc-src）
- ✅ 权限最小化（只读访问）
- ✅ 更容易审计和管理
- ✅ 泄露风险更低

### 安全建议

1. **定期轮换**: 建议每 90 天更新一次 token
2. **最小权限**: 只授予必需的权限（Contents: Read-only）
3. **监控使用**: 定期检查 token 的使用情况
4. **及时撤销**: 如果怀疑 token 泄露，立即撤销并重新生成

### Token 管理

查看和管理你的 tokens：
- Fine-grained tokens: https://github.com/settings/tokens?type=beta
- Classic tokens: https://github.com/settings/tokens

## 故障排除

### 错误：Repository not found

**原因**：
- PAT 未正确配置
- PAT 没有访问 xcc-src 仓库的权限
- Secret 名称不是 `GH_PAT`

**解决方法**：
1. 检查 Secret 名称是否为 `GH_PAT`（区分大小写）
2. 确认 Fine-grained token 已授权访问 `wshon/xcc-src` 仓库
3. 验证 token 的 Contents 权限至少为 Read-only
4. 尝试重新生成 token 并更新 Secret

### 错误：Token expired

**解决方法**：
1. 访问 https://github.com/settings/tokens?type=beta
2. 找到过期的 token，点击 "Regenerate token"
3. 或者创建新的 token
4. 更新仓库中的 `GH_PAT` Secret

### 错误：Bad credentials

**解决方法**：
1. 确认复制的 token 完整且正确
2. 检查 token 是否已被撤销
3. 重新生成 token 并更新 Secret

### 子模块克隆失败

检查 `.gitmodules` 文件格式：

```ini
[submodule "src"]
    path = src
    url = https://github.com/wshon/xcc-src.git
```

确保：
- 使用 HTTPS URL（不是 SSH URL `git@github.com:...`）
- URL 拼写正确
- 仓库名称正确

## 工作流说明

配置完成后，以下工作流会自动使用 `GH_PAT`：

- **release.yml**: 创建 Release 时自动构建并上传安装包
- **build.yml**: 手动触发的构建工作流

两个工作流都会在 checkout 步骤使用 `secrets.GH_PAT` 来访问私有子模块。
