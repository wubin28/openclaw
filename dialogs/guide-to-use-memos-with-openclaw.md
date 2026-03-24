# 将 MemOS 接入本地 OpenClaw 指南

> **说明**：本指南基于 OpenClaw 当前源代码，在原 MemOS 官方指南的基础上修正了若干与实际代码不符的内容。每处修正都有说明。
>
> **适用环境**：本地 Mac mini（通过 OpenClaw Mac 应用运行 gateway）。原指南中涉及云端部署的部分已相应调整。

---

## 为什么需要 MemOS 插件？

### OpenClaw 原生记忆的三个痛点

**痛点 1：Token 消耗随对话轮次线性增长**

```
第 1 次对话: 500 tokens
第 2 次对话: 500 + 800 = 1,300 tokens
第 3 次对话: 1,300 + 600 = 1,900 tokens
第 10 次对话: 10,000+ tokens
```

当 OpenClaw 执行屏幕监控、定时任务或复杂工作流时，增速更快。

**痛点 2：全局记忆膨胀失控**

`MEMORY.md` 随时间不断累积，所有历史记忆都会被注入每次对话的上下文，即便大部分内容与当前任务无关。

**痛点 3：记忆依赖模型主动记录**

OpenClaw 的记忆系统需要模型自行决定是否写入，容易遗漏你认为重要的信息。

### MemOS 插件的解决思路

| 效果 | 原理 |
|---|---|
| Token 成本可控 | 从"全量注入上下文"变为"按任务精确召回 3–5 条相关记忆" |
| 检索更准 | MemOS 提供结构化、多粒度、语义检索 + 规则过滤 |
| 记忆更干净 | 工具调用长输出经压缩/摘要后再写入，避免长输出反复污染上下文 |

---

## 快速开始

### 第 1 步：确认 OpenClaw 已安装并运行

```sh
# 安装最新版（如尚未安装）
npm install -g openclaw@latest

# 启动本地 gateway（Mac 用户：直接打开 OpenClaw Mac 应用即可）
# 验证 gateway 是否在线：
openclaw status
```

> **本地 Mac mini 说明**：你的 gateway 就是 OpenClaw Mac 应用本身。无需单独启动进程，确保 Mac 应用已打开即可。

---

### 第 2 步：获取并配置 MemOS API Key

#### 2.1 获取 API Key

登录 / 注册 [MemOS Cloud](https://cloud.memos.io)（第三方服务），在控制台获取你的 API Key（格式类似 `mpg-...`）。

> **注意**：MemOS Cloud 是 MemTensor 运营的云端服务。即使你的 OpenClaw 运行在本地 Mac mini 上，记忆数据仍会通过该插件发送到 MemOS Cloud 进行处理和存储。

#### 2.2 配置环境变量

**⚠️ 修正说明（对比原指南）**：原指南声称插件会依次读取 `~/.openclaw/.env`、`~/.moltbot/.env`、`~/.clawdbot/.env`。经核查源码（`src/config/state-dir-dotenv.ts`），**OpenClaw 只读取 `~/.openclaw/.env`** 这一个 dotenv 文件。`~/.moltbot/` 和 `~/.clawdbot/` 是历史遗留的配置目录，不会被作为 dotenv 文件加载。

**推荐方式：写入 `~/.openclaw/.env`**（OpenClaw 启动时自动加载）

```sh
echo 'MEMOS_API_KEY=mpg-你的key' >> ~/.openclaw/.env
```

**备选方式：写入 shell 环境变量**（只在当前 shell 生效，不推荐用于持久配置）

```sh
# zsh（macOS 默认）
echo 'export MEMOS_API_KEY="mpg-你的key"' >> ~/.zshrc
source ~/.zshrc
```

**最小配置**（`~/.openclaw/.env` 文件内容）：

```env
MEMOS_API_KEY=mpg-你的key
```

---

### 第 3 步：安装插件

#### 方案 A — 通过openclaw命令安装（推荐）

```sh
openclaw plugins install @memtensor/memos-cloud-openclaw-plugin@latest
openclaw gateway restart
```

---

#### 方案 B — 手动安装（不依赖 agent 命令）

1. 从 npm 下载最新 `.tgz` 包：

   ```sh
   npm pack @memtensor/memos-cloud-openclaw-plugin@latest
   ```

2. 解压到本地目录：

   ```sh
   mkdir -p ~/.openclaw/extensions/memos-cloud-openclaw-plugin
   tar -xzf memtensor-memos-cloud-openclaw-plugin-*.tgz \
     -C ~/.openclaw/extensions/memos-cloud-openclaw-plugin \
     --strip-components=1
   ```

3. 编辑 `~/.openclaw/openclaw.json`，添加以下配置：

   ```jsonc
   {
     "plugins": {
       "entries": {
         "memos-cloud-openclaw-plugin": { "enabled": true }
       },
       "load": {
         "paths": [
           "~/.openclaw/extensions/memos-cloud-openclaw-plugin"
         ]
       }
     }
   }
   ```

   > **说明**：`plugins.entries` 中每个键是插件 ID，`enabled: true` 启用该插件（布尔值，不是字符串）。`plugins.load.paths` 是 OpenClaw 会额外扫描插件的本地目录列表。

4. 重启 gateway（同方案 A）。

---

### 验证安装

重启后，发送以下消息验证插件是否已加载：

```
/status
```

或询问 agent：

```
已安装哪些插件？
```

---

### 第 4 步：测试记忆功能

完成安装后，进行多轮对话测试：

**第一次会话**：

```
我最喜欢的编程语言是 Python
我正在开发一个电商项目，使用 FastAPI + PostgreSQL
```

**关闭会话，重新启动一次新对话后**：

```
你还记得我喜欢用什么编程语言吗？
我之前说的项目用了哪些技术栈？
```

如果 MemOS 插件工作正常，agent 应能从 MemOS Cloud 检索到你在上一次会话中提到的信息并给出准确回答。

---

## 进阶配置

### 通过 `plugins.entries` 传入插件专属配置

如果插件支持配置项（参考 MemTensor 官方仓库文档），可以通过 `plugins.entries[id].config` 传入：

```jsonc
{
  "plugins": {
    "entries": {
      "memos-cloud-openclaw-plugin": {
        "enabled": true,
        "config": {
          "recallLimit": 5
        }
      }
    }
  }
}
```

### 替换内置记忆插件（可选）

OpenClaw 默认使用内置的 `memory-core` 插件管理 `MEMORY.md` / `memory/YYYY-MM-DD.md` 记忆文件。如果你希望完全用 MemOS 替代内置记忆，可以通过 `plugins.slots.memory` 切换：

```jsonc
{
  "plugins": {
    "slots": {
      "memory": "memos-cloud-openclaw-plugin"
    }
  }
}
```

> **注意**：仅当 MemOS 插件明确实现了 OpenClaw `memory` 插件槽的接口时才可这样配置，否则保持默认（不设置该项）即可。建议先不修改此项，确认插件基本功能正常后再考虑。

---

## 已修正的原指南内容汇总

| 原指南内容 | 实际情况（基于源码） |
|---|---|
| 插件读取 `~/.openclaw/.env`、`~/.moltbot/.env`、`~/.clawdbot/.env` 三个文件 | OpenClaw **只读取** `~/.openclaw/.env`，其他两个路径是历史遗留的配置目录，不作为 dotenv 文件加载（`src/config/state-dir-dotenv.ts`） |
| `openclaw plugins install <package>` 作为终端命令 | 该 CLI 命令**不存在**；插件安装通过向 agent 发送 `/plugins install <spec>` 消息完成（`src/auto-reply/reply/commands-plugins.ts`） |
| Windows 用户使用方案 B 手动安装 | 本指南面向本地 Mac mini，Windows 相关内容已移除 |
