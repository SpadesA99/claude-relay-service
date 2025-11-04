# Fork 仓库维护指南

本文档说明如何在 fork 的仓库中维护自己的分支，同时保持与原仓库的同步。

## 🌿 分支策略

### 分支说明

- **`main`**: 从原仓库同步的主分支（不要直接在此分支开发）
- **`custom`**: 你的自定义开发分支（触发自动构建和发布）
- **`dev`**: 日常开发分支（可选）

### 工作流程

```
原仓库 (upstream/main)
    ↓ 同步
你的 fork (origin/main)
    ↓ 合并
自定义分支 (origin/custom) → 自动构建 Docker 镜像
```

---

## 🔄 初始设置

### 1. 添加原仓库为 upstream

```bash
# 克隆你的 fork 仓库
git clone https://github.com/SpadesA99/claude-relay-service.git
cd claude-relay-service

# 添加原仓库为 upstream
git remote add upstream https://github.com/ORIGINAL_OWNER/claude-relay-service.git

# 验证远程仓库
git remote -v
# origin    https://github.com/SpadesA99/claude-relay-service.git (fetch)
# origin    https://github.com/SpadesA99/claude-relay-service.git (push)
# upstream  https://github.com/ORIGINAL_OWNER/claude-relay-service.git (fetch)
# upstream  https://github.com/ORIGINAL_OWNER/claude-relay-service.git (push)
```

### 2. 创建自定义分支

```bash
# 基于 main 创建 custom 分支
git checkout -b custom
git push -u origin custom
```

---

## 🚀 日常开发流程

### 方式 1：在 custom 分支直接开发（简单）

```bash
# 1. 切换到 custom 分支
git checkout custom

# 2. 修改代码
vim src/services/someService.js

# 3. 提交并推送
git add .
git commit -m "feat: 添加新功能"
git push origin custom

# ✅ 自动触发构建：版本号升级 → 构建 Docker 镜像 → 发布 Release
```

### 方式 2：使用 dev 分支开发（推荐用于复杂功能）

```bash
# 1. 创建开发分支
git checkout -b dev custom

# 2. 开发和测试
# ... 修改代码 ...
git add .
git commit -m "feat: 开发新功能"

# 3. 合并到 custom 分支触发发布
git checkout custom
git merge dev
git push origin custom

# ✅ 自动触发构建
```

---

## 🔄 同步原仓库更新

### 步骤 1：同步 main 分支

```bash
# 1. 切换到 main 分支
git checkout main

# 2. 拉取原仓库的最新更新
git fetch upstream
git merge upstream/main

# 3. 推送到你的 fork
git push origin main
```

### 步骤 2：将更新合并到 custom 分支

```bash
# 1. 切换到 custom 分支
git checkout custom

# 2. 合并 main 的更新
git merge main

# 3. 解决冲突（如果有）
# ... 手动解决冲突 ...
git add .
git commit -m "chore: 同步原仓库更新"

# 4. 推送到你的 custom 分支
git push origin custom

# ✅ 如果有实质性代码变更，会自动触发构建
```

### 自动化同步脚本（可选）

创建一个脚本 `sync-upstream.sh`：

```bash
#!/bin/bash

echo "🔄 同步原仓库更新..."

# 切换到 main 分支
git checkout main

# 拉取原仓库更新
git fetch upstream
git merge upstream/main

# 推送到你的 fork
git push origin main

# 切换到 custom 分支
git checkout custom

# 询问是否合并
read -p "是否将更新合并到 custom 分支? (y/n) " -n 1 -r
echo
if [[ $REPLY =~ ^[Yy]$ ]]; then
    git merge main

    # 检查是否有冲突
    if [ $? -eq 0 ]; then
        echo "✅ 合并成功，推送到 custom 分支..."
        git push origin custom
    else
        echo "⚠️ 有冲突需要解决，请手动解决后推送"
    fi
else
    echo "⏭️ 跳过合并到 custom 分支"
fi

echo "✅ 同步完成！"
```

使用方法：
```bash
chmod +x sync-upstream.sh
./sync-upstream.sh
```

---

## 🐳 Docker 镜像发布

### 自动触发条件

只有推送到 **`custom`** 分支时才会触发自动构建，且满足：
- ✅ 不是由 `github-actions[bot]` 提交
- ✅ 提交消息中不包含 `[skip ci]`
- ✅ 有实质性代码变更（不仅仅是文档、配置等）

### 镜像命名

构建后的镜像：
```bash
# Docker Hub
spadesa99/claude-relay-service:latest
spadesa99/claude-relay-service:v1.1.191
spadesa99/claude-relay-service:1.1.191

# GitHub Container Registry
ghcr.io/spadesa99/claude-relay-service:latest
ghcr.io/spadesa99/claude-relay-service:v1.1.191
ghcr.io/spadesa99/claude-relay-service:1.1.191
```

### 跳过自动构建

如果只是文档更新或想跳过构建：
```bash
git commit -m "docs: 更新文档 [skip ci]"
git push origin custom
```

---

## 📋 版本号管理

### 自动版本号

- 每次构建会自动递增 patch 版本（例如：`1.1.190` → `1.1.191`）
- 版本号存储在 `VERSION` 文件中
- 自动创建 git tag（例如：`v1.1.191`）

### 手动指定版本号

如果需要手动升级大版本或小版本：

```bash
# 1. 修改 VERSION 文件
echo "2.0.0" > VERSION

# 2. 提交并推送
git add VERSION
git commit -m "chore: bump version to 2.0.0"
git push origin custom

# ✅ 下次构建会使用 2.0.0，然后自动递增
```

---

## ⚙️ GitHub Actions 配置

### 必需的 Secrets

在你的 fork 仓库中配置：`https://github.com/SpadesA99/claude-relay-service/settings/secrets/actions`

1. **Docker Hub**（必需）:
   - `DOCKERHUB_USERNAME`: `spadesa99`
   - `DOCKERHUB_TOKEN`: 从 Docker Hub 获取的 Access Token

2. **Telegram 通知**（可选）:
   - `TELEGRAM_BOT_TOKEN`: 你的 Telegram Bot Token
   - `TELEGRAM_CHAT_ID`: 你的 Telegram Chat ID

### 查看构建状态

- Actions 页面：`https://github.com/SpadesA99/claude-relay-service/actions`
- Releases 页面：`https://github.com/SpadesA99/claude-relay-service/releases`

---

## 🎯 最佳实践

### 1. 分支隔离
- ✅ 保持 `main` 分支干净，只用于同步原仓库
- ✅ 所有自定义修改都在 `custom` 分支
- ✅ 使用 `dev` 分支进行实验性开发

### 2. 定期同步
```bash
# 每周或有重要更新时同步
./sync-upstream.sh
```

### 3. 冲突处理
- 优先保留你的自定义功能
- 仔细审查原仓库的变更
- 测试合并后的功能

### 4. 版本标记
```bash
# 在 custom 分支的重要里程碑打标签
git tag -a custom-v1.0.0 -m "自定义版本 1.0.0"
git push origin custom-v1.0.0
```

### 5. 文档维护
- 在 `custom` 分支维护你自己的 README
- 记录与原仓库的差异
- 标注自定义功能

---

## 🔧 高级配置

### 同时监听多个分支

如果需要 `main` 和 `custom` 都触发构建，修改 `.github/workflows/auto-release-pipeline.yml`：

```yaml
on:
  push:
    branches:
      - custom  # 自定义分支
      - main    # 同步分支
```

### 不同分支使用不同配置

可以创建多个 workflow 文件：

```yaml
# .github/workflows/custom-release.yml
name: Custom Release
on:
  push:
    branches:
      - custom

# .github/workflows/main-sync.yml
name: Main Sync
on:
  push:
    branches:
      - main
```

---

## 📚 常见问题

### Q1: 如何查看与原仓库的差异？

```bash
git checkout custom
git fetch upstream
git diff upstream/main...custom
```

### Q2: 如何放弃 custom 分支的某些提交？

```bash
# 回退到之前的提交
git checkout custom
git reset --hard <commit-hash>
git push origin custom --force
```

### Q3: 如何重新与原仓库对齐？

```bash
# 危险操作：完全放弃自定义修改，与原仓库对齐
git checkout custom
git reset --hard upstream/main
git push origin custom --force
```

---

## 📞 需要帮助？

- 查看构建日志：[Actions](https://github.com/SpadesA99/claude-relay-service/actions)
- 查看 Docker 镜像：[Docker Hub](https://hub.docker.com/r/spadesa99/claude-relay-service)
- 原仓库地址：[claude-relay-service](https://github.com/ORIGINAL_OWNER/claude-relay-service)
