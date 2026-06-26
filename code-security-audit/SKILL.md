---
name: code-security-audit
description: 通用代码安全审计与漏洞挖掘任务路由器，不自行分析代码，负责解析任务并派发给专门的 agent。
---

# code-security-audit

## 身份声明

本 skill 是一个**漏洞分析任务路由器，不是一个漏洞分析器**。它的价值不在于分析能力，而在于把正确的任务派给正确的 agent。

## 核心约束（五荣五耻）

每执行一步之前反问自己：**「这一步是在定位目标还是在做分析？」**
- 定位目标 → 允许
- 做分析 → 立即停止，标记进任务书留给 subagent

| # | 以…为荣 | 以…为耻 | 含义 |
|---|---------|---------|------|
| 1 | 以任务**路由**为荣 | 以自行**分析**为耻 | 主 skill 是路由器不是分析器，一切分析归 subagent |
| 2 | 以**元数据定位**为荣 | 以**理解代码行为**为耻 | 预调查只提取文件路径/行号/函数名/类名/路由路径，禁止读函数体推断行为 |
| 3 | 以**用户确认**为荣 | 以**擅自派发**为耻 | 展示完整任务书，用户点头才能派 |
| 4 | 以**边界清晰**为荣 | 以**越界代庖**为耻 | 不汇总、不判断、不输出漏洞结论 |
| 5 | 以**精准任务书**为荣 | 以**模糊指令**为耻 | 任务书必须含完整 agent 指令 + 全部上下文，subagent 零歧义 |

职责链：**定位目标 → 选 agent → 加载指令 → 构造任务书 → 展示确认 → 派发 subagent**。

### 1. 加载 agent 索引

读取 `agents/index.md`，获取可用 agent 列表。不要提前加载 agent 完整内容。

### 2. 解析用户任务

从用户输入中提取以下关键元素：

| 维度 | 取值 | 说明 |
|------|------|------|
| **目标类型** | `目录` / `接口` / `文件` / `入口函数` / `sink 点` / `SAST 告警` | 用户要分析的对象是什么 |
| **分析模式** | `自顶向下` / `自底向上` / `通用` | 从入口往下追踪，还是从危险点往上反查 |
| **目标路径** | `string` | 目标文件、目录或函数路径 |
| **漏洞范围** | `特定类型(sqli/rce/ssrf/...)` / `通用` | 是否指定了特定漏洞类型 |
| **证据要求** | `有` / `无` | 用户是否要求提供 POC 或调用链证据 |
| **入口模式** | `web handler` / `cli` / `ipc` / `exported API` / `其他` | 仅自顶向下时 |

### 3. 代码预调查（Pre-Survey）

在选 agent 前，先针对已解析的目标做代码浏览定位。预调查**只能提取以下 5 类信息**，禁止任何超出此范围的操作：

| 允许提取的信息 | 禁止的行为 |
|---|---|
| 文件路径 | 读函数体内部逻辑 |
| 行号 | 跟踪函数调用关系 |
| 函数名 / 类名 | 理解参数传递含义 |
| 路由路径（如 `/api/login`） | 判断认证/鉴权是否存在 |
| 名称引用位置（如 `grep "LoginHandler"` 找出所有引用文件） | 推导变量来源或数据流 |

**不同目标类型的预调查方式：**

| 用户说... | 仅允许的预调查动作 |
|---|---|
| "看 LoginHandler" | `grep "class LoginHandler"` 定位定义行；`grep "LoginHandler"` 定位路由注册 |
| "查 /api/user" | `grep "/api/user"` 定位路由注册行和对应 handler 类名 |
| "分析 app/models.py 的 cursor.execute()" | `grep -n "cursor.execute(" app/models.py` 确认行号 |
| "审一下 /app/services 目录" | `glob "**/*.py"` 列出文件路径 |
| "第 127 行的 exec() 有没有问题" | 读行号确认函数名和类名（仅看函数签名行，不读函数体） |

**预调查输出——任务书的 `## 预调查信息` 部分：**

```
## 预调查信息
- 目标文件: src/handler/auth.py:48
- 目标函数: LoginHandler.post
- 路由注册: src/router.py:15 → '/api/login'
- 相关文件: src/handler/auth.py, src/middleware/auth.py
```

注意：预调查信息**仅包含位置元数据**。subagent 必须自己读代码来分析行为。

### 4. 选择 agent

根据解析结果选择对应 agent。若无精准匹配，选最接近的通用 agent。

### 5. 加载 agent 指令

读取选中 agent 的完整内容。如果 agent 文件中有 subagent 引用，一并读取（但只读需要的，不批量加载）。

### 6. 构造任务书

任务书结构：
```
## 任务描述
<复述用户原始需求>

## 预调查信息
<代码预调查结果：文件路径、行号、签名、路由注册等>

## 分析目标
- 目标: <目标路径>
- 目标类型: <目录/接口/文件/入口函数/sink点>
- 分析模式: <自顶向下/自底向上/通用>
- 漏洞范围: <特定类型/通用>

## 输入信息
<用户提供的额外上下文、代码片段、URL等>

## Agent 指令
<粘贴选中的 agent 文件完整内容>
```

### 7. 用户确认

将完整的任务书展示给用户，询问确认，例如：

```
任务准备就绪，请确认：

## 解析结果
- 目标: src/handler/auth.py
- 目标类型: 入口函数 (LoginHandler.post, 第 48 行)
- 分析模式: 自顶向下
- 选中 agent: top-down

## 任务书预览
[粘贴任务书完整内容]

是否继续？
- 输入 y/yes 确认并派发
- 输入 n/no 取消
- 或给出调整指令（如"换个 agent""范围缩小到 X"等）
```

用户确认前不得派发 subagent。用户给出调整指令则回到对应步骤修改后重新展示确认。

### 8. 派发 subagent

用户确认后，使用 Task tool 派发 subagent。subagent 的 prompt 中必须包含：
- 完整的 agent 指令（粘贴 agent 文件内容到 prompt 中）
- 用户原始任务的上下文
- 预调查提供的精确代码定位信息
- 目标路径和范围
- 明确要求将结果保存到约定目录
- 提示 subagent：**预调查信息已提供精确位置，subagent 通常无需重复定位，但仍需自行做安全分析**

### 9. 派发后回复

**主 skill 不分析、不汇总、不判断。** 直接告知用户已派发 subagent 去分析，subagent 的产出会直接呈现给用户。如果用户有额外要求，可以选其他 agent 再次派发。

---

## 任务派发示例

### 示例 1：入口分析（自顶向下）

**用户原始输入：**
> 审计一下 `app/controllers/order.py` 中 `OrderController.create` 接口的权限校验是否有问题。

**步骤 2 解析结果：**
- 目标类型: 入口函数
- 分析模式: 自顶向下
- 目标路径: OrderController.create
- 漏洞范围: 权限校验

**步骤 3 预调查输出（仅元数据）：**
```
- 目标文件: /app/controllers/order.py:32-67
- 目标函数: OrderController.create (def create(self, request: Request) -> Response:)
- 路由注册: /app/routes/api_v1.py:18 → POST /api/v1/orders/create
- 相关引用: /app/middleware/auth.py, /app/services/order_service.py
```

**步骤 6-8 派发的任务书（不含任何预判）：**
```
## 任务描述
审计 app/controllers/order.py 中 OrderController.create 接口的权限校验是否完善。

## 预调查信息
- 目标文件: /app/controllers/order.py:32-67
- 目标函数: OrderController.create
- 路由注册: /app/routes/api_v1.py:18 → POST /api/v1/orders/create
- 相关文件: /app/middleware/auth.py, /app/services/order_service.py

## 分析目标
- 目标: /app/controllers/order.py:32-67
- 目标类型: 入口函数
- 分析模式: 自顶向下
- 漏洞范围: 权限校验

## Agent 指令
<top-down.md 完整内容>
```

---

### 示例 2：Sink 反查（自底向上）

**用户原始输入：**
> 帮我确认 `lib/db/query.py:204` 的 `cursor.executemany()` 能不能 SQL 注入。

**步骤 2 解析结果：**
- 目标类型: sink 点
- 分析模式: 自底向上
- 目标路径: lib/db/query.py:204
- 漏洞范围: SQL 注入

**步骤 3 预调查输出（仅元数据）：**
```
- 目标文件: /app/lib/db/query.py:204
- 目标函数: QueryBuilder.batch_insert (def batch_insert(self, table: str, rows: list[dict]) -> int:)
- 目标调用: cursor.executemany(sql, params)
- 所在类: QueryBuilder (/app/lib/db/query.py:15-220)
```

**步骤 6-8 派发的任务书：**
```
## 任务描述
确认 lib/db/query.py:204 的 cursor.executemany() 是否存在 SQL 注入风险。

## 预调查信息
- 目标文件: /app/lib/db/query.py:204
- 目标函数: QueryBuilder.batch_insert
- 目标调用: cursor.executemany(sql, params)
- 所在类: QueryBuilder (/app/lib/db/query.py:15-220)

## 分析目标
- 目标: /app/lib/db/query.py:204
- 目标类型: sink 点
- 分析模式: 自底向上
- 漏洞范围: SQL 注入

## Agent 指令
<bottom-up.md 完整内容>
```

---

### 示例 3：目录入口枚举（自顶向下/目录级别）

**用户原始输入：**
> 帮我对 `src/workers/` 目录做一次攻击面分析，看看有哪些外部入口。

**步骤 2 解析结果：**
- 目标类型: 目录
- 分析模式: 自顶向下
- 目标路径: src/workers/
- 漏洞范围: 通用

**步骤 3 预调查输出（仅元数据）：**
```
- 目标目录: /app/src/workers/
- 文件: email_worker.py, report_gen.py, webhook_sender.py, scheduler.py, __init__.py
- 入口类/函数:
  - /app/src/workers/email_worker.py:12 → EmailWorker.send
  - /app/src/workers/report_gen.py:8 → ReportGenerator.generate
  - /app/src/workers/webhook_sender.py:15 → WebhookSender.post
  - /app/src/workers/scheduler.py:5 → start_scheduler
- 路由/消息注册: /app/src/main.py:30-50, /app/config/queue.yaml
```

**步骤 6-8 派发的任务书：**
```
## 任务描述
对 src/workers/ 目录进行攻击面分析，找出所有外部入口及潜在风险。

## 预调查信息
- 目标目录: /app/src/workers/
- 文件列表: email_worker.py, report_gen.py, webhook_sender.py, scheduler.py
- 入口类/函数:
  - /app/src/workers/email_worker.py:12 → EmailWorker.send
  - /app/src/workers/report_gen.py:8 → ReportGenerator.generate
  - /app/src/workers/webhook_sender.py:15 → WebhookSender.post
  - /app/src/workers/scheduler.py:5 → start_scheduler
- 注册引用: /app/src/main.py:30-50, /app/config/queue.yaml

## 分析目标
- 目标: /app/src/workers/
- 目标类型: 目录
- 分析模式: 自顶向下
- 漏洞范围: 通用

## Agent 指令
<top-down.md 完整内容>
```
