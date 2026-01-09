# Git使用规范

## 1. 分支管理规范

### 1.1 分支类型

```
master/main      # 主分支（生产环境）
├── develop      # 开发分支
├── feature/*    # 功能分支
├── hotfix/*     # 紧急修复分支
├── release/*    # 发布分支
└── bugfix/*     # Bug修复分支
```

### 1.2 分支说明

| 分支类型 | 命名规则 | 生命周期 | 说明 |
|---------|---------|---------|------|
| master | master | 永久 | 生产环境代码，只能合并不能直接提交 |
| develop | develop | 永久 | 开发环境代码 |
| feature | feature/功能名 | 临时 | 新功能开发 |
| hotfix | hotfix/问题描述 | 临时 | 生产环境紧急修复 |
| release | release/版本号 | 临时 | 预发布分支 |
| bugfix | bugfix/问题描述 | 临时 | 测试环境Bug修复 |

---

## 2. 分支工作流

### 2.1 Git Flow 工作流

```
master (v1.0)
  ↓
develop
  ↓
feature/user-login ← 开发新功能
  ↓
develop ← 合并功能
  ↓
release/v1.1 ← 创建发布分支
  ↓
master (v1.1) ← 发布到生产
```

### 2.2 功能开发流程

```bash
# 1. 从develop创建功能分支
git checkout develop
git pull origin develop
git checkout -b feature/user-login

# 2. 开发功能并提交
git add .
git commit -m "feat: 实现用户登录功能"

# 3. 推送到远程
git push origin feature/user-login

# 4. 创建Pull Request
# 在GitLab/GitHub上创建PR，指定reviewer

# 5. Code Review通过后合并到develop
# 通过网页合并或命令行合并
git checkout develop
git merge --no-ff feature/user-login
git push origin develop

# 6. 删除功能分支
git branch -d feature/user-login
git push origin --delete feature/user-login
```

### 2.3 紧急修复流程

```bash
# 1. 从master创建hotfix分支
git checkout master
git pull origin master
git checkout -b hotfix/fix-payment-bug

# 2. 修复问题并提交
git add .
git commit -m "fix: 修复支付金额计算错误"

# 3. 推送到远程
git push origin hotfix/fix-payment-bug

# 4. 合并到master
git checkout master
git merge --no-ff hotfix/fix-payment-bug
git tag -a v1.0.1 -m "紧急修复支付Bug"
git push origin master --tags

# 5. 同步到develop
git checkout develop
git merge --no-ff hotfix/fix-payment-bug
git push origin develop

# 6. 删除hotfix分支
git branch -d hotfix/fix-payment-bug
git push origin --delete hotfix/fix-payment-bug
```

---

## 3. 提交规范

### 3.1 Commit Message 格式

```
<type>(<scope>): <subject>

<body>

<footer>
```

#### 格式说明
- **type**: 提交类型（必填）
- **scope**: 影响范围（可选）
- **subject**: 简短描述（必填）
- **body**: 详细描述（可选）
- **footer**: 关联issue（可选）

### 3.2 提交类型

| 类型 | 说明 | 示例 |
|------|------|------|
| feat | 新功能 | feat: 添加用户注册功能 |
| fix | Bug修复 | fix: 修复订单金额计算错误 |
| docs | 文档更新 | docs: 更新API文档 |
| style | 代码格式调整 | style: 格式化代码 |
| refactor | 重构 | refactor: 重构用户服务层 |
| perf | 性能优化 | perf: 优化商品查询SQL |
| test | 测试相关 | test: 添加订单服务单元测试 |
| chore | 构建/工具变动 | chore: 升级Spring Boot版本 |
| revert | 回滚 | revert: 回滚登录功能 |

### 3.3 提交示例

#### ✅ 正确示例
```bash
# 简单提交
git commit -m "feat: 实现用户登录功能"

# 完整提交
git commit -m "feat(user): 实现用户登录功能

1. 添加登录接口
2. 实现JWT token生成
3. 添加登录日志记录

Closes #123"

# 多文件修改
git commit -m "refactor(order): 重构订单服务

1. 提取订单计算逻辑
2. 优化库存扣减流程
3. 添加事务控制

Related to #456"
```

#### ❌ 错误示例
```bash
# 不明确的提交
git commit -m "修改代码"
git commit -m "fix bug"
git commit -m "update"

# 一次提交多个不相关的修改
git commit -m "feat: 添加登录功能，修复订单Bug，更新配置文件"
```

### 3.4 提交粒度

#### 原则
- 一个提交只做一件事
- 相关的修改放在一起
- 不相关的修改分开提交

#### 示例
```bash
# ✅ 正确 - 提交粒度合适
git add src/main/java/com/company/controller/UserController.java
git add src/main/java/com/company/service/UserService.java
git commit -m "feat: 实现用户注册功能"

git add src/main/resources/application.yml
git commit -m "chore: 修改数据库连接配置"

# ❌ 错误 - 一次提交过多不相关内容
git add .
git commit -m "各种修改"
```

---

## 4. 代码合并规范

### 4.1 合并策略

#### Fast-Forward（快进合并）
```bash
# 适用场景：没有冲突，分支历史线性
git merge feature/user-login
```

#### No-Fast-Forward（非快进合并）
```bash
# 推荐：保留分支历史
git merge --no-ff feature/user-login
```

#### Squash（压缩合并）
```bash
# 适用场景：合并前压缩多个提交为一个
git merge --squash feature/user-login
git commit -m "feat: 实现用户登录功能"
```

### 4.2 冲突解决

```bash
# 1. 拉取最新代码
git checkout develop
git pull origin develop

# 2. 合并功能分支
git merge feature/user-login
# Auto-merging src/main/java/com/company/service/UserService.java
# CONFLICT (content): Merge conflict in src/main/java/com/company/service/UserService.java

# 3. 查看冲突文件
git status

# 4. 手动解决冲突
# 编辑冲突文件，删除冲突标记
<<<<<<< HEAD
代码A
=======
代码B
>>>>>>> feature/user-login

# 5. 标记冲突已解决
git add src/main/java/com/company/service/UserService.java

# 6. 完成合并
git commit -m "Merge branch 'feature/user-login' into develop"
```

---

## 5. 标签管理

### 5.1 标签规范

#### 语义化版本号
```
v主版本号.次版本号.修订号

v1.0.0  # 初始版本
v1.1.0  # 新增功能
v1.1.1  # Bug修复
v2.0.0  # 重大变更
```

### 5.2 标签操作

```bash
# 创建标签
git tag -a v1.0.0 -m "发布v1.0.0版本"

# 推送标签到远程
git push origin v1.0.0

# 推送所有标签
git push origin --tags

# 删除本地标签
git tag -d v1.0.0

# 删除远程标签
git push origin --delete v1.0.0

# 查看所有标签
git tag -l

# 查看标签详情
git show v1.0.0

# 基于标签创建分支
git checkout -b hotfix/fix-bug v1.0.0
```

---

## 6. Code Review 规范

### 6.1 提交 PR 前检查

- [ ] 代码编译通过
- [ ] 单元测试全部通过
- [ ] 代码格式符合规范
- [ ] 没有明显的Bug
- [ ] 添加了必要的注释
- [ ] 更新了相关文档
- [ ] PR描述清晰完整

### 6.2 PR 描述模板

```markdown
## 变更类型
- [ ] 新功能
- [ ] Bug修复
- [ ] 重构
- [ ] 文档更新

## 变更说明
简要说明本次变更的内容和原因

## 测试说明
说明如何测试本次变更

## 相关Issue
Closes #123

## 截图（如果适用）
附上界面变更的截图

## 检查清单
- [ ] 代码编译通过
- [ ] 单元测试通过
- [ ] 代码规范检查通过
- [ ] 已添加必要的注释
```

### 6.3 Review 检查要点

#### 功能正确性
- 业务逻辑是否正确
- 边界条件是否处理
- 异常情况是否考虑

#### 代码质量
- 命名是否规范
- 代码结构是否清晰
- 是否有重复代码
- 是否符合设计模式

#### 性能
- 是否有性能问题
- SQL是否优化
- 是否有N+1查询

#### 安全性
- 是否有SQL注入风险
- 敏感信息是否加密
- 是否有权限校验

---

## 7. 常用命令

### 7.1 日常开发

```bash
# 查看状态
git status

# 查看差异
git diff

# 查看提交历史
git log --oneline --graph

# 暂存修改
git stash
git stash pop

# 撤销修改
git checkout -- <file>

# 修改最后一次提交
git commit --amend

# 查看远程分支
git branch -r

# 拉取并合并
git pull origin develop

# 推送到远程
git push origin feature/user-login
```

### 7.2 分支操作

```bash
# 创建并切换分支
git checkout -b feature/user-login

# 切换分支
git checkout develop

# 删除本地分支
git branch -d feature/user-login

# 强制删除本地分支
git branch -D feature/user-login

# 删除远程分支
git push origin --delete feature/user-login

# 重命名分支
git branch -m old-name new-name
```

### 7.3 回滚操作

```bash
# 回滚最后一次提交（保留修改）
git reset --soft HEAD~1

# 回滚最后一次提交（丢弃修改）
git reset --hard HEAD~1

# 回滚到指定提交
git reset --hard <commit-id>

# 创建回滚提交
git revert <commit-id>
```

---

## 8. Git 配置

### 8.1 全局配置

```bash
# 配置用户名和邮箱
git config --global user.name "张三"
git config --global user.email "zhangsan@company.com"

# 配置编辑器
git config --global core.editor "vim"

# 配置别名
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.cm commit
git config --global alias.lg "log --oneline --graph"

# 配置换行符（Windows）
git config --global core.autocrlf true

# 配置换行符（Mac/Linux）
git config --global core.autocrlf input
```

### 8.2 .gitignore 配置

```gitignore
# Java
*.class
*.jar
*.war
*.ear
target/
*.iml

# IDE
.idea/
.vscode/
.settings/
.project
.classpath

# Log
*.log
logs/

# OS
.DS_Store
Thumbs.db

# Other
*.bak
*.swp
*.tmp
```

---

## 9. 最佳实践

### DO（应该做）
✅ 频繁提交，保持提交粒度合适  
✅ 写清晰的commit message  
✅ 提交前进行代码审查  
✅ 合并前解决所有冲突  
✅ 定期同步develop分支  
✅ 使用 --no-ff 合并保留历史  

### DON'T（不应该做）
❌ 直接提交到master/develop  
❌ 提交不完整的代码  
❌ 提交包含密码等敏感信息  
❌ 一次提交多个不相关修改  
❌ 使用不清晰的commit message  
❌ 忽略合并冲突  

---

## 10. 检查清单

### 提交前检查
- [ ] 代码编译通过
- [ ] 测试用例通过
- [ ] 代码格式化完成
- [ ] 没有console.log/System.out
- [ ] 没有注释掉的代码
- [ ] 没有敏感信息
- [ ] commit message规范

### 合并前检查
- [ ] 已经过Code Review
- [ ] 所有评论已处理
- [ ] CI/CD流水线通过
- [ ] 没有冲突
- [ ] 功能测试通过
- [ ] 已同步最新代码
