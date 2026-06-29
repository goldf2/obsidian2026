# 知识库文档编写规范

> **仓库**: tech-notes (技术知识库)
> **版本**: 1.0.0
> **最后更新**: 2026-01-16

---

## 📚 仓库定位

**知识库 (tech-notes)** 存储**可复用的通用技术知识**，与特定服务器或项目无关。

### 核心原则

1. **通用性**: 内容适用于多个项目和环境
2. **可复用性**: 可以直接应用到不同场景
3. **完整性**: 从原理到实践的完整流程
4. **准确性**: 所有命令和代码经过验证

---

## ✅ 应该存储在本仓库

- ✅ 技术通用指南（Git、Docker、Linux、Python 等）
- ✅ 最佳实践文档（性能优化、安全加固等）
- ✅ 技术原理说明（如何工作、为什么这样设计）
- ✅ 跨项目可复用的配置模板
- ✅ 故障排查手册
- ✅ 技术选型对比
- ✅ 工具使用指南

### 示例

```
✅ 正确示例：
- git/ssh-key-setup-guide.md
  内容：如何在任何 Linux 服务器上配置 SSH 密钥
  适用：所有服务器、所有开发者

- docker/dockerfile-best-practices.md
  内容：编写 Dockerfile 的通用最佳实践
  适用：所有使用 Docker 的项目

- python/virtual-env-guide.md
  内容：Python 虚拟环境管理通用方法
  适用：所有 Python 项目
```

---

## ❌ 不应该存储在本仓库

- ❌ 特定服务器的配置记录 → 应放在 **server_docs**
- ❌ 特定项目的业务逻辑文档 → 应放在**项目仓库**
- ❌ 包含敏感信息的配置 → 不应提交到任何仓库
- ❌ 一次性的操作记录 → 应放在 **server_docs** 的 maintenance/
- ❌ 工作规范和规则定义 → 应放在 **claude_ruler**

### 反例

```
❌ 错误示例：
- AL03-ssh-setup.md
  问题：特定于 AL03 服务器
  应该放在：server_docs/AL03/configuration/

- our-company-git-workflow.md
  问题：特定于某个公司/团队
  应该放在：项目仓库或 server_docs

- api-endpoints-for-project-x.md
  问题：特定于某个项目
  应该放在：项目仓库的文档目录
```

---

## 📂 目录结构规范

### 按技术主题分类

```
tech-notes/
├── README.md                # 知识库总览
├── RULES.md                 # 本文件（编写规范）
├── git/                     # Git 版本控制
├── linux/                   # Linux 系统管理
├── docker/                  # Docker 容器化
├── python/                  # Python 开发
├── javascript/              # JavaScript 开发
├── web2py/                  # Web2py 框架
├── database/                # 数据库相关
├── security/                # 安全相关
├── performance/             # 性能优化
├── devops/                  # DevOps 实践
└── tools/                   # 开发工具
```

### 文件命名规范

**格式**: `主题-具体内容.md`

**规则**：
- 使用小写字母
- 使用连字符 `-` 分隔单词
- 使用描述性名称
- 避免缩写（除非是通用缩写如 ssh、api）
- 不在文件名中包含版本号

**示例**：
```
✅ 正确：
- ssh-key-setup-guide.md
- dockerfile-best-practices.md
- python-virtual-environment.md
- git-workflow-comparison.md

❌ 错误：
- ssh.md                    (太简略)
- SSHKeySetup.md           (使用大写)
- ssh_key_setup.md         (使用下划线)
- git-v2.md                (包含版本号)
```

---

## 📝 文档编写标准

### 必须包含的部分

每个文档必须包含以下结构：

```markdown
# 文档标题

> **版本**: 1.0.0
> **最后更新**: YYYY-MM-DD
> **适用范围**: 说明适用的环境/场景
> **标签**: #tag1 #tag2 #tag3

---

## 📋 目录

- [概述](#概述)
- [原理说明](#原理说明)  （可选，但推荐）
- [前置要求](#前置要求)
- [操作步骤](#操作步骤)
- [最佳实践](#最佳实践)
- [故障排查](#故障排查)
- [参考资料](#参考资料)

---

## 概述

简要说明（2-3 句话）：
- 这个技术/工具是什么
- 为什么需要它
- 何时使用它

---

## 原理说明

（可选但推荐）解释工作原理，帮助读者理解。

---

## 前置要求

列出需要满足的条件：
- 系统要求
- 必需的软件/工具
- 必需的权限
- 前置知识

---

## 操作步骤

### 步骤 1: ...

```bash
# 实际可用的命令
command --with-options
```

**说明**: 解释这一步在做什么

### 步骤 2: ...

...

---

## 最佳实践

### ✅ 推荐做法

1. **做法 1**
   - 原因
   - 示例

2. **做法 2**
   - 原因
   - 示例

### ❌ 避免做法

1. **反模式 1**
   - 问题
   - 正确做法

---

## 故障排查

### 问题 1: 错误描述

**原因**: ...

**解决方法**:
```bash
# 解决命令
```

---

## 参考资料

- [官方文档](URL)
- [相关文章](URL)
- [相关知识库文档](./related-doc.md)

---

*版本: 1.0.0*
*最后更新: YYYY-MM-DD*
*维护者: 团队名称*
```

---

## 💡 代码示例要求

### 所有代码必须可执行

```bash
# ✅ 正确：完整可执行的命令
git clone https://github.com/user/repo.git
cd repo
npm install

# ❌ 错误：占位符或不完整命令
git clone <repository-url>
cd <directory>
npm install
```

### 提供多种场景

```bash
# 场景 1: Ubuntu/Debian
sudo apt-get install package

# 场景 2: CentOS/RHEL
sudo yum install package

# 场景 3: macOS
brew install package
```

### 添加必要的注释

```bash
# 创建目录
mkdir -p ~/project/data

# 设置权限（重要：必须是 700）
chmod 700 ~/project/data
```

---

## 📊 文档质量标准

### 必须满足的条件

- [ ] 标题清晰，准确描述内容
- [ ] 包含版本号和更新日期
- [ ] 包含完整的目录
- [ ] 原理说明清晰易懂
- [ ] 步骤完整且有序
- [ ] 所有代码示例可执行
- [ ] 包含故障排查部分
- [ ] 引用相关资源
- [ ] 无拼写和语法错误
- [ ] 链接可正常访问

### 推荐包含的内容

- [ ] 架构图或流程图
- [ ] 对比表格（多种方案对比）
- [ ] 实际案例分析
- [ ] 安全注意事项
- [ ] 性能考虑
- [ ] 成本考虑（如有）

---

## 🔄 文档更新流程

### 何时更新文档

- 技术有重大版本更新
- 发现错误或不准确的信息
- 社区提供更好的实践方法
- 添加新的故障排查案例
- 补充缺失的内容

### 更新步骤

#### 简化工作流（推荐）

对于文档仓库，大部分操作是**新增文档**，可以**直接提交到主分支**：

```bash
# 1. 创建/更新文档
vi git/new-guide.md

# 2. 提交并推送
git add .
git commit -m "docs(git): 添加新指南"
git push origin master
```

**适用场景**（直接提交）：
- ✅ 创建新文档（占 90% 操作）
- ✅ 修正拼写/格式错误
- ✅ 小幅内容补充
- ✅ 更新版本号/日期

#### 使用分支（可选）

仅在以下情况使用分支：

```bash
# 1. 创建分支
git checkout -b update/major-refactor

# 2. 进行大规模修改
...

# 3. 提交更新
git add .
git commit -m "docs: 重构文档结构"

# 4. 合并到主分支
git checkout master
git merge update/major-refactor
git push origin master

# 5. 删除分支
git branch -d update/major-refactor
```

**适用场景**（使用分支）：
- ⚠️ 重大文档重构（目录结构调整）
- ⚠️ 多人协作编辑同一文档
- ⚠️ 需要 review 的重要更新
- ⚠️ 实验性内容（不确定是否采纳）

**原因**：
- 文档仓库按主题/日期隔离，冲突风险极低
- 新增文档不影响现有内容
- 文档错误影响小，可以快速修正

### 版本号规则

遵循**语义化版本** (Semantic Versioning)：

- **主版本号** (Major): 重大变更，可能不兼容旧版本
  - 示例：1.0.0 → 2.0.0
- **次版本号** (Minor): 添加新内容，向后兼容
  - 示例：1.0.0 → 1.1.0
- **修订号** (Patch): 修正错误、小改进
  - 示例：1.0.0 → 1.0.1

---

## 📖 文档引用规范

### 引用其他知识库文档

```markdown
## 相关文档

- [Git 凭据管理](./git/credential-management.md)
- [Docker 网络配置](./docker/networking-guide.md)
```

### 引用规则库

```markdown
## 参考规则

- [RULE-DEVOPS-005: 文档管理规则](https://gitee.com/goldf2/claude_ruler/blob/main/devops/documentation-management.md)
```

### 引用运维库（作为实际应用示例）

```markdown
## 实际应用案例

- [AL03 服务器 SSH 配置记录](https://gitee.com/goldf2/server_docs/blob/master/AL03/configuration/git-ssh-setup.md)
  - 展示了如何在实际环境中应用本指南
```

### 引用外部资源

```markdown
## 外部资源

- [GitHub SSH 官方文档](https://docs.github.com/en/authentication/connecting-to-github-with-ssh)
- [Stack Overflow 讨论](URL)
```

---

## ⚠️ 安全和隐私

### 禁止包含的内容

- ❌ 真实的密码、密钥、令牌
- ❌ 真实的 IP 地址、域名（除非是公开服务）
- ❌ 真实的用户名、邮箱（除非是示例账号）
- ❌ 内部系统架构细节
- ❌ 敏感的业务信息

### 使用示例值

```markdown
✅ 正确：
- 用户名：`example_user`
- 邮箱：`user@example.com`
- 域名：`example.com`
- IP：`192.0.2.1` (RFC 5737 测试地址)

❌ 错误：
- 用户名：`john_smith_real`
- 邮箱：`john@ourcompany.com`
- 域名：`internal.ourcompany.com`
- IP：`10.0.5.123` (可能是真实内网地址)
```

---

## 📋 提交规范

### Commit Message 格式

```
docs(category): 简短描述

详细说明（可选）
```

**category 示例**：
- `git`: Git 相关文档
- `docker`: Docker 相关文档
- `python`: Python 相关文档
- `linux`: Linux 相关文档

**action 动词**：
- `添加` / `add`: 新增文档
- `更新` / `update`: 更新现有文档
- `修正` / `fix`: 修正错误
- `删除` / `remove`: 删除文档

**示例**：
```bash
# 添加新文档
git commit -m "docs(docker): 添加 Dockerfile 最佳实践指南"

# 更新现有文档
git commit -m "docs(git): 更新 SSH 密钥配置步骤"

# 修正错误
git commit -m "docs(python): 修正虚拟环境激活命令错误"
```

---

## 🎯 AI 助手工作指引

### 创建新文档时

1. **确认归属**：确认内容属于知识库（通用、可复用）
2. **读取本规范**：在创建文档前，必须先读取 `tech-notes/RULES.md`
3. **选择目录**：确定文档应放在哪个主题目录下
4. **使用模板**：遵循文档编写标准模板
5. **验证内容**：确保所有代码示例可执行
6. **提交推送**：使用规范的 commit message

### 更新现有文档时

1. **读取现有文档**：先读取要更新的文档
2. **读取本规范**：确认更新符合规范要求
3. **更新版本号**：根据变更类型更新版本号
4. **更新日期**：修改"最后更新"日期
5. **验证链接**：确保所有链接仍然有效
6. **提交推送**：说明更新内容

### 判断文档归属

**问自己三个问题**：

1. **通用性**：这个内容适用于多个项目/环境吗？
   - 是 → 可能属于知识库
   - 否 → 可能属于运维库或项目文档

2. **可复用性**：其他团队/项目可以直接使用吗？
   - 是 → 可能属于知识库
   - 否 → 可能属于运维库或项目文档

3. **时效性**：这是一次性操作记录还是长期有效的指南？
   - 长期有效 → 可能属于知识库
   - 一次性记录 → 属于运维库

**决策矩阵**：

| 通用 | 可复用 | 长期有效 | 归属 |
|------|--------|----------|------|
| ✅ | ✅ | ✅ | **知识库 (tech-notes)** |
| ❌ | ❌ | ❌ | **运维库 (server_docs)** 维护记录 |
| ❌ | ❌ | ✅ | **运维库 (server_docs)** 配置记录 |
| ✅ | ❌ | ✅ | **项目文档** 或 **运维库** |

---

## 🔗 相关资源

### 规则定义

- [RULE-DEVOPS-005: 文档管理与知识库规则](https://gitee.com/goldf2/claude_ruler/blob/main/devops/documentation-management.md)
- [CLAUDE.md - 文档管理基础规则](https://gitee.com/goldf2/claude_ruler/blob/main/CLAUDE.md)

### 其他仓库

- [规则库 (claude_ruler)](https://gitee.com/goldf2/claude_ruler)
- [运维库 (server_docs)](https://gitee.com/goldf2/server_docs)

### 参考标准

- [Markdown 规范](https://www.markdownguide.org/)
- [语义化版本](https://semver.org/lang/zh-CN/)
- [Conventional Commits](https://www.conventionalcommits.org/zh-hans/)

---

*版本: 1.0.0*
*最后更新: 2026-01-16*
*维护者: DevOps 团队*
