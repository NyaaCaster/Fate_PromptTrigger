# NyaaFrame — Claude Code 项目规则

## 项目概述

NyaaFrame 是一个标准开发环境模板项目，用于快速启动 WebApp 和轻量服务开发。

- 堆栈：Nginx + Vite + React + TypeScript + Tailwind
- 容器化：Docker（多阶段构建，最小体积）
- 仓库：https://github.com/NyaaCaster/NyaaFrame.git
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
- 私有镜像仓库：`h.hony-wen.com:5000`（HTTP，无认证）
- 推送流程：`docker tag <local>:<tag> h.hony-wen.com:5000/<name>:<tag>` → `docker push`

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

## 模板项目初始化流程

当基于此模板创建新项目时，按以下步骤执行：

1. 确认项目名称和开发目的
2. 更新 `meta.json`（项目唯一元数据来源）
3. 确定项目技术堆栈，拉取对应依赖
4. 更新 Docker 配置（从 meta.json 读取容器名和端口）
5. 根据实际需求修改 `rebuild.ps1` 脚本
6. 确定 GitHub 仓库地址（从 meta.json 读取）
7. 修改 LICENSE 文件签名和地址（从 meta.json 读取）
8. 更新 `src/version.ts` 的 `APP_NAME`
9. 重新生成 README
10. 建立 `.ref/` 目录（已被 .gitignore 排除版本管理）
11. 清理初始化模板流程和 Claude Code 备忘
12. 初始化 GitHub 仓库并进行首次推送
13. 通知开始进行开发

### meta.json 说明

`meta.json` 是项目的标准化元数据文件，记录项目名称和特征信息。初始化流程中所有需要项目名称、地址、端口等信息的步骤均从此文件读取，确保单一来源、全局一致。
