# skill-capability-probe 使用方法

## 方式一：直接调用

告诉智能体：

```
请使用 skill-capability-probe 技能，对当前环境进行能力探测。
```

智能体会按顺序执行 8 项测试，最后输出结构化报告。

## 方式二：预设环境变量

先设置环境变量，再调用：

```bash
export PROBE_TEST_VAR="my-test-value"
```

然后让智能体执行探测。如果报告中显示 `PROBE_TEST_VAR=my-test-value`，说明环境变量传递链路正常。

如果未设置该变量，智能体会使用默认值 `probe-default`，测试仍能通过，但无法验证环境变量传递是否正常。

## 方式三：部署后自检

在新部署的智能体环境中，作为第一个运行的 Skill，确认环境是否就绪。

推荐流程：

1. 部署智能体环境
2. 运行 `skill-capability-probe`
3. 检查报告中的通过项
4. 根据失败项的补救建议修复环境
5. 重新运行直到 8/8 通过
6. 再部署其他 Skills（如 `container-pip-bootstrap`、`manual-generate-fudan`）

## 报告解读

### 全部通过（8/8）

环境完全支持 Skills，可使用所有技能。

### 部分通过

根据失败项判断影响：

| 失败项 | 影响哪些 Skills |
|---|---|
| SKILL.md 加载 | 所有 Skills（根本性问题） |
| 环境变量读取 | manual-generate-fudan（需要 BROWSERLESS_URL） |
| 文件写入 | manual-generate-fudan（需要输出手册和截图） |
| 文件读取 | 所有需要读取配置文件的 Skills |
| 命令执行 | container-pip-bootstrap（需要执行 pip 安装） |
| 多步骤执行 | manual-generate-fudan（多步骤探索流程） |
| 清理测试文件 | 影响较小，但可能导致文件残留 |
| 输出规范化 | 影响输出可读性 |

### 严重受限（0-3/8 通过）

建议先检查智能体平台配置，确认：
- SKILL.md 是否在正确的 skills 目录下
- 容器是否有可写的工作目录
- 容器是否允许执行 shell 命令
