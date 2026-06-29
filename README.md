# Agent Skills Collection

适用于**权限受限的容器化智能体**的 Skills 集合。

## 为什么需要这类 Skills？

许多企业级 AI Agent 采用容器化部署，运行在**严格隔离的沙箱环境**中：

- **无本地浏览器** — 无法直接使用 Puppeteer/Playwright 本地模式
- **无系统级权限** — 不能安装软件包、修改系统配置
- **网络受限** — 只能访问白名单域名或特定 API
- **文件系统隔离** — 工作目录受限，无法访问宿主机

这类 Agent 需要依赖**远程服务**（如 Browserless）来执行浏览器操作，而非本地安装。

本仓库的 Skills 专为这种场景设计，所有依赖的外部服务均通过**环境变量配置**，不依赖本地安装或硬编码地址。

## 目录结构

```
agent-skills/
├── skills/                     # Skills 目录（智能体指令）
│   └── manual-generate-fudan/  # 复旦大学业务系统用户手册生成
│       └── SKILL.md
├── docs/                       # 用户文档（部署/配置指南）
│   └── manual-generate-fudan/
│       └── browserless-setup.md
└── README.md
```

**说明：**
- `skills/` — 给智能体看的指令文件，会被 Agent 按需加载
- `docs/` — 给用户看的配置文档，Agent 不读取

## 已有 Skills

| Skill | 适用场景 | 外部依赖 |
|---|---|---|
| `manual-generate-fudan` | 复旦大学业务系统用户手册生成（ehall、教务、研究生等） | Browserless（远程浏览器服务） |
| `container-pip-bootstrap` | 容器内引导安装 pip 和 Python 库（非 root、无 sudo 环境） | 无（仅需 curl/wget + python3） |

### manual-generate-fudan

探索业务系统并生成功能覆盖清单和用户手册：
- 通过复旦 UIS 统一身份认证登录
- 使用 Browserless 远程浏览器探索系统界面
- 输出 Markdown/Word 手册 + 截图打包

**配置要求：**
```bash
export BROWSERLESS_URL=http://your-server:3000  # 必填
export BROWSERLESS_TOKEN=your-token             # 可选，如果服务需要认证
```

详细配置参考：[docs/manual-generate-fudan/browserless-setup.md](docs/manual-generate-fudan/browserless-setup.md)

### container-pip-bootstrap

在权限受限的容器中引导安装 pip 和 Python 库：
- 使用 `get-pip.py` 引导安装 pip（无需 root/sudo）
- 使用 `--break-system-packages` 绕过 PEP 668 限制
- 使用 `--user` 安装到用户目录
- 包含常用库安装指南（python-docx、openpyxl、pdfplumber 等）

**适用环境：**
- 非 root 用户运行的容器（如 OpenClaw 的 `node` 用户）
- 无 sudo / apt-get 权限
- Python 存在但 pip 缺失，或 pip 安装报 `externally-managed-environment`

**无需外部依赖**，仅需容器内有 `curl` 或 `wget` + `python3`。

## 如何使用

### 方式一：直接引用 SKILL.md

在对话中引用 skill 文件：

```
@skills/manual-generate-fudan/SKILL.md
帮我生成 ehall.fudan.edu.cn 的用户手册
```

### 方式二：复制到 Agent 的 skills 目录

将 `skills/<skill-name>/` 目录复制到你的 Agent 工作区：

```
your-agent-workspace/skills/manual-generate-fudan/SKILL.md
```

### 方式三：安装到 OpenClaw/Trae/Claude Code

根据你使用的 Agent 平台，将 Skills 安装到对应目录：

| 平台 | Skills 目录 |
|---|---|
| OpenClaw | `<workspace>/skills/` 或 `~/.openclaw/skills/` |
| Trae | `~/.trae/skills/` 或 `<project>/.trae/skills/` |
| Claude Code | `~/.claude/skills/` |

## 设计原则

本仓库的 Skills 遵循以下设计原则：

1. **外部服务通过环境变量配置** — 不硬编码地址、Token、API Key
2. **Frontmatter 声明所有依赖** — 使用 `metadata.openclaw.envVars` 明确标注必填/选填变量
3. **无本地安装依赖** — 不依赖 `npm install`、`apt-get` 等系统级操作
4. **工作目录落盘** — 复杂任务边执行边写入文件，不依赖上下文记忆
5. **风险操作明确标记** — 删除、提交、审批等高风险操作需用户确认

## 配置外部服务

Skills 依赖的外部服务需要在部署 Agent 的环境（容器/服务器）中预先配置：

| 服务 | 用途 | 配置文档 |
|---|---|---|
| Browserless | 远程浏览器服务 | [browserless-setup.md](docs/manual-generate-fudan/browserless-setup.md) |

**Browserless 快速部署：**
```bash
docker run -d -p 3000:3000 ghcr.io/browserless/chromium
```

然后在 Agent 环境设置：
```bash
export BROWSERLESS_URL=http://your-server-ip:3000
```

## 贡献新的 Skill

欢迎贡献适用于容器化 Agent 的 Skills：

1. Fork 本仓库
2. 在 `skills/` 下创建新目录
3. 编写 `SKILL.md`（遵循 AgentSkills 规范）
4. 在 `docs/` 下添加配置文档（如果依赖外部服务）
5. 提交 Pull Request

**Skill 要求：**
- 所有外部依赖通过环境变量配置
- Frontmatter 正确声明 `envVars`
- 不依赖本地软件安装
- 提供清晰的配置文档

## 相关资源

- [AgentSkills 规范](https://agentskills.io/)
- [OpenClaw Skills 文档](https://docs.openclaw.ai/tools/skills)
- [Trae Skills 最佳实践](https://docs.trae.ai/ide/best-practice-for-how-to-write-a-good-skill)
- [Browserless 官方文档](https://docs.browserless.io/)