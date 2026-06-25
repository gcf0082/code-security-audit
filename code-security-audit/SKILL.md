---
name: code-security-audit
description: >
  通用代码安全审计与漏洞挖掘 skill。当用户要求对代码进行安全审计、漏洞挖掘、风险分析、接口安全审查、入口函数/攻击面分析、sink 点反查时触发。
  也适用于用户提到 "审一下这段代码有没有风险"、"看看这个 handler"、"帮我挖漏洞"、"分析这个入口是否有问题"、
  "这个 sink 点能不能利用"、"自顶向下分析"、"自底向上分析" 等场景。
  不适用于单纯的代码阅读理解、性能优化或功能开发任务。
---

# code-security-audit

## 核心约束

主 skill 可以浏览阅读代码，**但目的仅限于将任务具体化**——定位完整文件路径、行号范围、函数签名、路由注册信息等。**不允许做安全判断、调用链追踪、数据流分析、漏洞判定。**

| 允许（仅为定位目标） | 禁止（留给 subagent） |
|---|---|
| grep/glob 搜索类名、函数名、路由注册 | 判断参数是否可控 |
| 读函数签名和 docstring | 追踪调用链路 |
| 确定文件路径和行号范围 | 分析是否存在注入/越权等漏洞 |
| 找出路由 → handler 的映射 | 判定认证/鉴权是否缺失 |
| 读目录结构了解代码布局 | 输出"风险""问题""漏洞"等结论 |

主 skill 职责链：**定位目标 → 选 agent → 加载指令 → 构造任务书 → 派发 subagent**。

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

在选 agent 前，先针对已解析的目标做代码浏览定位。通过 grep/glob/read 将模糊描述转化为精确位置：

**不同目标类型的预调查方式：**

| 用户说... | 预调查动作 |
|---|---|
| "看 LoginHandler" | `grep "class LoginHandler"` + `grep "def login"` + `grep "LoginHandler"` 路由注册 |
| "查 /api/user" | `grep "/api/user"` 定位路由注册和对应 handler |
| "分析 app/models.py 的 cursor.execute()" | `grep -n "cursor.execute(" app/models.py` 确认行号 |
| "审一下 /app/services 目录" | `glob "**/*.py"` 遍历文件 + 初步识别外部入口 |
| "第 127 行的 exec() 有没有问题" | `read -o 120 -n 20` 确认上下文 |

**预调查输出——任务书的 `## 预调查信息` 部分：**

```
## 预调查信息
- 目标文件: src/handler/auth.py
- 目标函数: LoginHandler.post (第 48 行)
- 路由注册: router.add('/api/login', LoginHandler) (src/router.py:15)
- 函数签名: async def post(self, request: Request) -> Response
- 相关文件: src/handler/auth.py, src/middleware/auth.py
```

### 4. 选择 agent

根据解析结果选择对应 agent。若无精准匹配，选最接近的通用 agent。

### 5. 加载 agent 指令

读取选中 agent 的完整内容。如果 agent 文件中有 subagent 引用，一并读取（但只读需要的，不批量加载）。

### 6. 构造任务书并派发 subagent

**必须使用 subagent（Task tool）派发，不得在同一上下文中执行分析。**

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

subagent 的 prompt 中必须包含：
- 完整的 agent 指令（粘贴 agent 文件内容到 prompt 中）
- 用户原始任务的上下文
- 预调查提供的精确代码定位信息
- 目标路径和范围
- 明确要求将结果保存到约定目录
- 提示 subagent：**预调查信息已提供精确位置，subagent 通常无需重复定位，但仍需自行做安全分析**

### 7. 派发后回复

**主 skill 不分析、不汇总、不判断。** 直接告知用户已派发 subagent 去分析，subagent 的产出会直接呈现给用户。如果用户有额外要求，可以选其他 agent 再次派发。
