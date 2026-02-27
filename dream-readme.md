# 🌙 Dream — OpenClaw 记忆蒸馏技能

> 主动维护你的 `MEMORY.md`，让 OpenClaw 真正记住你。

---

## 它解决什么问题

OpenClaw 原生的记忆机制有三个未被处理的缺陷：

**1. MEMORY.md 只增不减，会静默截断**
OpenClaw 在 20,000 字符处截断 MEMORY.md，不报错、不提示，AI 安静地丢失后半段上下文。你以为它记得，它其实已经不知道了。Dream 在 18,000 字符时主动触发压缩，将过期内容归档，确保 MEMORY.md 始终在有效范围内。

**2. 没有永久档案**
重要的记忆被清理后就消失了。Dream 维护一个只追加、永不删除的 `ledger.md`，凡是进入过长期记忆的内容，哪怕后来被遗忘，历史记录永远保留，支持深度回溯检索。

**3. 没有 Re-emergence 检测**
你主动清除了某条记忆，但它后来又在对话中出现——这说明它比你以为的更重要。Dream 检测这种"遗忘后再现"的模式，自动将其重新写入记忆并提升优先级。

---

## 工作原理

```
memory/YYYY-MM-DD.md        ← OpenClaw 原生自动写入（compaction flush）
         │
         │  每日 03:30，Dream 读取并蒸馏
         ▼
     MEMORY.md               ← Dream 主动维护，硬上限 18,000 字符
         │
         │  重要但已过期的内容
         ▼
     ledger.md               ← 永久档案，只追加，永不删除
```

**Dream 是蒸馏者，不是捕获者。**
捕获由 OpenClaw 原生的 compaction flush 完成，搜索由原生 `openclaw memory search` 完成。Dream 只做原生没做的事：蒸馏、压缩、归档、Re-emergence 检测。

---

## 文件结构

安装后会创建以下文件：

```
~/.openclaw/workspace/
└── MEMORY.md                    ← Dream 主动维护（原有文件，Dream 接管）

~/Documents/Obsidian/dream-vault/   ← 由 DREAM_VAULT_PATH 配置
├── ledger.md                    ← 永久档案
├── ledger-index.json            ← 档案结构化索引
├── meta/
│   ├── active-days.json         ← 活跃天记录
│   ├── last-review.txt          ← 上次蒸馏时间
│   ├── dream-state.txt          ← 系统状态
│   ├── removed-entries.json     ← Re-emergence 追踪
│   └── review-YYYY-MM.md        ← 每月蒸馏日志
└── obsidian-index/
    ├── _index.md                ← 内容主索引
    └── topics/<topic>.md        ← 按主题分类的内容索引
```

---

## 安装

### 前置要求

- OpenClaw（Node ≥ 22，Gateway 端口 18789）
- `jq`（JSON 处理）
- `wc`（字符计数，系统内置）
- `md5sum` 或 `md5`（哈希，系统内置）

```bash
# macOS
brew install jq

# Linux
apt install jq
```

### 安装步骤

```bash
# 1. 克隆到 OpenClaw skills 目录
cd ~/.openclaw/workspace/skills
git clone https://github.com/<your-username>/dream-skill dream

# 2. 赋予脚本执行权限
chmod +x dream/dream-tools.sh

# 3. 配置 vault 路径（加入 shell 配置文件）
echo 'export DREAM_VAULT_PATH="$HOME/Documents/Obsidian/dream-vault"' >> ~/.zshrc
source ~/.zshrc

# 4. 初始化
./dream/dream-tools.sh --init

# 5. 重启 OpenClaw gateway，让技能生效
openclaw gateway restart
```

### 配置定时蒸馏

在你的 `SOUL.md` 末尾加入：

```markdown
## Dream 定时任务
每天 03:30 执行 dream review --scheduled
```

---

## 使用方式

### 自动运行（推荐）

安装后无需任何操作。Dream 每日 03:30 自动蒸馏，静默运行，不打扰你。

### 手动触发

在 OpenClaw 对话中直接说：

| 你说什么 | Dream 做什么 |
|---------|------------|
| `dream` / `复盘` | 立即执行蒸馏 |
| `整理记忆` | 立即执行蒸馏 |
| `你记得我什么` | 输出当前 MEMORY.md 快照 |
| `dream status` | 查看系统状态 |
| `dream search <关键词>` | 搜索 memory + 永久档案 + Obsidian 索引 |
| `dream index <内容>` | 将文章/网页收录到 Obsidian 索引 |
| `dream forget <描述>` | 从记忆中清除（档案保留） |
| `dream init` | 首次使用引导 |

---

## `dream-tools.sh` 是什么

这是 Dream 的执行引擎。**AI 负责语义判断，脚本负责精确执行。**

把精确计算和文件操作交给脚本而不是 AI，原因：AI 做日期差运算容易出错，文件原子写入需要 shell 级别的 `mv` 保证，字符数统计需要 `wc` 而不是 AI 估算。

脚本提供的能力：

| 命令 | 作用 |
|------|------|
| `--check-idle` | 检测 OpenClaw 是否空闲（超时 5 秒默认返回 busy）|
| `--check-size` | 返回 MEMORY.md 字符数和压缩状态 |
| `--hash` | 生成 8 位短哈希，用于条目 ID 和去重 |
| `--atomic-write` | 原子替换文件，MEMORY.md 额外做字符数校验 |
| `--ledger-append` | 向永久档案追加记录，同步更新索引 |
| `--ledger-search` | 在档案索引中关键词检索 |
| `--ledger-mark-reemergence` | 标记条目为"遗忘后再现" |
| `--record-removed` | 记录从 MEMORY.md 移除的条目（Re-emergence 追踪）|
| `--check-reemergence` | 检测新内容是否与曾被遗忘的条目相似 |
| `--record-active-day` | 记录今日为活跃天（去重） |
| `--active-days-since` | 计算指定日期后的活跃天数 |
| `--dedup-index` | 检查 Obsidian 索引是否已有该 URL |
| `--init` | 初始化 vault 目录结构 |
| `--status` | 输出系统状态摘要（低 IO，只读 meta）|

---

## 与其他 memory 技能的关系

Dream **不与现有 memory 技能冲突**，而是填补空白：

- **rosepuppy/memory-complete**：负责实时捕获，写入 `memory/YYYY-MM-DD.md`
  → Dream 读取这些文件做蒸馏，两者职责互补，可以共存

- **openclaw 原生 memory search**：负责日常搜索
  → Dream 的 `dream search` 调用原生命令，不自建索引

- **单独使用 Dream**：OpenClaw 原生的 compaction flush 已经会写 `memory/YYYY-MM-DD.md`，无需安装额外技能

---

## MEMORY.md 结构

Dream 将 MEMORY.md 维护为四个区块：

```markdown
## 当前状态
正在进行的项目、未完成的决策、近期重要变化

## 稳定认知
技术栈、工作环境、决策风格、核心偏好

## 关系与背景
重要的人、正在进行的合作

## Dream
（自动维护）永久档案最近 5 条入档摘要
```

---

## 安全说明

- 所有文件操作严格限制在 `DREAM_VAULT_PATH` 和 OpenClaw workspace 内
- `dream-tools.sh` 不联网，不调用任何外部 API
- `dream forget` 只清除活跃记忆，永久档案不受影响
- 原子写入（tmp → mv）防止崩溃导致文件损坏

---

## License

MIT
