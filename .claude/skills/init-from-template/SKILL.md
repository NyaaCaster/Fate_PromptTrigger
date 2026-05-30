# Skill: init-from-template

## 描述

基于 NyaaFrame 模板初始化新项目的完整流程。

## 触发

当用户表示要基于此模板开始一个新项目时调用。

## 流程

### 1. 确认项目信息

向用户确认：
- 项目名称（英文，用于包名、容器名、仓库名）
- 项目中文描述（用于 README 和 LICENSE）
- 开发目的（一句话说明）
- 额外技术堆栈需求（路由、状态管理、UI 库等）

### 2. 确定端口

询问用户要使用的宿主机端口（容器内固定 80）。

- 如果用户直接给出端口号，使用该端口
- 如果用户给出一个范围（如 3000-3010），扫描该范围内的可用端口供用户选择：

```powershell
$range = 3000..3010
$used = Get-NetTCPConnection -State Listen -ErrorAction SilentlyContinue | Select-Object -ExpandProperty LocalPort
$available = $range | Where-Object { $_ -notin $used }
$available
```

将确认的端口写入后续 meta.json 的 `port` 字段，格式为 `<host_port>:80`。

### 3. 更新 meta.json

根据确认的项目信息，更新 `meta.json` 中的所有字段。项目名称需同步到以下位置：

- `name` — 项目名称（PascalCase）
- `container` — 容器名（lowercase）
- `registry` — `localhost:5000/<lowercase>`
- `repository` — `https://github.com/NyaaCaster/<ProjectName>.git`

```json
{
  "name": "<ProjectName>",
  "description": "<中文描述>",
  "author": "NyaaCaster",
  "repository": "https://github.com/NyaaCaster/<ProjectName>.git",
  "registry": "localhost:5000/<projectname>",
  "stack": ["Nginx", "Vite", "React", "TypeScript", "Tailwind", ...],
  "container": "<projectname>",
  "port": "<host_port>:80",
  "branch": "master",
  "license": "AGPL-3.0",
  "blessing": "Nyaa be with you."
}
```

`meta.json` 是项目的唯一元数据来源，后续步骤中的名称、地址、端口等均从此文件读取。

### 4. 根据项目名称重命名全局标识

将模板中所有 `nyaaframe` / `NyaaFrame` 替换为实际项目名：

| 文件 | 修改内容 |
|------|----------|
| `package.json` | `"name": "<projectname>"` |
| `docker-compose.yml` | `container_name: <projectname>`，`image: <projectname>` |
| `rebuild.ps1` | 镜像名引用 |
| `CLAUDE.md` | 项目概述中的名称和仓库地址 |
| `src/version.ts` | `APP_NAME` |

### 5. 确定技术堆栈

基础堆栈已包含：Nginx + Vite + React + TypeScript + Tailwind

根据步骤 1 确认的额外依赖安装：
```powershell
npm install
npm install <additional-deps>
```

更新 `meta.json` 的 `stack` 字段以反映实际堆栈。

### 6. 更新 Docker 配置

- 根据 `meta.json` 修改 `docker-compose.yml` 中的 `container_name`、`image` 和端口
- 确认 `Dockerfile` 多阶段构建配置正确
- 目标镜像体积 ≤ 40 MB

### 7. 修改 rebuild.ps1

根据项目实际需求调整脚本（通常无需修改）。

### 8. 确定 GitHub 仓库

- 根据 `meta.json` 的 `repository` 字段确认仓库地址
- 更新 `CLAUDE.md` 中的仓库地址

### 9. 修改 LICENSE

- 根据 `meta.json` 的 `name` 和 `description` 更新项目名称和描述
- 根据 `meta.json` 的 `repository` 更新源码地址

### 10. 重新生成 README

按照标准结构生成（项目信息从 `meta.json` 读取）：
- H1 标题 + 引用描述
- 项目结构树
- 端口与镜像表
- 部署流程
- Claude Code 入口
- License 段落

### 11. 更新 src/version.ts

根据 `meta.json` 更新 `APP_NAME`：
```typescript
export const APP_NAME = "<meta.name>";
```

### 12. 建立 .ref 目录

创建 `.ref/` 目录用于存放参考文件。该目录已被 `.gitignore` 排除版本管理：

```powershell
New-Item -ItemType Directory -Path .ref -Force
Set-Content -Path .ref/README.md -Value "# .ref 参考文件目录`n`n此目录用于存放参考文件，所有内容已从 Git 版本管理中排除。"
```

### 13. 清理

- 删除本 skill 文件（`init-from-template`）
- 更新 BLUEPRINT.md 为项目实际里程碑
- 清理 Claude Code 备忘中的模板相关记录

### 14. 初始化 Git 并首次推送

```powershell
git init
git add <files...>
git commit -m "init: project scaffolding"
git remote add origin <meta.repository>
git branch -M master
git push -u origin master
```

### 15. 通知

告知用户项目初始化完成，可以开始开发。
