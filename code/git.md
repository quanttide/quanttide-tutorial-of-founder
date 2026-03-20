# Git 使用最佳实践

本教程介绍量潮科技项目中 Git 的最佳实践。

## 子模块管理

### 更新仓库

```bash
# 仓库级别配置（当前仓库生效）
git config submodule.recurse true

# 全局配置（所有仓库生效）
git config --global submodule.recurse true
```

### 更新子模块

使用完整命令：

```bash
git submodule update --remote --merge
```

### 切换子模块分支

进入子模块目录后，使用标准 git 命令：

```bash
cd <子模块目录>
git checkout main
git pull origin main
```

### 提交子模块更改

1. 在子模块内提交更改：

```bash
cd <子模块目录>
git add -A
git commit -m "描述性信息"
git push
```

2. 在主仓库更新子模块引用：

```bash
cd ..
git add <子模块目录>
git commit -m "更新子模块：<子模块名称>"
git push
```

## 常用命令

### 查看状态

```bash
git status
git diff
git log --oneline -10
```

### 撤销更改

```bash
# 撤销未暂存的更改
git restore <文件>

# 撤销已暂存的更改
git restore --staged <文件>

# 回退到指定提交
git reset --hard <提交哈希>
```

### 分支管理

```bash
# 创建并切换分支
git checkout -b <分支名>

# 切换分支
git checkout <分支名>

# 合并分支
git merge <分支名>
```

## 解决冲突

### 拉取时冲突

```bash
git pull --rebase
# 如果有冲突，手动解决后：
git add <冲突文件>
git rebase --continue
```

### 推送时冲突

```bash
git pull --rebase origin main
# 解决冲突后：
git push
```

## 最佳实践

### 提交信息

- 使用中文描述更改内容
- 简洁明了（建议不超过 50 字）
- 示例：`更新记忆定义：基础教程→工作教程`

### 原子提交

- 每次提交独立完整
- 避免混合不相关的更改
- 先提交子模块，再更新主仓库引用

### 验证优先

- 提交前检查 `git status`
- 推送前确保本地测试通过
- 使用 `git diff` 确认更改内容

### 提交后验证

**重要**：每次提交推送后，必须验证云端链接。

```bash
# 检查远程提交状态
git log --oneline -3
git ls-remote origin main

# 使用 GitHub API 验证
gh api repos/<组织>/<仓库>/commits/main --jq '.sha[0:7] + " " + .commit.message'
```

#### 子模块特别注意

子模块更新容易出错，提交后务必检查：

1. **检查子模块是否成功推送**

```bash
cd <子模块目录>
git log --oneline -3
git ls-remote origin main
```

2. **检查主仓库子模块引用是否更新**

```bash
cd ..
git diff <子模块目录>
```

3. **验证 GitHub 链接**

- 主仓库：https://github.com/<组织>/<主仓库>/blob/main/<子模块目录>
- 子模块：https://github.com/<组织>/<子模块仓库>/blob/main/<修改的文件>

#### 常见问题

- **推送失败**：使用代理解决网络问题
- **分离头指针**：子模块提交前确保在 main 分支
- **冲突解决**：使用 `git pull --rebase` 后手动解决冲突
