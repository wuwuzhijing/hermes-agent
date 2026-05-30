# wuwuzhijing 自定义分支工作流

本分支（`custom/workflow-docs`）记录 Hermes Agent 的自定义维护工作流。
这是对上游 [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) 的个人 fork。

---

## 仓库架构

```
origin  -> NousResearch/hermes-agent  (上游，只拉不推，push 已禁用)
my      -> wuwuzhijing/hermes-agent   (个人 fork，默认 push remote)
```

- `main` 分支始终跟踪 `origin/main`，保持与上游一致
- 自定义修改在 `custom/*` 分支上进行
- `main` 上从不直接改代码

---

## 自定义层次

Hermes 设计了三层自定义隔离，越往右越底层、冲突风险越大：

| 层次 | 内容 | 位置 | 生效方式 | 适合 |
|------|------|------|----------|------|
| 用户层 | config、skill、plugin、memory、persona | `~/.hermes/` | 即时或重启 | 行为调整、经验沉淀、工具扩展 |
| 分支层 | 改 Hermes 源码 | `hermes-agent` 的 `custom/*` 分支 | 重启 hermes | 核心逻辑修改、gateway 扩展 |
| 上游层 | 提 PR 回上游 | NousResearch/hermes-agent PR | 合入后 | 通用功能贡献 |

**原则：能用用户层解决的，不进分支层。**

---

## 四类改动场景

### 场景 1：调整 Agent 行为/人格/配置

涉及：persona、max_turns、timeout、模型切换、回退策略、终端参数等

**操作路径：**

```bash
# 编辑配置
vim ~/.hermes/config.yaml   # 主配置
vim ~/.hermes/.env            # 密钥

# 改 agent persona
hermes                          # 进入 CLI
/persona                         # 编辑当前会话人格
```

**常用调整项：**

- `agent.max_turns`: 每次对话最大 tool call 轮数（当前 40）
- `agent.gateway_timeout`: gateway 超时秒数（当前 1800）
- `agent.api_max_retries`: API 调用重试次数（当前 1）
- `terminal.timeout`: 终端命令超时秒数（当前 180）
- `terminal.lifetime_seconds`: 持久 shell 生命周期（当前 300）
- `model`: 默认模型/provider 切换
- `fallback_providers`: 主 provider 不可用时的回退链

### 场景 2：添加自定义工具/命令

涉及：新的 tool、slash command、lifecycle hook

**操作路径：**

```bash
# 创建插件目录
mkdir -p ~/.hermes/plugins/<plugin-name>

# 文件结构
~/.hermes/plugins/<plugin-name>/
├── plugin.yaml     # 插件声明
├── __init__.py     # register(ctx) 入口
├── schemas.py      # tool schema 定义（LLM 可见）
└── tools.py        # tool handler 实现
```

**plugin.yaml 示例：**

```yaml
name: my-tool
version: 1.0.0
description: 我的自定义工具
provides_tools:
  - my_tool_name
```

**__init__.py 注册入口：**

```python
from . import schemas, tools

def register(ctx):
    ctx.register_tool(
        name="my_tool_name",
        toolset="my-tool",
        schema=schemas.MY_TOOL,
        handler=tools.my_handler
    )
```

**tool handler 签名称：**

```python
def my_handler(args: dict, **kwargs) -> str:
    # 总是返回 JSON 字符串，永远不抛异常
    return json.dumps({"result": "..."})
```

**查看已加载工具：**

```bash
hermes tools
```

**调试插件加载：**

```bash
HERMES_PLUGINS_DEBUG=1 hermes
```

### 场景 3：扩展 Gateway / 平台适配

涉及：新的 messaging 平台（钉钉、飞书等），或修改现有平台行为

**操作路径：**

- 新平台适配：在 `custom/*` 分支下改源码，位置 `gateway/platforms/`
- 已有平台行为调整：同上述 plugin 方式，或改 `gateway/run.py`
- 注意：gateway 进程独立运行，改完需重启 `hermes gateway restart`

### 场景 4：修改 Hermes 核心逻辑

涉及：cli.py、run_agent.py、tools/、agent/

**操作路径：**

```bash
cd ~/.hermes/hermes-agent

# 从 main 开自定义分支
git checkout main
git pull origin main
git checkout -b custom/<feature-name>

# 修改代码...

# 验证（内部测试）
python -m py_compile <changed_files>
bash -n <changed_scripts>

# 提交并推送
git add <files>
git commit -m "custom: <描述>"
git push my custom/<feature-name>
```

**注意：**

- `main` 分支不直接改代码，始终保持与上游同步
- 每个自定义功能独立一个分支，便于 rebase 和 cherry-pick
- push 前需用户确认（用户偏好）
- commit message 使用标准中文

---

## 日常同步上游

```bash
cd ~/.hermes/hermes-agent

# 更新 main
git checkout main
git pull origin main
git push my main

# 将自定义分支 rebase 到最新 main
git checkout custom/<feature-name>
git rebase main

# 解决冲突后
git push my custom/<feature-name> --force-with-lease
```

---

## 经验沉淀

- 使用经验（大模型接入、provider 配置等）→ `~/.hermes/skills/devops/hermes-usage-experience/SKILL.md`
- 工程经验（DMS/RKNN/标注等）→ `~/.hermes/memories/MEMORY.md`
- 可复用工作流 → 新建 skill 到 `~/.hermes/skills/<category>/<name>/`

---

## 环境信息

- 安装路径：`~/.hermes/hermes-agent/`
- 用户配置：`~/.hermes/config.yaml`
- 密钥文件：`~/.hermes/.env`
- 日志目录：`~/.hermes/logs/`
- 当前主模型：DeepSeek `deepseek-v4-pro`（`api.deepseek.com/v1`）
- SSH Key：`wuwuzhijing` 已关联 GitHub
- 终端后端：local
- 已启用 gateway 平台：CLI、Weixin

---

## 常用命令速查

```bash
hermes                    # 启动 CLI
hermes --tui              # 启动 TUI
hermes tools              # 查看已加载工具
hermes plugins            # 查看已加载插件
hermes model              # 切换模型
hermes config             # 交互式配置
hermes logs               # 查看日志
hermes logs --follow      # 实时日志
hermes gateway start      # 启动 gateway
hermes gateway restart    # 重启 gateway
hermes skills             # 查看 skills
hermes skills update      # 更新 skills（curator）
hermes cron               # 查看定时任务
hermes session            # 管理会话
```
