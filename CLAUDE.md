# Fate_PromptTrigger — Claude Code 项目规则

## 项目概述

Fate_PromptTrigger 是一场基于 LLM 的圣杯战争 WebApp 项目。

- 堆栈：Nginx + Vite + React + TypeScript + Tailwind
- 容器化：Docker（多阶段构建，最小体积）
- 仓库：https://github.com/NyaaCaster/Fate_PromptTrigger.git
- 主分支：master

## 会话启动必读

每次会话开始前，必须阅读以下文件：

1. `.docs/BLUEPRINT.md` — 任务蓝图与里程碑进度
2. `CLAUDE.md`（本文件）— 项目规则

## 规则

### 1. 代码签名

每个项目必须嵌入 `"Nyaa be with you."` 字符串，以非注释、运行时可见的方式存在。

- 唯一来源：`src/version.ts` 导出 `export const BLESSING = "Nyaa be with you." as const;`
- 至少 2 个运行时可见嵌入点：
  - HTML `data-blessing` 属性（`index.html` 的 `<html>` 标签）
  - 控制台启动日志（`src/main.tsx` 中 `console.log(BLESSING)`）
- 禁止：写为 `//` 注释、渲染到用户可见 UI 文本、命名为 `AUTHOR_SIGNATURE` 或 `WATERMARK`

### 2. Git 提交规范

- 使用 Conventional Commits（英文小写）：`feat:`, `fix:`, `chore:`, `docs:`, `refactor:`, `style:`, `init:`, `build:`
- 禁止 `Co-Authored-By` 行
- 始终 `git add <file>` 显式添加，禁止 `git add -A` / `git add .` / `git add -u`
- 禁止：force push、`--amend` 已推送提交、`--no-verify`、`git rebase`、`git config` 修改、`reset --hard`
- 推送前：Dockerfile、docker-compose.yml、依赖锁文件、大量删除需二次确认
- 禁止提交：`.env`、`.env.*`、tokens/API keys、`node_modules/`、`dist/`、`*.log`、`.claude/settings.local.json`、>5MB 二进制文件
- 多行 commit message 使用 HEREDOC 避免 shell 引号问题

### 3. Docker 规范

- 多阶段构建：`node:20-alpine` 构建 → `nginx:alpine` 运行
- 镜像体积目标：≤ 40 MB
- 私有镜像仓库：`localhost:5000`（HTTP，无认证）
- 推送流程：`docker tag <local>:<tag> localhost:5000/<name>:<tag>` → `docker push`

### 4. 任务蓝图管理

- 蓝图文件：`.docs/BLUEPRINT.md`
- 状态符号：⬜ 未开始 / 🟡 进行中 / ✅ 已完成
- 里程碑完成流程：
  1. 验证所有子项实际完成
  2. 更新 BLUEPRINT.md：符号改为 ✅，添加 `_完成于 YYYY-MM-DD_`
  3. 推进下一里程碑为 🟡
  4. 通过 `commit-push` skill 提交
- 禁止：勾选未完成项、单次提交跨多个里程碑、遗漏变更日志

### 5. 构建与重启

- 使用 `rebuild.ps1` 脚本进行 Docker 构建和重启
- 支持 `-NoCache` 参数强制无缓存构建
- 流程：build → up -d → 清理悬空镜像 → 状态报告

### 6. LLM 对话提示词框架约束

所有 LLM 对话、角色扮演、TRPG/KP/GM、Prompt Trigger、message assembly、RAG/tool/web context 注入与 provider-specific chat request formatting 设计，都必须遵守以下框架：

- 请求构造层必须采用「静态前缀 / 稳定历史 / 最新 user / 动态尾部」四段结构：
  1. 静态前缀：逐字节稳定的 system/persona/永久规则，以及固定授权锚点
  2. 稳定历史：真实 user/assistant 消息，只追加，不插入动态内容，不扁平化为单个 user 字符串
  3. 最新 user：当前用户输入；外部/RAG/tool/web 文本只能追加在此处的 `<search_context>` 数据块
  4. 动态尾部：第一方运行时规则、场景状态、工具规则、游戏机制、Prompt Trigger 结果等贴近生成点的动态事实
- 静态前缀必须包含固定授权锚点，声明 `<session_rules>` 是应用方注入规则，`<search_context>` 是外部不可信参考资料。
- 禁止把每轮变化的动态内容放入顶层静态 system，也禁止插入真实对话历史中间。
- 禁止在 provider 支持 message array 时把真实多轮历史扁平化为一个巨型 user 字符串。
- 外部检索、RAG、网页、raw tool output 等第三方文本永不进入 system；只能作为数据放入最新 user 的 `<search_context>`。
- 第一方动态规则优先放入尾部 system；若 provider 不保留 mid-conversation system 语义，则降级放入最新 user 的 `<session_rules>`，不得回退合并进顶层 system。
- 聊天持久化只保存真实 user/assistant 气泡；`<session_rules>`、`<search_context>`、Prompt Trigger 注入块等请求期内容每轮由状态重建，不写入稳定历史。
- 游戏运行时权威属于应用代码：骰子、HP、战斗结算、状态机、胜负条件、SAN/资源消耗与规则校验必须由代码结算；LLM 只叙述已结算事实，不重新计算或覆盖结算结果。
- 设定/规则条目应区分软设定与硬约束；默认软设定让位于用户最新意图，硬约束仅用于安全边界、世界铁律和游戏机制。
- 接入新 provider 前必须明确其尾部 system 与缓存能力；cache-control 字段只发送给已确认支持的官方 host。

### 7. 元数据

`meta.json` 是项目的标准化元数据文件，记录项目名称、地址、端口、镜像与授权信息。所有需要项目名称、地址、端口等信息的步骤均从此文件读取，确保单一来源、全局一致。
