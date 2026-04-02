# 从零到一：用 `learn-claude-code` 实现简化版 Claude Code 代理（中文详解）

## 🌟 文章摘要

在人工智能领域，Claude Code 代表了一种新型的编程助手，它不仅仅是代码生成工具，更是一个智能代理系统，能够自主执行任务、调用工具、管理上下文。本文基于开源仓库 `learn-claude-code`，一步步指导你构建一个简化版的 Claude Code。通过这个过程，你将深入理解 LLM（大语言模型）代理的核心架构：**模型（Agent）** 负责决策，**工具系统（Harness）** 负责执行。

核心理念：
- **模型是智能大脑**：Claude/GPT 等 LLM 负责理解任务、制定计划、调用工具。
- **Harness 是执行环境**：提供工具、上下文管理、任务调度、子代理协作等机制。
- **价值**：从理论到实践，掌握 Agent 设计模式，适用于自动化编程、代码审查、项目管理等场景。

本文将通过仓库中的 12 个渐进式模块（s01 到 s12），展示如何从最小循环构建到完整系统。每个步骤都包含代码解读、运行示例和关键洞察。

---

## 🧱 一、前提与准备

### 1.1 仓库概述
`learn-claude-code` 是 Anthropic 官方的教学仓库，旨在通过 Python 代码演示 Claude Code 的内部机制。它不是完整的 Claude Code 实现，而是“学习版”，强调模块化设计。你可以从这里起步，逐步添加功能，最终构建自己的 Agent 系统。

仓库结构：
- [`agents`](agents )：12 个渐进式 Agent 实现，从简单到复杂。
- [`skills`](skills )：可插拔的知识模块（如代码审查、PDF 处理）。
- [`docs`](docs )：多语言文档，详细解释每个概念。
- [`web`](web )：Next.js 前端，用于可视化 Agent 执行流程。

### 1.2 环境搭建
1. **克隆仓库**：
   ```bash
   git clone https://github.com/shareAI-lab/learn-claude-code
   cd learn-claude-code
   ```

2. **安装依赖**：
   ```bash
   pip install -r requirements.txt
   ```
   依赖包括 `anthropic`（Claude API）、`openai`（可选）、`rich`（终端美化）等。

3. **配置 API Key**：
   - 复制环境文件：`cp [`.env.example`](.env.example ) .env`
   - 编辑 `.env`，添加 `ANTHROPIC_API_KEY=your_key_here`
   - 如果使用其他模型（如 GPT），添加相应 Key。

4. **验证环境**：
   ```bash
   python -c "import anthropic; print('Claude API ready')"
   ```

### 1.3 学习路径
- **顺序执行**：从 `s01_agent_loop.py` 开始，每个文件都是前一个的扩展。
- **终极目标**：运行 `s_full.py`，体验完整系统。
- **实践建议**：边读代码边运行，观察输出日志。每个模块都有注释，解释关键逻辑。

---

## 🔁 二、核心循环（最小可运行 Agent） — s01

### 2.1 背景
在构建 Agent 时，最基础的需求是让 LLM 能与外部世界交互，而非只是生成文本。传统的聊天机器人只能“说”，无法“做”。核心循环解决了这个问题，让模型能反复决策和执行，形成闭环。

### 2.2 当前问题
如果没有循环，Agent 只能单次响应，无法处理多步骤任务（如读取文件后修改）。此外，缺乏工具调用会导致模型只能“猜测”结果，无法验证或执行。

### 2.3 解决方式
引入无限循环：模型响应 → 检查是否需要工具 → 执行工具 → 将结果反馈给模型 → 继续。关键代码在循环条件和工具处理上。

文件位置：[`agents/s01_agent_loop.py`](agents/s01_agent_loop.py "agents/s01_agent_loop.py")

### 2.4 代码结构与关键代码
```python
messages = [{"role": "user", "content": "Hello, Claude!"}]  # 初始化对话

while True:  # 无限循环，确保持续交互
    response = client.messages.create(
        model="claude-3-5-sonnet-20241022",
        max_tokens=1024,
        messages=messages,
        tools=tools  # 工具定义
    )
    
    if response.stop_reason == "tool_use":  # 检查是否需要工具
        # 执行工具
        tool_name = response.content[0].name
        tool_args = response.content[0].input
        result = TOOL_HANDLERS[tool_name](**tool_args)  # 调用工具函数
        # 将结果塞回 messages，确保上下文连续
        messages.append({"role": "assistant", "content": response.content})
        messages.append({"role": "user", "content": f"Tool result: {result}"})
    else:
        # 最终答复
        print(response.content[0].text)
        break  # 退出循环
```

### 2.5 运行示例
```bash
python agents/s01_agent_loop.py
```
输入："帮我列出当前目录的文件"

输出：
- 模型调用 `list_dir` 工具
- 执行结果：显示文件列表
- 模型总结："当前目录有 README.md, agents/, docs/ 等"

### 2.6 关键洞察
- **决策 vs 执行分离**：模型只“说”要做什么，工具负责“做”。
- **上下文积累**：`messages` 记录历史，确保连续性。
- **为什么重要**：这是 Agent 的“心脏”，所有复杂功能都基于此循环。
- **扩展点**：添加更多工具、状态管理、错误处理。

---

## 🛠️ 三、工具系统扩展 — s02

### 3.1 背景
单一工具（如 s01 的 `list_dir`）限制了 Agent 的能力。现实任务需要文件读写、搜索、命令执行等多样操作。工具扩展让 Agent 成为“多功能助手”。

### 3.2 当前问题
s01 只有一个工具，Agent 无法处理复杂任务，如搜索代码或修改文件。工具调用硬编码，无法动态添加新功能。

### 3.3 解决方式
引入工具注册系统：定义 `tools` 列表和 `TOOL_HANDLERS` 字典。新工具只需添加 spec 和 handler，不改循环逻辑。

文件位置：[`agents/s02_tool_use.py`](agents/s02_tool_use.py "agents/s02_tool_use.py")

### 3.4 工具注册机制与关键代码
- **工具定义**：每个工具有一个 `spec`（名称、描述、参数）和 `handler`（执行函数）。
- **示例工具**：
  - `bash`：执行 shell 命令
  - `read_file`：读取文件内容
  - `grep_search`：文本搜索
- **注册方式**：
  ```python
  tools = [
      {
          "name": "bash",
          "description": "Run a bash command",
          "input_schema": {"type": "object", "properties": {"command": {"type": "string"}}}
      }
  ]
  TOOL_HANDLERS = {"bash": run_bash}  # 字典映射工具名到函数
  ```

### 3.5 运行示例
输入："读取 [`README.md`](README.md ) 的前10行"

- 模型调用 `read_file(path="README.md", start_line=1, end_line=10)`
- 执行：读取文件
- 输出：文件内容摘要

### 3.6 关键洞察
- **模块化**：新工具只需添加 spec 和 handler，不改循环。
- **安全考虑**：工具执行在沙箱中，避免恶意命令。
- **性能**：工具调用是同步的，后续模块会引入异步。
- **为什么重要**：工具丰富度决定 Agent 能力上限。

---

## 🧭 四、任务计划（Todo/任务列表） — s03

### 4.1 背景
复杂任务（如“构建网站”）容易让 Agent “思维漂移”，即偏离目标或遗漏步骤。任务计划引入“先计划再执行”的模式，提高可靠性。

### 4.2 当前问题
s02 的 Agent 在多步骤任务中可能遗忘或重复，导致效率低或错误。缺乏任务跟踪，无法验证进度。

### 4.3 解决方式
添加 Todo 系统：Agent 先生成任务列表，再逐个执行。使用状态管理（pending/done）跟踪进度。

文件位置：[`agents/s03_todo_write.py`](agents/s03_todo_write.py "agents/s03_todo_write.py")

### 4.4 实现机制与关键代码
- **Todo 模型**：任务列表，每个项有状态（pending/done）。
- **流程**：
  1. 用户输入任务
  2. Agent 生成 todo 列表
  3. 逐个执行，更新状态
- **代码片段**：
  ```python
  def generate_todo(task):
      # 调用模型生成步骤
      response = client.messages.create(...)
      return parse_todo(response.content)
  
  def execute_todo(todo_list):
      for item in todo_list:
          if item.status == "pending":  # 检查状态
              # 执行步骤
              result = run_tool(item.action)
              item.status = "done"  # 更新状态
  ```

### 4.5 运行示例
输入："帮我创建一个简单的 Python 脚本"

- 生成 Todo：1. 创建文件 2. 写代码 3. 测试运行
- 执行：逐步完成，输出最终脚本。

### 4.6 关键洞察
- **控制规模**：防止 Agent 跑偏。
- **可验证**：用户可调整计划。
- **扩展**：后续模块将 Todo 持久化到文件。

---

## 🧩 五、子代理（上下文隔离）— s04

### 5.1 背景
大任务需要分解，但主上下文会积累无关信息，导致 Token 浪费或混乱。子代理提供隔离，让每个子任务独立运行。

### 5.2 当前问题
s03 的 Todo 执行共享上下文，复杂任务会导致上下文污染，模型难以聚焦。缺乏递归能力，无法处理嵌套任务。

### 5.3 解决方式
引入 `spawn_subagent`：创建独立 `messages` 会话，执行子任务后合并结果。

文件位置：[`agents/s04_subagent.py`](agents/s04_subagent.py "agents/s04_subagent.py")

### 5.4 实现细节与关键代码
- **Spawn 子代理**：`spawn_subagent(task)` 创建新会话。
- **上下文隔离**：子代理的 `messages` 只包含相关历史。
- **结果合并**：子代理完成时，将结果传回主代理。
- **代码**：
  ```python
  def spawn_subagent(task):
      sub_messages = [{"role": "user", "content": task}]  # 独立上下文
      # 运行子循环
      result = agent_loop(sub_messages)
      return result  # 合并到主代理
  ```

### 5.5 运行示例
输入："先搜索代码，再修改文件"

- 主代理：调用子代理处理搜索
- 子代理：独立执行搜索，返回结果
- 主代理：基于结果修改文件

### 5.6 关键洞察
- **可组合性**：Agent 可以递归调用自己。
- **安全性**：隔离防止上下文泄露。
- **多角色**：后续用于团队协作。

---

## 📚 六、知识按需加载（知识注入）— s05

### 6.1 背景
Agent 需要外部知识（如 API 文档），但预加载所有知识会超 Token 限。按需加载确保高效使用上下文。

### 6.2 当前问题
s04 的上下文固定，无法动态注入新知识。Agent 缺乏领域专长，回答不准确。

### 6.3 解决方式
添加 `load_skill` 工具：需要时读取 [`skills`](skills ) 目录文件，注入到 `messages`。

文件位置：[`agents/s05_skill_loading.py`](agents/s05_skill_loading.py "agents/s05_skill_loading.py")

### 6.4 机制与关键代码
- **Skills 目录**：存放知识模块（如代码审查规则）。
- **加载工具**：`load_skill(name)` 读取文件，注入到 `messages`。
- **示例**：
  ```python
  def load_skill(skill_name):
      content = read_file(f"skills/{skill_name}/SKILL.md")
      messages.append({"role": "system", "content": content})  # 注入知识
  ```

### 6.5 运行示例
输入："如何做代码审查？"

- Agent 调用 `load_skill("code-review")`
- 注入知识：审查规则
- 输出：基于规则的建议

### 6.6 关键洞察
- **效率**：只加载相关知识。
- **动态**：知识可更新，无需重启。
- **扩展**：支持外部 API 或数据库。

---

## 🧠 七、上下文压缩（Context Compact）— s06

### 7.1 背景
长对话积累大量 Token，导致成本高或超限。压缩机制保留关键信息，丢弃细节。

### 7.2 当前问题
s05 的 `messages` 无限增长，模型“遗忘”早期信息，连续性差。

### 7.3 解决方式
实现三层压缩：近期完整、中期摘要、远期丢弃。

文件位置：[`agents/s06_context_compact.py`](agents/s06_context_compact.py "agents/s06_context_compact.py")

### 7.4 三层压缩与关键代码
- **近期**：保留完整
- **中期**：摘要
- **远期**：丢弃细节，只留关键点
- **代码**：
  ```python
  def compact_context(messages):
      recent = messages[-10:]  # 最近10条保留完整
      summary = summarize(messages[:-10])  # 摘要其余
      return summary + recent  # 合并
  ```

### 7.5 运行示例
长任务后，上下文自动压缩，保持性能。

### 7.6 关键洞察
- **连续性**：压缩后仍能理解历史。
- **技术**：依赖嵌入模型或简单摘要。
- **挑战**：平衡信息丢失与效率。

---

## 🗂️ 八、任务系统（持久任务板）— s07

### 8.1 背景
长期/跨会话的任务（例如 TODO、issue、子任务）不能仅靠对话历史保存。需要一个对话外的“任务板”，使任务状态在上下文压缩、会话重启后依然可用。

### 8.2 当前问题
早期模块将任务信息保存在内存或对话里，容易在上下文压缩或进程重启后丢失；任务间依赖也难以管理。

### 8.3 解决方式
`s07` 提供 `TaskManager`：把任务以 `task_{id}.json` 的形式存入 `.tasks/` 目录，支持 create/get/update/list、依赖关系和完成后自动解除阻塞。

文件位置：[`agents/s07_task_system.py`](agents/s07_task_system.py)

### 8.4 实现要点与关键代码
- 持久化：每个任务是一个 JSON 文件，包含 `id, subject, status, blockedBy, blocks, owner` 等字段。 
- 依赖处理：当任务被标记为 `completed` 时，会从其它任务的 `blockedBy` 中移除该 id（见 `_clear_dependency`）。

关键代码（节选自 `TaskManager`）：
```python
def create(self, subject: str, description: str = "") -> str:
  task = {
    "id": self._next_id, "subject": subject, "description": description,
    "status": "pending", "blockedBy": [], "blocks": [], "owner": "",
  }
  self._save(task)
  self._next_id += 1
  return json.dumps(task, indent=2)

def update(self, task_id: int, status: str = None,
       add_blocked_by: list = None, add_blocks: list = None) -> str:
  task = self._load(task_id)
  if status:
    if status not in ("pending", "in_progress", "completed"):
      raise ValueError(f"Invalid status: {status}")
    task["status"] = status
    if status == "completed":
      self._clear_dependency(task_id)
  # ... handle add_blocked_by / add_blocks ...

def _clear_dependency(self, completed_id: int):
  for f in self.dir.glob("task_*.json"):
    task = json.loads(f.read_text())
    if completed_id in task.get("blockedBy", []):
      task["blockedBy"].remove(completed_id)
      self._save(task)
```

在 `TOOLS` / `TOOL_HANDLERS` 中，s07 暴露了 `task_create`, `task_update`, `task_list`, `task_get` 等工具，供模型以工具调用的方式管理任务。

### 8.5 运行示例
用户或 Agent 可以调用 `task_create` 创建任务，后续用 `task_list` 查看，或在完成后用 `task_update` 标记 `completed`，从而触发依赖解除。

### 8.6 关键洞察
- 把“状态”移出对话历史是保证长期可用性的关键。
- 单文件 JSON 实现简单、可观察且易于调试，适合教学与原型。

---

## ⏳ 九、后台异步（Background Tasks）— s08

### 9.1 背景
有些操作（编译、长时间命令、远程请求）不应阻塞主 Agent 循环。需要“发起即走”的后台执行，并在完成时把结果回传给主循环。

### 9.2 当前问题
同步工具会让 Agent 在等待长任务时无法继续处理其他工作，导致响应性降低。

### 9.3 解决方式
`s08` 提供 `BackgroundManager`：在线程中运行命令、把完成通知放到通知队列，主循环在下一个 LLM 调用前把这些通知注入上下文。

文件位置：[`agents/s08_background_tasks.py`](agents/s08_background_tasks.py)

### 9.4 实现要点与关键代码
`BackgroundManager.run` 立即返回 task_id，实际工作在 `_execute` 中完成并把结果追加到 `_notification_queue`：
```python
def run(self, command: str) -> str:
  task_id = str(uuid.uuid4())[:8]
  self.tasks[task_id] = {"status": "running", "result": None, "command": command}
  thread = threading.Thread(target=self._execute, args=(task_id, command), daemon=True)
  thread.start()
  return f"Background task {task_id} started: {command[:80]}"

def _execute(self, task_id: str, command: str):
  try:
    r = subprocess.run(command, shell=True, cwd=WORKDIR, capture_output=True, text=True, timeout=300)
    output = (r.stdout + r.stderr).strip()[:50000]
    status = "completed"
  except Exception as e:
    output = f"Error: {e}"
    status = "error"
  self.tasks[task_id]["status"] = status
  self.tasks[task_id]["result"] = output or "(no output)"
  with self._lock:
    self._notification_queue.append({"task_id": task_id, "status": status, "command": command[:80], "result": (output or "(no output)")[:500]})

def drain_notifications(self) -> list:
  with self._lock:
    notifs = list(self._notification_queue)
    self._notification_queue.clear()
  return notifs
```

主循环在每次 LLM 调用前调用 `BG.drain_notifications()`，并把结果以一个小块文本注入 `messages`，让模型能看到完成的后台任务：
```python
notifs = BG.drain_notifications()
if notifs and messages:
  notif_text = "\n".join(f"[bg:{n['task_id']}] {n['status']}: {n['result']}" for n in notifs)
  messages.append({"role": "user", "content": f"<background-results>\n{notif_text}\n</background-results>"})
```

### 9.5 运行示例
发起 `background_run` 后立即返回，Agent 可继续处理其他工具；完成后通知被注入，下次 LLM 调用时模型可以基于结果做出后续决策。

### 9.6 关键洞察
- 后台执行提高并发利用率，但需要小心结果注入与竞态（race）问题。

---

## 🤝 十、团队协作与协议（Mailbox + 领队 / 协议）— s09/s10

### 10.1 背景
单个 Agent 能力有限，模拟团队（lead + 多名 teammate）能演示复杂协作场景：分工、审批、优雅关停等。

### 10.2 当前问题
没有统一的通信机制和协议，Agent 之间难以协作或达成一致（例如计划审批、关机请求）。

### 10.3 解决方式
`s09` 提供基于文件的 `MessageBus`（每人一个 JSONL inbox）和 `TeammateManager`（spawn 持久化线程）；`s10` 在此基础上增加了结构化协议（shutdown / plan_approval），并通过 request_id 做关联。

文件：[`agents/s09_agent_teams.py`](agents/s09_agent_teams.py) 和 [`agents/s10_team_protocols.py`](agents/s10_team_protocols.py)

### 10.4 实现要点与关键代码
- `MessageBus.send/read_inbox/broadcast` 使用 JSONL 文件追加／清空实现可靠的点对点通信：
```python
def send(self, sender: str, to: str, content: str, msg_type: str = "message", extra: dict = None) -> str:
  msg = {"type": msg_type, "from": sender, "content": content, "timestamp": time.time()}
  if extra: msg.update(extra)
  inbox_path = self.dir / f"{to}.jsonl"
  with open(inbox_path, "a") as f:
    f.write(json.dumps(msg) + "\n")
  return f"Sent {msg_type} to {to}"
```

- `TeammateManager.spawn` 启动一个守护线程运行 `_teammate_loop`，该循环会读取 inbox、将消息注入 `messages`，调用 LLM 并在遇到 `tool_use` 时执行本地工具（见 `_exec`）：
```python
thread = threading.Thread(target=self._teammate_loop, args=(name, role, prompt), daemon=True)
thread.start()
```

- s10 在 lead 端维护 `shutdown_requests` / `plan_requests` 字典，并提供 `handle_shutdown_request` / `handle_plan_review`：
```python
def handle_shutdown_request(teammate: str) -> str:
  req_id = str(uuid.uuid4())[:8]
  with _tracker_lock:
    shutdown_requests[req_id] = {"target": teammate, "status": "pending"}
  BUS.send("lead", teammate, "Please shut down gracefully.", "shutdown_request", {"request_id": req_id})
  return f"Shutdown request {req_id} sent to '{teammate}'"

def handle_plan_review(request_id: str, approve: bool, feedback: str = "") -> str:
  with _tracker_lock:
    req = plan_requests.get(request_id)
  if not req:
    return f"Error: Unknown plan request_id '{request_id}'"
  with _tracker_lock:
    req["status"] = "approved" if approve else "rejected"
  BUS.send("lead", req["from"], feedback, "plan_approval_response", {"request_id": request_id, "approve": approve, "feedback": feedback})
  return f"Plan {req['status']} for '{req['from']}'"
```

### 10.5 运行示例
Lead 可以用 `spawn_teammate` 启动成员；成员通过 `send_message`/`read_inbox` 与 lead 或其他成员通信；lead 可发出 `shutdown_request` 或回复 `plan_approval`。

### 10.6 关键洞察
- 文件式 inbox 简单可观测，适合示范与调试。
- 协议（request_id）把异步请求和回复关联起来，是分布式协调的基础。

---

## 🏁 十一、自治代理（自选任务）— s11

### 11.1 背景
把控制权交给 Agent：让它们主动扫描任务板并认领工作，可以模拟更少中央调度的场景（更贴近真实多智能体系统）。

### 11.2 当前问题
基于 lead 的分配在大规模时成为瓶颈；人工触发不够灵活。

### 11.3 解决方式
`s11` 在 teammate 内实现“工作/空闲”两阶段循环：工作阶段处理当前任务，返 `idle` 后轮询 inbox 和 `.tasks/`，发现未认领任务则 `claim_task` 并注入到对话里继续工作。

文件：[`agents/s11_autonomous_agents.py`](agents/s11_autonomous_agents.py)

### 11.4 实现要点与关键代码
- 扫描与认领：
```python
def scan_unclaimed_tasks() -> list:
  TASKS_DIR.mkdir(exist_ok=True)
  unclaimed = []
  for f in sorted(TASKS_DIR.glob("task_*.json")):
    task = json.loads(f.read_text())
    if (task.get("status") == "pending" and not task.get("owner") and not task.get("blockedBy")):
      unclaimed.append(task)
  return unclaimed

def claim_task(task_id: int, owner: str) -> str:
  with _claim_lock:
    path = TASKS_DIR / f"task_{task_id}.json"
    task = json.loads(path.read_text())
    task["owner"] = owner
    task["status"] = "in_progress"
    path.write_text(json.dumps(task, indent=2))
  return f"Claimed task #{task_id} for {owner}"
```

- 身份注入（在上下文被压缩后重新注入身份信息）：
```python
def make_identity_block(name: str, role: str, team_name: str) -> dict:
  return {"role": "user", "content": f"<identity>You are '{name}', role: {role}, team: {team_name}. Continue your work.</identity>"}
```

- 循环要点：工作阶段遇到 `idle` 请求则进入空闲轮询；空闲轮询会检查 inbox 与未认领任务，发现任务则认领并把任务提示注入 `messages`，恢复工作。

### 11.5 运行示例
启动一个自治 teammate，观察它如何在空闲时自动取得未认领任务并继续工作。

### 11.6 关键洞察
- 自治代理减少中央调度开销，但需要一致的任务板与锁（示例中用文件 + 线程锁）。

---

## 🧱 十二、工作区隔离（Worktree）— s12

### 12.1 背景
并发修改同一仓库会导致文件冲突、分支冲突与依赖问题。真实工程需要隔离执行环境。

### 12.2 当前问题
多 Agent 在同一工作目录并发执行会冲突，难以安全回滚或并行合并。

### 12.3 解决方式
`s12` 使用 Git worktree 为每个并发任务提供独立工作区，并维护 `.worktrees/index.json` 与事件日志 `.worktrees/events.jsonl` 来记录生命周期。

文件：[`agents/s12_worktree_task_isolation.py`](agents/s12_worktree_task_isolation.py)

### 12.4 实现要点与关键代码
- 检测仓库根目录：`detect_repo_root` 返回 git 仓库根（若存在），否则使用当前目录。
- `WorktreeManager.create` 通过 `git worktree add -b wt/{name} {path} {base_ref}` 创建工作树，并将条目写入索引，同时（可选）把 worktree 绑定到任务：
```python
def create(self, name: str, task_id: int = None, base_ref: str = "HEAD") -> str:
  branch = f"wt/{name}"
  self._run_git(["worktree", "add", "-b", branch, str(path), base_ref])
  entry = {"name": name, "path": str(path), "branch": branch, "task_id": task_id, "status": "active", "created_at": time.time()}
  idx = self._load_index()
  idx["worktrees"].append(entry)
  self._save_index(idx)
  if task_id is not None:
    self.tasks.bind_worktree(task_id, name)
  return json.dumps(entry, indent=2)
```

- `run` 在指定 worktree 路径内运行命令；`remove` 支持强制删除并可在关闭时把任务标为 `completed`，同时写事件到事件日志；`keep` 将 worktree 标记为保留。

### 12.5 运行示例
在一个 git 仓库中，Agent 可以：
- 创建 worktree：`worktree_create(name="auth-refactor", task_id=12)`
- 在 worktree 中运行测试：`worktree_run(name="auth-refactor", command="pytest -q")`
- 完成并移除：`worktree_remove(name="auth-refactor", complete_task=True)`

### 12.6 关键洞察
- 把执行环境隔离到独立目录可以显著降低并发冲突风险。
- 事件日志（`.worktrees/events.jsonl`）提供可审计的生命周期记录，方便自动化流程与回滚策略。

---

## 🧪 终极（整合版）— s_full

文件：[`agents/s_full.py`](agents/s_full.py)

整合所有功能，形成完整 Harness。运行它，体验从简单到复杂的演化。

---

## 💡 写博客结构建议（章节提纲）

1. 引言：Agent + Harness 理念
2. 准备：环境搭建
3. 核心：s01-s12 详细解读
4. 实践：运行示例和改造
5. 结论：未来展望

---

## ✨ 撰写提示（内容浓度）

- **图表**：用 Mermaid 画流程图。
- **代码**：关键片段 + 注释。
- **对比**：Claude Code vs 简化版。
- **实践**：鼓励读者运行并修改。

---

## 🏁 结语

通过 `learn-claude-code`，你不仅学到代码，更掌握 Agent 架构思维。从最小循环到完整系统，每一步都是进步。未来，结合训练数据，你可以构建更智能的 Agent。开始你的旅程吧！
