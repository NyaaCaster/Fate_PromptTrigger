# NyaaFrame

> 标准开发环境模板 — 用于快速启动 WebApp 和轻量服务开发

## 技术堆栈

Nginx + Vite + React + TypeScript + Tailwind

## 项目结构

```
NyaaFrame/
├── .claude/skills/         # Claude Code 技能定义
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
├── Dockerfile              # 多阶段构建（node-alpine → nginx-alpine）
├── docker-compose.yml      # 容器编排
├── index.html              # HTML 入口
├── meta.json               # 项目元数据（名称、堆栈、端口等）
├── nginx.conf              # Nginx 配置
├── package.json            # 依赖管理
├── rebuild.ps1             # Docker 构建重启脚本
├── tsconfig.json           # TypeScript 配置
└── vite.config.ts          # Vite 配置
```

## 端口与镜像

| 服务 | 容器名 | 端口 | 镜像仓库 |
|------|--------|------|----------|
| app | nyaaframe | 3000:80 | h.hony-wen.com:5000/nyaaframe |

## 使用方式

### 作为模板创建新项目

1. 复制本项目到新目录
2. 使用 Claude Code 执行 `init-from-template` skill
3. 按提示完成项目信息配置

### 本地开发

```bash
npm install
npm run dev
```

### Docker 部署

```powershell
.\rebuild.ps1           # 使用缓存构建
.\rebuild.ps1 -NoCache  # 无缓存构建
```

## Claude Code 入口

会话开始时必读：

| 文件 | 用途 |
|------|------|
| `CLAUDE.md` | 项目规则与约定 |
| `.docs/BLUEPRINT.md` | 任务蓝图与进度 |

Skills：

| 技能 | 用途 |
|------|------|
| `rebuild` | Docker 构建重启 |
| `commit-push` | Git 提交推送 |
| `sync-blueprint` | 蓝图维护 |
| `init-from-template` | 模板初始化新项目 |

## License

AGPL-3.0 — 详见 [LICENSE](LICENSE)
