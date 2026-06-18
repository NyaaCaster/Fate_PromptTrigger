# Fate_PromptTrigger

> 一场基于 LLM 的圣杯战争。

## 技术堆栈

Nginx + Vite + React + TypeScript + Tailwind

## 项目结构

```text
Fate_PromptTrigger/
├── .docs/                  # 项目文档
│   └── BLUEPRINT.md        # 开发任务蓝图
├── .ref/                   # 参考文件（不纳入版本管理）
├── src/
│   ├── App.tsx             # 根组件
│   ├── main.tsx            # 入口文件
│   ├── index.css           # 全局样式（Tailwind）
│   ├── version.ts          # 版本与代码签名
│   └── vite-env.d.ts       # Vite 类型声明
├── CLAUDE.md               # Claude Code 项目规则
├── Dockerfile              # 多阶段构建（node-alpine -> nginx-alpine，BuildKit 缓存）
├── docker-compose.yml      # 容器编排
├── index.html              # HTML 入口
├── LICENSE                 # AGPL-3.0 许可证
├── meta.json               # 项目元数据（名称、堆栈、端口等）
├── nginx.conf              # Nginx 配置
├── package.json            # 依赖管理
├── postcss.config.js       # PostCSS 配置（Tailwind）
├── rebuild.ps1             # Docker 构建重启脚本
├── tsconfig.json           # TypeScript 配置
└── vite.config.ts          # Vite 配置
```

## 端口与镜像

| 服务 | 容器名 | 镜像 | 端口 |
|------|--------|------|------|
| client | fateprompttrigger | fateprompttrigger:latest | 5111:80 |
| registry | — | localhost:5000/fateprompttrigger | — |

> `meta.json` 是项目元数据单一来源；端口与镜像信息应与该文件保持一致。

## 部署流程

```powershell
npm install
npm run build
.\rebuild.ps1           # 使用缓存构建并重启
.\rebuild.ps1 -NoCache  # 无缓存构建并重启
```

## Claude Code 入口

会话开始时必读：

| 文件 | 用途 |
|------|------|
| `CLAUDE.md` | 项目规则与约定 |
| `.docs/BLUEPRINT.md` | 任务蓝图与进度 |

常用 Skills：

| 技能 | 用途 |
|------|------|
| `rebuild` | Docker 构建重启 |
| `commit-push` | Git 提交推送 |
| `sync-blueprint` | 蓝图维护 |

## License

AGPL-3.0 — 详见 [LICENSE](LICENSE)
