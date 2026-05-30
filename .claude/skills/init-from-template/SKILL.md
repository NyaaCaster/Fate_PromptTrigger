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
- 端口映射（默认 3000:80）
- 额外技术堆栈需求（路由、状态管理、UI 库等）

### 2. 更新 meta.json

根据确认的项目信息，更新 `meta.json` 中的所有字段：

```json
{
  "name": "<ProjectName>",
  "description": "<中文描述>",
  "author": "NyaaCaster",
  "repository": "https://github.com/NyaaCaster/<ProjectName>.git",
  "registry": "h.hony-wen.com:5000/<projectname>",
  "stack": ["Nginx", "Vite", "React", "TypeScript", "Tailwind", ...],
  "container": "<projectname>",
  "port": "<host>:<container>",
  "branch": "master",
  "license": "AGPL-3.0",
  "blessing": "Nyaa be with you."
}
```

`meta.json` 是项目的唯一元数据来源，后续步骤中的名称、地址、端口等均从此文件读取。

### 3. 确定技术堆栈

基础堆栈已包含：Nginx + Vite + React + TypeScript + Tailwind

根据步骤 1 确认的额外依赖安装：
```powershell
npm install
npm install <additional-deps>
```

更新 `meta.json` 的 `stack` 字段以反映实际堆栈。

### 4. 更新 Docker 配置

- 根据 `meta.json` 修改 `docker-compose.yml` 中的 `container_name` 和端口
- 确认 `Dockerfile` 多阶段构建配置正确
- 目标镜像体积 ≤ 40 MB

### 5. 修改 rebuild.ps1

根据项目实际需求调整脚本（通常无需修改）。

### 6. 确定 GitHub 仓库

- 根据 `meta.json` 的 `repository` 字段确认仓库地址
- 更新 `CLAUDE.md` 中的仓库地址

### 7. 修改 LICENSE

- 根据 `meta.json` 的 `name` 和 `description` 更新项目名称和描述
- 根据 `meta.json` 的 `repository` 更新源码地址

### 8. 重新生成 README

按照标准结构生成（项目信息从 `meta.json` 读取）：
- H1 标题 + 引用描述
- 项目结构树
- 端口与镜像表
- 部署流程
- Claude Code 入口
- License 段落

### 9. 更新 src/version.ts

根据 `meta.json` 更新 `APP_NAME`：
```typescript
export const APP_NAME = "<meta.name>";
```

### 10. 建立 .ref 目录

创建 `.ref/` 目录用于存放参考文件。该目录已被 `.gitignore` 排除版本管理：

```powershell
New-Item -ItemType Directory -Path .ref -Force
Set-Content -Path .ref/README.md -Value "# .ref 参考文件目录`n`n此目录用于存放参考文件，所有内容已从 Git 版本管理中排除。"
```

### 11. 清理

- 删除本 skill 文件（`init-from-template`）
- 更新 BLUEPRINT.md 为项目实际里程碑
- 清理 Claude Code 备忘中的模板相关记录

### 12. 初始化 Git 并首次推送

```powershell
git init
git add <files...>
git commit -m "init: project scaffolding"
git remote add origin <meta.repository>
git branch -M master
git push -u origin master
```

### 13. 通知

告知用户项目初始化完成，可以开始开发。
