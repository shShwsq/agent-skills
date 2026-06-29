---
name: container-pip-bootstrap
description: 在权限受限的容器化环境中引导安装 Python pip 和第三方库。适用于非 root 用户、无 sudo、无 apt-get 的容器（如 OpenClaw、Node.js 基础镜像）。使用 get-pip.py 引导 pip，使用 --break-system-packages 绕过 PEP 668 限制。当用户需要在容器内安装 Python 库、遇到 "externally-managed-environment" 错误、或需要设置 Python 工具链时使用此技能。
---

# 容器内 Python pip 引导与库安装

本技能用于在权限受限的容器化环境中安装 Python 包管理器 pip 和第三方 Python 库。

## 适用环境

本技能适用于以下特征的容器：

- **非 root 用户运行**（如 `node` 用户、UID 1000）
- **无 sudo / su 权限**
- **无 apt-get / yum / apk** 等系统包管理器
- **Python 可能存在但 pip 缺失**
- **遇到 PEP 668 限制**（`externally-managed-environment` 错误）

典型场景：OpenClaw 容器、Node.js 基础镜像、Docker `cap_drop: ALL` 的沙箱容器。

## 核心原则

1. **不依赖系统包管理器** — 不使用 `apt-get`、`yum`、`apk`
2. **不依赖 root 权限** — 不使用 `sudo`
3. **优先使用用户级安装** — `pip install --user` 或 `pip install --break-system-packages`
4. **失败时记录原因** — 安装失败时明确记录错误信息，不要静默跳过
5. **安装后验证** — 每次安装后验证是否成功

## 第一阶段：环境检测

安装任何 Python 库前，先检测当前环境。

### 1.1 检查 Python 版本

```bash
python3 --version 2>/dev/null || python --version 2>/dev/null
```

记录主版本号（如 3.11、3.12），后续安装命令需匹配。

### 1.2 检查 pip 是否可用

```bash
pip3 --version 2>/dev/null || pip --version 2>/dev/null
```

- 如果返回版本号 → pip 已可用，跳到 [第三阶段：安装 Python 库](#第三阶段安装-python-库)
- 如果报错或无输出 → 需要引导安装 pip，继续第二阶段

### 1.3 检查 curl 或 wget

```bash
curl --version 2>/dev/null | head -1 || wget --version 2>/dev/null | head -1
```

至少需要一个来下载 `get-pip.py`。如果两者都没有，标记为"环境限制，无法引导 pip"。

### 1.4 检查当前用户

```bash
whoami && id
```

确认当前用户（预期非 root，如 `node`）。

## 第二阶段：引导安装 pip

使用官方 `get-pip.py` 脚本安装 pip。这是最可靠的非 root 安装方式。

### 2.1 下载 get-pip.py

```bash
# 优先使用 curl
curl -sSL https://bootstrap.pypa.io/get-pip.py -o /tmp/get-pip.py

# 如果 curl 不可用，使用 wget
wget -qO /tmp/get-pip.py https://bootstrap.pypa.io/get-pip.py
```

### 2.2 执行安装

```bash
python3 /tmp/get-pip.py --user
```

- `--user` 标志将 pip 安装到用户目录（`~/.local/bin`），不需要 root 权限
- 安装后 pip 位于 `~/.local/bin/pip3`

### 2.3 将 pip 加入 PATH

```bash
# 检查 ~/.local/bin 是否在 PATH 中
echo $PATH | grep -q "\.local/bin" || export PATH="$HOME/.local/bin:$PATH"

# 验证 pip 是否可用
pip3 --version
```

如果 `pip3 --version` 仍不可用，手动指定完整路径：

```bash
~/.local/bin/pip3 --version
```

### 2.4 清理安装脚本

```bash
rm -f /tmp/get-pip.py
```

## 第三阶段：安装 Python 库

### 3.1 处理 PEP 668 限制

新版 Python（3.11+）在系统级安装时会报错：

```
error: externally-managed-environment
× This environment is externally managed
```

**解决方法：** 使用 `--break-system-packages` 标志：

```bash
pip3 install --user --break-system-packages <package-name>
```

### 3.2 标准安装命令模板

```bash
# 确保 PATH 包含用户 bin 目录
export PATH="$HOME/.local/bin:$PATH"

# 安装单个包
pip3 install --user --break-system-packages <package-name>

# 安装多个包
pip3 install --user --break-system-packages <pkg1> <pkg2> <pkg3>

# 安装指定版本
pip3 install --user --break-system-packages <package-name>==1.2.3
```

### 3.3 验证安装

```bash
# 验证包是否可导入
python3 -c "import <package_name>; print(<package_name>.__version__)"
```

## 第四阶段：常用库安装指南

以下是容器化环境中常用的 Python 库及其安装注意事项。

### 文档处理类

| 库 | 用途 | 安装命令 | 验证 |
|---|---|---|---|
| `python-docx` | Word (.docx) 生成 | `pip3 install --user --break-system-packages python-docx` | `python3 -c "import docx"` |
| `openpyxl` | Excel (.xlsx) 读写 | `pip3 install --user --break-system-packages openpyxl` | `python3 -c "import openpyxl"` |
| `python-pptx` | PowerPoint (.pptx) 生成 | `pip3 install --user --break-system-packages python-pptx` | `python3 -c "import pptx"` |
| `pdfplumber` | PDF 文本提取 | `pip3 install --user --break-system-packages pdfplumber` | `python3 -c "import pdfplumber"` |
| `PyPDF2` | PDF 合并/拆分 | `pip3 install --user --break-system-packages PyPDF2` | `python3 -c "import PyPDF2"` |
| `reportlab` | PDF 生成 | `pip3 install --user --break-system-packages reportlab` | `python3 -c "import reportlab"` |

### 网络请求类

| 库 | 用途 | 安装命令 |
|---|---|---|
| `requests` | HTTP 请求 | `pip3 install --user --break-system-packages requests` |
| `httpx` | 异步 HTTP 请求 | `pip3 install --user --break-system-packages httpx` |
| `websockets` | WebSocket 客户端 | `pip3 install --user --break-system-packages websockets` |

### 数据处理类

| 库 | 用途 | 安装命令 |
|---|---|---|
| `pandas` | 数据分析 | `pip3 install --user --break-system-packages pandas` |
| `openpyxl` | Excel 读写 | `pip3 install --user --break-system-packages openpyxl` |
| `json5` | JSON5 解析 | `pip3 install --user --break-system-packages json5` |

### 文件处理类

| 库 | 用途 | 安装命令 |
|---|---|---|
| `Pillow` | 图片处理 | `pip3 install --user --break-system-packages Pillow` |
| `python-magic` | 文件类型识别 | `pip3 install --user --break-system-packages python-magic` |
| `chardet` | 编码检测 | `pip3 install --user --break-system-packages chardet` |

### 工具类

| 库 | 用途 | 安装命令 |
|---|---|---|
| `beautifulsoup4` | HTML 解析 | `pip3 install --user --break-system-packages beautifulsoup4` |
| `lxml` | XML/HTML 解析 | `pip3 install --user --break-system-packages lxml` |
| `pyyaml` | YAML 解析 | `pip3 install --user --break-system-packages pyyaml` |
| `python-dotenv` | .env 文件读取 | `pip3 install --user --break-system-packages python-dotenv` |

## 第五阶段：Node.js 包安装（补充）

OpenClaw 等基于 Node.js 的容器通常已有 `npm`，可直接使用：

```bash
# 全局安装（需要 --prefix 指定用户目录，避免权限问题）
npm install --prefix ~/.local <package-name>

# 或局部安装到工作目录
cd /workspaces && npm install <package-name>
```

常用 Node.js 库：

| 库 | 用途 | 安装命令 |
|---|---|---|
| `playwright` | 浏览器自动化 | `npm install playwright` |
| `archiver` | ZIP 打包 | `npm install archiver` |
| `axios` | HTTP 请求 | `npm install axios` |
| `cheerio` | HTML 解析 | `npm install cheerio` |

## 常见问题处理

### 问题 1：pip install 报 "Permission denied"

**原因：** 尝试写入系统目录。

**解决：** 确保使用 `--user` 标志：

```bash
pip3 install --user --break-system-packages <package-name>
```

### 问题 2：安装后 import 失败

**原因：** `~/.local/bin` 或 `~/.local/lib` 未在 PATH/PYTHONPATH 中。

**解决：**

```bash
# 添加到 PATH
export PATH="$HOME/.local/bin:$PATH"

# 添加到 PYTHONPATH（如果 import 仍失败）
export PYTHONPATH="$HOME/.local/lib/python$(python3 -c 'import sys; print(f"{sys.version_info.major}.{sys.version_info.minor}")')/site-packages:$PYTHONPATH"
```

建议将以上配置写入 `~/.bashrc`：

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
```

### 问题 3：编译失败（如 lxml、Pillow）

**原因：** 容器内缺少编译工具（gcc、make）和开发头文件。

**解决：**

1. 优先安装预编译的 wheel 版本：

```bash
pip3 install --user --break-system-packages --only-binary :all: <package-name>
```

2. 如果无预编译版本，标记为"环境缺少编译工具，需要管理员在镜像层面预装"。

### 问题 4：网络超时或无法访问 PyPI

**原因：** 容器网络受限。

**解决：**

1. 检查网络连通性：

```bash
curl -sI https://pypi.org/ | head -1
```

2. 如果无法访问 PyPI，需要管理员配置内网镜像源：

```bash
pip3 install --user --break-system-packages -i https://pypi.example.com/simple <package-name>
```

### 问题 5：get-pip.py 下载失败

**原因：** 无法访问 `bootstrap.pypa.io`。

**解决：**

1. 尝试使用国内镜像：

```bash
curl -sSL https://bootstrap.pypa.io/get-pip.py -o /tmp/get-pip.py || \
  curl -sSL https://mirrors.aliyun.com/pypi/get-pip.py -o /tmp/get-pip.py
```

2. 如果仍失败，检查 `ensurepip` 是否可用：

```bash
python3 -m ensurepip --user
```

## 安装检查清单

每次安装完成后，按以下清单验证：

1. [ ] pip 是否可用：`pip3 --version`
2. [ ] 包是否安装成功：`pip3 list 2>/dev/null | grep <package-name>`
3. [ ] 包是否能导入：`python3 -c "import <package_name>"`
4. [ ] PATH 是否正确：`echo $PATH | grep local/bin`
5. [ ] 记录安装的包名和版本，便于后续复现

## 安装日志规范

每次安装操作应记录以下信息：

```
[安装日志]
- 时间：<timestamp>
- 环境：<python版本> <容器信息>
- 操作：安装 <package-name>
- 命令：<实际执行的命令>
- 结果：成功/失败
- 版本：<安装的版本号>
- 备注：<如有异常或特殊处理>
```
