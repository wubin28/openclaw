# 用 OpenClaw 解决 AI 编程助手的两大记忆痛点

> 本文基于 OpenClaw 源码和官方文档，提供具体可操作的解决方案。

---

## 两大痛点

**痛点 A：每次会话都要重新介绍自己**
每次打开 AI 编程助手，都要重新解释架构背景、技术栈选型、团队编码规范、已知的坑……大量精力耗在"让 AI 追上现在"，而不是真正写代码。

**痛点 B：团队知识沉淀在人的脑子里，AI 无法积累**
老员工离职，经验归零。AI 工具无法跨会话积累团队级的编码惯例、架构决策和历史教训，团队为此反复付出"重新踩坑"的成本。

---

## OpenClaw 的解决架构

OpenClaw 的记忆系统基于一个核心原则：**文件即真相**——AI 只"记得"写进磁盘的内容。

```
工作空间（~/.openclaw/workspace/）
├── AGENTS.md          ← 每次会话自动注入：行为规则、优先级
├── SOUL.md            ← 每次会话自动注入：人格、语气
├── USER.md            ← 每次会话自动注入：用户信息
├── TOOLS.md           ← 每次会话自动注入：本地工具和约定
├── MEMORY.md          ← 每次会话自动注入：长期记忆（策划后的精华）
└── memory/
    ├── 2026-03-24.md  ← 昨日日志（自动注入）
    └── 2026-03-25.md  ← 今日日志（当天自动注入）
```

这些文件在每个会话开始时自动注入系统提示（System Prompt）。无论你是今天第一次开口，还是三个月后回来，AI 都能看到这些文件。

---

## 痛点 A 的解决方案：个人持久化上下文

### 核心思路

把那些你每次都要重新解释的内容，**一次性写进工作空间文件**，让 OpenClaw 在每次会话开始时自动帮你"温习"。

### 操作步骤

#### 第 1 步：找到你的工作空间

```bash
# 默认路径
ls ~/.openclaw/workspace/

# 如果不确定路径，查看配置
cat ~/.openclaw/openclaw.json | grep workspace
```

#### 第 2 步：编写 AGENTS.md（行为规则文件）

这是最重要的文件。把"每次都要解释"的内容写在这里。

```bash
# 打开或创建文件
nano ~/.openclaw/workspace/AGENTS.md
# 或者用你偏好的编辑器
code ~/.openclaw/workspace/AGENTS.md
```

**AGENTS.md 内容模板（编程项目示例）：**

```markdown
# 项目背景

我在开发一个电商订单系统，技术栈：
- 后端：Node.js + TypeScript + Express
- 数据库：PostgreSQL（主库）+ Redis（缓存）
- 部署：Docker + Kubernetes，运行在 AWS EKS
- 测试：Jest + Supertest

## 当前阶段

正在重构支付模块，将单体 PaymentService 拆分为三个独立服务：
OrderPaymentService、RefundService、PaymentAuditService。

# 编码规范

- TypeScript：启用 strict 模式，禁止 any
- 函数命名：动词开头，camelCase（例：processOrder, validatePayment）
- 错误处理：统一使用 AppError 基类，带 errorCode 字段
- 注释：只在"为什么这样做"不明显时加注释，不注释"做了什么"

# 已知的坑

- PaymentGateway SDK 的 `charge()` 方法是同步假异步，内部实际上有 500ms 阻塞，不要在请求路径上直接调用，必须放进队列
- 订单状态机：CREATED → PAID 转换必须加分布式锁（用 Redis SETNX），曾经因此出现重复扣款 bug
- 数据库：orders 表超过 2000 万行，全表扫描会导致生产告警，所有查询必须走索引

# 如何帮我工作

- 优先建议已有模式的一致解法，而不是引入新的抽象
- 遇到权衡时，说清楚每个方案的利弊
- 修改现有代码前，先解释你的理解是否正确
```

#### 第 3 步：编写 USER.md（告诉 AI 你是谁）

```bash
nano ~/.openclaw/workspace/USER.md
```

```markdown
# 关于我

后端工程师，5 年 Node.js 经验，熟悉系统设计。
对 TypeScript 类型体操感到头疼，更喜欢实用的类型而不是炫技。
沟通风格：直接说结论，不需要铺垫，技术细节给完整的，不要简化。
```

#### 第 4 步：让 AI 主动往 MEMORY.md 里写

在对话中，当你解释了某个重要决策或规范时，直接要求 AI 记下来：

```
把刚才我说的关于支付幂等性处理方案记到 MEMORY.md 里
```

**AI 会自动调用 `memory_get`/文件写工具把内容追加到 `MEMORY.md`**，下次会话就能自动看到。

也可以设置自动记忆规则（在 AGENTS.md 里加）：

```markdown
# 记忆规则

每当我们做出以下类型的决策，主动记录到 MEMORY.md：
- 架构决策（选了哪个方案，为什么）
- 排查过的 bug 根因
- 第三方库的已知限制
- 接口约定变更
```

#### 第 5 步：验证效果

```bash
# 查看 MEMORY.md 当前内容
cat ~/.openclaw/workspace/MEMORY.md

# 查看今日记忆日志
cat ~/.openclaw/workspace/memory/$(date +%Y-%m-%d).md

# 用语义搜索查找记忆
openclaw memory search "支付幂等性"
openclaw memory search "已知 bug"
```

#### 第 6 步（可选）：启用向量搜索，让语义检索更准

当 MEMORY.md 内容增多后，开启语义检索可以让 AI 找到"意思相近但措辞不同"的记忆。

```bash
# 检查当前记忆搜索状态
openclaw memory status --deep

# 强制重新索引
openclaw memory index --force

# 搜索测试
openclaw memory search --query "数据库性能问题" --max-results 5
```

在 `~/.openclaw/openclaw.json` 里配置语义搜索（需要 OpenAI 或其他 embedding 服务）：

```jsonc
{
  "agents": {
    "defaults": {
      "memorySearch": {
        "enabled": true,
        "provider": "openai",
        "model": "text-embedding-3-small",
        "query": {
          "maxResults": 6,
          "minScore": 0.35
        }
      }
    }
  }
}
```

---

## 痛点 B 的解决方案：团队知识共享

### 核心思路

把工作空间文件（AGENTS.md、MEMORY.md 等）放进**私有 Git 仓库**，团队成员共用同一份知识库。每人配置自己的 OpenClaw 工作空间指向这个共享仓库。

### 架构示意

```
私有 Git 仓库（例：github.com/your-team/ai-workspace）
│
├── AGENTS.md         ← 团队编码规范、架构决策、已知的坑
├── MEMORY.md         ← 团队长期知识库（精华沉淀）
├── TOOLS.md          ← 团队工具约定
├── memory/           ← 历史日志（可选，看团队是否想共享）
│   ├── 2026-01-15.md
│   └── 2026-03-20.md
└── .gitignore

每位团队成员：
~/.openclaw/workspace → 克隆这个仓库
```

### 操作步骤

#### 第 1 步：创建团队 AI 工作空间仓库

**在 GitHub/GitLab 上创建私有仓库**（以 GitHub CLI 为例）：

```bash
# 方法 A：用 GitHub CLI
gh auth login
cd ~/.openclaw/workspace
git init
gh repo create your-team/ai-workspace --private --source . --remote origin --push

# 方法 B：在 GitHub 网页上创建私有仓库后
cd ~/.openclaw/workspace
git init
git remote add origin https://github.com/your-team/ai-workspace.git
git add AGENTS.md MEMORY.md TOOLS.md USER.md memory/
git commit -m "初始化团队 AI 工作空间"
git push -u origin main
```

#### 第 2 步：明确文件分工

在仓库里建议这样规划：

| 文件 | 内容 | 由谁维护 |
|------|------|---------|
| `AGENTS.md` | 团队规范、架构决策、已知的坑 | 全员 PR 贡献 |
| `MEMORY.md` | 精华知识（定期整理） | 有经验的工程师整理 |
| `TOOLS.md` | 本地工具和脚本约定 | 全员 |
| `USER.md` | **不放入共享仓库**（个人私有） | 每人本地维护 |
| `memory/YYYY-MM-DD.md` | 日常工作日志 | 可选：放入共享 |

> ⚠️ `USER.md` 是个人信息，不应提交到团队仓库。可以在 `.gitignore` 里排除它：
> ```
> USER.md
> .DS_Store
> *.key
> ```

#### 第 3 步：每位团队成员完成初始配置

新成员加入时：

```bash
# 1. 克隆团队工作空间仓库
git clone https://github.com/your-team/ai-workspace.git ~/.openclaw/workspace

# 2. 创建自己的 USER.md（本地私有，不提交）
cat > ~/.openclaw/workspace/USER.md << 'EOF'
# 关于我
姓名：张三
角色：后端工程师，2 年 Go 经验
偏好：直接说结论，代码示例优先于文字解释
EOF

# 3. 验证工作空间结构
ls ~/.openclaw/workspace/
openclaw memory status
```

如果成员的工作空间已经在别的路径，在 `~/.openclaw/openclaw.json` 里指定：

```jsonc
{
  "agents": {
    "defaults": {
      "workspace": "~/.openclaw/workspace"
    }
  }
}
```

#### 第 4 步：建立团队知识维护流程

**知识写入流程（建议）：**

1. **事中记录**：发现重要决策或踩坑时，在 AI 对话中让它记进 MEMORY.md：
   ```
   把刚才排查的 Redis 连接池泄漏的根因和解法记到 MEMORY.md 里
   ```

2. **定期整理**：每周/每月，由一名工程师整理 `memory/` 日志，提炼精华到 `MEMORY.md`：
   ```bash
   # 查看本周日志
   cat ~/.openclaw/workspace/memory/2026-03-*.md

   # 整理后提交
   git add MEMORY.md
   git commit -m "knowledge: 补充支付模块重构决策记录"
   git push
   ```

3. **其他成员同步**：
   ```bash
   cd ~/.openclaw/workspace
   git pull origin main
   ```

#### 第 5 步：老员工离职时的知识转移

员工离职前，执行知识归档：

```bash
# 1. 让 AI 整理该员工的历史记忆
# 在对话中说：
# "请查看 memory/ 目录下过去三个月的日志，整理出关于 [负责模块] 的关键决策、已知限制和注意事项，写入 MEMORY.md"

# 2. 提交知识转移文档
git add MEMORY.md
git commit -m "knowledge: 整理 [姓名] 负责的 [模块] 知识转移"
git push
```

#### 第 6 步（可选）：配置自动备份

```bash
# 在 crontab 里加入定时推送（每天凌晨 2 点）
crontab -e
# 添加：
0 2 * * * cd ~/.openclaw/workspace && git add -A && git diff --staged --quiet || git commit -m "auto: daily memory backup $(date +%Y-%m-%d)" && git push
```

---

## 两个方案的效果对比

| | 没有方案 | 痛点 A 方案（个人持久化） | 痛点 B 方案（团队共享） |
|--|---------|----------------------|---------------------|
| 每次会话是否需要重新介绍 | 是，10-20 分钟 | 否，自动注入 | 否，自动注入 |
| 个人经验能否跨会话保留 | 否 | 是 | 是 |
| 团队规范是否共享 | 否 | 否 | 是 |
| 老员工离职知识是否保留 | 否 | 部分（本地） | 是（Git 历史） |
| 新成员上手效率 | 低 | 低 | 高（克隆即获得） |

---

## 常见问题

**Q：AGENTS.md 写多长合适？**

建议控制在 2000 字以内，超过后会占用大量 token。精华 > 完整：把最关键的规范、最容易踩的坑放进去，细节可以放到 MEMORY.md 里用语义搜索按需召回。

**Q：每天的 memory/YYYY-MM-DD.md 日志会越来越多吗？**

会。OpenClaw 有自动清理机制，旧日志超过配置的磁盘限制会被归档。你也可以定期手动 `git push` 后删除本地旧日志。

**Q：团队成员的 USER.md 都不同，会不会冲突？**

不会。USER.md 放在 `.gitignore` 里，每人本地维护自己的版本，不提交到共享仓库。

**Q：如何让 AI 在会话结束时自动总结并保存记忆？**

在 AGENTS.md 里加入规则：
```markdown
# 会话结束时
每次对话即将结束时（我说"结束"或"再见"），主动把本次会话的关键决策和新发现记录到 memory/YYYY-MM-DD.md。
```

---

## 相关文档

- 内部文档：`dialogs/docs-sessions-and-memory-by-claude-code.md`（会话与记忆完整指南）
- 内部文档：`dialogs/guide-to-use-memos-with-openclaw.md`（MemOS 云端记忆插件，适合需要更精确语义召回的场景）
- 官方文档：https://docs.openclaw.ai/concepts/memory
- 官方文档：https://docs.openclaw.ai/concepts/agent-workspace
- 官方文档：https://docs.openclaw.ai/reference/memory-config
