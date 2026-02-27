# auto-shell

> **终端即为聊天框** — 在 Zsh 里输入自然语言，按 Ctrl+Alt+P 直接得到可执行命令。

AI 驱动的 Shell 命令生成工具。架构：本地 Python Daemon（FastAPI）+ 原生 Zsh 插件，无 PTY 劫持，无感融入现有工作流。

---

## 功能亮点

- **Ctrl+Alt+P 触发**：在命令行输入中文（或英文）描述，按 Ctrl+Alt+P 将其替换为可执行命令
- **智能路由**：简单任务直接给命令；复杂多步任务自动切换为 Agent 模式
- **Agent 会话模式**：持续建议 → 用户确认 → 执行结果自动反馈 → AI 给出下一步（结对编程体验）
- **三种安全等级**：Default（每步确认）/ Auto（AI 判断危险性）/ Full-Auto（全自动）
- **OpenAI 兼容 API**：支持 ChatGPT、GLM、DeepSeek、Llama 等任何兼容 OpenAI 格式的模型
- **轻量无侵入**：仅 hook `Ctrl+Alt+P` / `Ctrl+A` / `Ctrl+X,A`，不修改 `PS1`，不包装 PTY
- **流式输出控制**：可选择实时显示或等待完成后显示，避免终端闪烁
- **智能检测模式**：支持正则检测或 AI 判断输出类型，解释性文字直接显示而非放入命令行

---

## 架构

```
用户在 Zsh 输入自然语言
        │
        │ Ctrl+Alt+P
        ▼
┌──────────────────┐       HTTP (127.0.0.1:28001)       ┌─────────────────────┐
│  Zsh 插件        │ ─────────────────────────────────► │  Python Daemon       │
│  auto-shell      │                                    │  (FastAPI + uvicorn) │
│  .plugin.zsh     │ ◄───────────────────────────────── │                      │
└──────────────────┘       command / session step       └─────────┬───────────┘
                                                                   │ OpenAI API
                                                                   ▼
                                                        ┌─────────────────────┐
                                                        │  LLM (GLM / GPT 等)  │
                                                        │  Function Calling    │
                                                        └─────────────────────┘
```

---

## 环境要求

| 组件 | 版本要求 |
|------|---------|
| Python | ≥ 3.11 |
| Zsh | ≥ 5.8 |
| `jq` | 任意版本（用于 JSON 解析，推荐安装） |
| `curl` | 任意版本 |

---

## 安装

### 方式一：从 GitHub Release 安装（推荐）

1. 前往 [Releases](https://github.com/cacaview/auto-shell/releases) 页面
2. 下载最新版本的 `auto_shell-*.whl` 或 `auto-shell-*.tar.gz`
3. 安装：
```bash
pip install auto_shell-1.0.0-py3-none-any.whl
# 或
pip install auto-shell-1.0.0.tar.gz
```

### 方式二：从源码安装

```bash
git clone https://github.com/cacaview/auto-shell.git
cd auto-shell
pip install -e .
```

---

## 快速开始

### 1. 初始化配置

安装后首次运行会自动创建配置文件：

```bash
# 查看插件安装路径
pip show auto-shell | grep Location

# 初始化配置文件到 ~/.config/auto-shell/
auto-shell init
```

或者手动创建配置文件：

```bash
mkdir -p ~/.config/auto-shell
cat > ~/.config/auto-shell/config.yaml << 'EOF'
llm:
  api_base: "https://api.deepseek.com"
  api_key: "sk-your-api-key"
  model: "deepseek-chat"
  temperature: 0.1
  max_tokens: 200

daemon:
  host: "127.0.0.1"
  port: 28001

shell:
  stream_output: false
  smart_detect_mode: "regex"
EOF
```

### 2. 加载 Zsh 插件

找到插件路径并添加到 `~/.zshrc`：

```bash
# 查看插件路径
AUTO_SHELL_PATH=$(pip show auto-shell | grep Location | cut -d' ' -f2)
echo "插件路径: $AUTO_SHELL_PATH/auto_shell"

# 添加到 ~/.zshrc
echo 'source '"$AUTO_SHELL_PATH"'/auto_shell/plugin/auto-shell.plugin.zsh' >> ~/.zshrc
source ~/.zshrc
```

### 3. 启动 Daemon

```zsh
# 启动后台服务
auto-shell-start

# 检查状态
auto-shell-status
```

### 4. 使用

在终端输入自然语言，按 **Ctrl+Alt+P** 获取命令：

```
~ % 查找大文件<Ctrl+Alt+P>
→ find . -type f -size +100M
```

---

## 配置文件位置

auto-shell 按以下顺序查找配置文件：

1. 环境变量 `AUTO_SHELL_CONFIG` 指定的路径
2. 当前目录的 `config.yaml`
3. `~/.config/auto-shell/config.yaml`
4. `~/.auto-shell/config.yaml`
5. `/etc/auto-shell/config.yaml`

### 配置示例

**方式一：直接 source（临时，适合测试）**
```zsh
source /path/to/auto-shell/plugin/auto-shell.plugin.zsh
```

**方式二：写入 `~/.zshrc`（永久）**
```zsh
# 在 ~/.zshrc 末尾追加：
unset AUTO_SHELL_DAEMON_URL    # 清除可能错误的历史值
source /path/to/auto-shell/plugin/auto-shell.plugin.zsh
```

**方式三：oh-my-zsh 自定义插件目录**
```zsh
ln -s /path/to/auto-shell/plugin \
      ~/.oh-my-zsh/custom/plugins/auto-shell
# 在 ~/.zshrc 的 plugins 列表中加入 auto-shell
```

---

## 配置

编辑 `config.yaml`：

```yaml
llm:
  api_base: "http://127.0.0.1:8000/v1"   # OpenAI 兼容 API 地址
  api_key:  "sk-your-api-key"
  model:    "gpt-4o-mini"                 # 或 glm-4、deepseek-chat 等
  temperature: 0.1
  max_tokens:  200

daemon:
  host: "127.0.0.1"
  port: 28001

agent:
  default_mode: "default"                 # default | auto | full_auto
  max_iterations: 10
  dangerous_commands:                     # 需要用户确认的命令前缀（正则）
    - "^rm"
    - "^sudo"
    - "^chmod"
    - "^dd"

shell:
  request_timeout: 30                     # 请求超时时间（秒）
  stream_output: false                    # 是否启用流式输出
  smart_detect_mode: "regex"              # 智能检测模式: regex | ai
```

### 配置说明

- **stream_output**: 控制输出方式
  - `true`: 实时流式显示 AI 生成的命令
  - `false`: 等待 AI 完成后一次性显示（推荐）

- **smart_detect_mode**: 检测 AI 输出是否为解释性文字
  - `regex`: 使用正则表达式检测中文标点符号
  - `ai`: 让 AI 自己判断并标记输出类型

---

## 使用方法

### 1. 启动 Daemon

```zsh
# 前台运行（开发调试）
python -m uvicorn auto_shell.server:app \
    --host 127.0.0.1 --port 28001

# 后台运行
nohup python -m uvicorn auto_shell.server:app \
    --host 127.0.0.1 --port 28001 --log-level warning \
    > /tmp/auto-shell.log 2>&1 &

# 或使用插件内置命令
auto-shell-start
```

### 2. 获取命令建议（普通模式）

在 Zsh 命令行输入自然语言描述，然后**按 Ctrl+Alt+P**：

```
(base) ~ % 查找当前目录下最大的20个文件<Ctrl+Alt+P>
→ find . -type f -printf '%s %p\n' | sort -nr | head -20
```

按 **Enter** 执行，按 **Esc** 取消。

### 3. Agent 多步任务模式

复杂任务（如"安装并配置 nginx 然后重启"）会自动触发 Agent 模式提示：

```
→ 🤖 任务较复杂，已切换到 Agent 模式 — 再次 Ctrl+Alt+P 启动
```

再次按 Ctrl+Alt+P 启动会话，按 Enter 执行每一步，直到任务完成。

---

## 快捷键

| 按键 | 动作 |
|------|------|
| **Ctrl + Alt + P** | 获取命令建议 / 推进 Agent 步骤 |
| **Ctrl + A** | 循环切换主模式：`suggest` ↔ `agent` |
| **Ctrl + X, Ctrl + A** | 循环切换 Agent 子模式：`default` → `auto` → `full_auto` |
| **Esc** | 取消/清空当前建议 |

主模式说明：
- `suggest`: 普通模式，输入描述后获取单条命令
- `agent`: Agent 模式，处理多步骤复杂任务

---

## Agent 运行模式

| 模式 | 说明 | 命令确认 |
|------|------|---------|
| `default` | 每步都放入缓冲区等待用户确认 | 全部 |
| `auto` | AI 自动判断安全性 | 仅高危命令 |
| `full_auto` | 无拦截，全部自动执行（⚠️ 慎用） | 无 |

切换方式：
```zsh
auto-shell-mode agent        # 主模式切换到 agent
auto-shell-mode auto         # Agent 子模式切换到 auto
auto-shell-mode full_auto    # Agent 子模式切换到 full_auto
auto-shell-mode status       # 查看当前状态
auto-shell-mode stop         # 停止当前 Agent 会话
```

---

## 管理命令

```zsh
auto-shell-start             # 后台启动 Daemon
auto-shell-stop              # 停止 Daemon
auto-shell-status            # 查看 Daemon 状态
auto-shell-test              # 运行插件自测
auto-shell-mode status       # 查看当前模式
```

---

## 项目结构

```
auto-shell/
├── auto_shell/
│   ├── config.py        # 配置管理
│   ├── context.py       # 终端上下文收集器
│   ├── llm_client.py    # LLM 客户端
│   ├── agent.py         # Agent 引擎
│   ├── server.py        # FastAPI Daemon
│   └── cli.py           # 命令行入口
├── plugin/
│   └── auto-shell.plugin.zsh   # Zsh 插件
├── tests/
│   └── test_auto_shell.py      # 测试套件
├── config.yaml.example   # 配置模板
└── pyproject.toml        # 项目依赖
```

---

## HTTP API 接口

Daemon 服务默认运行在 `http://127.0.0.1:28001`，提供以下接口：

### 健康检查

```
GET /health
```

响应：
```json
{"status": "ok", "timestamp": "2024-01-01T12:00:00"}
```

### 获取配置

```
GET /config
```

响应：
```json
{
  "llm_api_base": "https://api.openai.com",
  "llm_model": "gpt-4o-mini",
  "daemon_host": "127.0.0.1",
  "daemon_port": 28001,
  "agent_mode": "default",
  "stream_output": false,
  "smart_detect_mode": "regex"
}
```

### 命令建议（同步）

```
POST /v1/suggest
Content-Type: application/json

{
  "query": "查找大文件",
  "cwd": "/home/user",
  "os": "Linux",
  "shell": "zsh"
}
```

响应：
```json
{
  "command": "find . -type f -size +100M",
  "explanation": "查找当前目录下大于100MB的文件",
  "is_dangerous": false,
  "use_agent": false
}
```

### 命令建议（流式 SSE）

```
POST /v1/suggest/stream
Content-Type: application/json

{
  "query": "查找大文件",
  "cwd": "/home/user",
  "os": "Linux",
  "shell": "zsh"
}
```

响应（SSE 流）：
```
data: {"chunk": "find"}
data: {"chunk": " . -type"}
data: {"chunk": " f -size +100M"}
data: [DONE]
```

### 启动 Agent 会话

```
POST /v1/agent/session/start
Content-Type: application/json

{
  "task": "安装并配置 nginx",
  "cwd": "/home/user",
  "os": "Linux",
  "shell": "zsh",
  "mode": "default"
}
```

响应：
```json
{
  "session_id": "abc123",
  "iteration": 1,
  "action": "execute",
  "command": "sudo apt update",
  "is_dangerous": false,
  "needs_confirmation": true
}
```

### 推进 Agent 会话

```
POST /v1/agent/session/step
Content-Type: application/json

{
  "session_id": "abc123",
  "last_command": "sudo apt update",
  "last_exit_code": 0
}
```

### 删除 Agent 会话

```
DELETE /v1/agent/session/{session_id}
```

### 上报命令执行结果

```
POST /v1/command/result
Content-Type: application/json

{
  "command": "ls -la",
  "exit_code": 0,
  "stdout": "file1.txt\nfile2.txt",
  "stderr": ""
}
```

---

## 路线图

- [x] Stage 0：PoC — Daemon + 基础 API + 插件原型
- [x] Stage 1：MVP — 上下文收集 + 命令替换
- [x] Stage 2：Agent 模式 — ReAct 循环 + 会话管理
- [ ] Stage 3：体验优化

---

## License

MIT

---

## 贡献者

- cacaview - 项目创始人
- iFlow CLI - 功能开发与优化