---
title: 现代单 Agent 架构详解：Router、ReAct、Plan-and-Execute 与 Reflection
published: 2026-09-04
description: 面向具备大模型 API 调用经验的开发者，从控制流、状态、终止条件和真实执行过程出发，讲清 Router、ReAct、Plan-and-Execute 与 Reflection 四种主流单 Agent 架构。
image: "./images/single-agent-architecture-misa-cover.png"
tags: [AI Agent, Agent Architecture, Router, ReAct, Plan-and-Execute, Reflection, FastAPI, Sandbox, 系统设计]
category: 人工智能
draft: false
lang: zh-CN
slug: modern-single-agent-architecture-evolution
---

很多所谓的 Agent 教程，实际只做了两件事：从向量库查一段资料，再让模型决定是否调用某个 API。然后架构图里画一个大脑、几个工具和一堆箭头，就宣布 Agent 搭完了。

问题在于，RAG 只负责给模型补充资料，Tool Calling 只规定模型如何请求外部能力。它们都没有回答 Agent 系统最重要的问题：

- 当前任务进行到哪一步；
- 下一步由程序决定，还是由模型决定；
- 工具返回结果后，模型是否还会继续行动；
- 一个长任务如何拆分和排序；
- 结果出错后，系统如何发现并修正；
- 什么情况下必须停止。

这些问题合在一起，才构成 Agent 的控制流。

本文讨论四种常见的单 Agent 架构：Router、ReAct、Plan-and-Execute 和 Reflection。它们不是从初级到高级的四个等级，而是四种不同的任务控制方式。实际系统可以只使用其中一种，也可以将它们组合起来。

## 一、先搞清楚：什么是单 Agent 架构

假设我们已经能调用一个大模型：

```python
response = model.generate(prompt)
```

这还不是 Agent。因为模型只能生成一次文本，没有状态，也不能与环境交互。

在它外面加上五个部分，才得到一个最小的 Agent Runtime（Agent 运行时）：

```text
用户目标 Goal
    ↓
控制器 Controller  ←→  状态 State
    ↓
模型 Model
    ↓
工具 Tools  →  环境 Observation
    ↓
终止条件 Stop Condition
```

这五部分分别解决不同问题：

| 部分 | 作用 | 例子 |
| --- | --- | --- |
| Goal | 定义最终要完成什么 | “生成上个月的销售报表” |
| Controller | 决定下一步怎么走 | 先读文件，还是直接写代码 |
| State | 保存任务进度 | 已找到 12 个文件，尚未生成报表 |
| Tools | 对外部环境执行动作 | 读文件、查数据库、运行 Python |
| Stop Condition | 防止无限运行 | 最多 8 步、90 秒或测试通过即结束 |

四种架构的差异，主要就在 Controller：

| 架构 | Controller 如何决定下一步 |
| --- | --- |
| Router | 只决定一次：把请求交给哪个固定流程 |
| ReAct | 每执行一步，就根据最新结果决定下一步 |
| Plan-and-Execute | 先生成完整计划，再按计划逐项执行 |
| Reflection | 执行后由验证器检查，失败则分析原因并重试 |

先记住这张表。后面所有代码和场景，都是这四句话的展开。

## 二、Router：只负责把请求送到正确的处理流程

### 2.1 Router 到底做了什么

Router 的工作很像医院分诊台。分诊人员不负责做手术，只负责判断病人应该去内科、外科还是急诊。

在 Agent 系统里，Router 接收用户请求，识别意图，然后把请求交给预先写好的 Handler（处理器）：

```text
用户请求
   ↓
识别意图
   ├─ 查询订单 → OrderHandler
   ├─ 申请退款 → RefundHandler
   ├─ 产品问答 → KnowledgeHandler
   └─ 无法判断 → ClarificationHandler
```

整个过程只做一次决策。进入 Handler 后，后面的步骤都已经由代码写死，模型不能临时改变流程。

```ts
type Intent = "order_query" | "refund" | "knowledge_qa";

type RouteDecision = {
	intent: Intent;
	confidence: number;
	orderId?: string;
};

async function handleMessage(message: string) {
	const decision = await classify(message);

	if (decision.confidence < 0.8) {
		return askForClarification(message);
	}

	return handlers[decision.intent](decision);
}
```

比如用户说：“订单 A1024 怎么还没发货？”分类器返回：

```json
{
	"intent": "order_query",
	"confidence": 0.96,
	"orderId": "A1024"
}
```

程序随后调用 `OrderHandler`。至于 Handler 是查数据库、调用物流 API，还是返回缓存，都与 Router 无关。

### 2.2 ToB 和 ToC 的 Router 为什么不同

两者都做意图分类，但优化目标不同。

#### ToB：先保证权限和确定性

企业系统中，“查询合同”和“作废合同”可能只差两个字，却对应完全不同的权限和风险。因此正确流程是：

```text
验证用户身份
   ↓
确认租户和角色
   ↓
识别业务意图
   ↓
校验参数
   ↓
代码检查权限
   ↓
调用业务 Handler
```

模型可以判断“用户似乎想作废合同”，但不能判断“用户有权作废合同”。权限必须由代码或策略引擎决定。

#### ToC：先控制成本和响应速度

消费场景请求量大，可以让便宜的组件先处理简单请求：

```text
第一道：规则
“/help”                         → 帮助页面
消息中含订单号                  → 订单分类器

第二道：小模型
“怎么还没到？”                  → 物流查询
“这个颜色还有货吗？”            → 商品咨询

第三道：通用模型
前两道都无法判断的开放表达       → 通用对话
```

这叫级联路由。假设规则处理 50% 请求，小模型处理 40%，只有 10% 进入通用大模型，系统的吞吐量和成本会比“所有请求都调用大模型”稳定很多。

### 2.3 高并发 Router 要解决什么

高并发场景的重点不是把分类 Prompt 写得更长，而是控制资源：

- 分类输出必须使用枚举和 JSON Schema，不能接受模型自创意图；
- 低置信度进入澄清流程，不能强行选择一个 Handler；
- 高频意图使用规则、缓存或小模型；
- 不同 Handler 使用独立限流，避免退款服务故障拖垮订单查询；
- 缓存键包含租户、用户权限摘要和 Router 版本，防止跨租户复用；
- 记录误分类样本，观察每类意图的 Precision 和 Recall。

### 2.4 什么时候用 Router

判断标准只有一个：**用户请求虽然多样，但后续处理流程可以提前列举。**

客服分流、模型选择、搜索策略选择、文档解析器选择都适合 Router。需要根据中间结果临时决定下一步的任务，不适合只用 Router。

## 三、ReAct：执行一步，观察结果，再决定下一步

### 3.1 ReAct 解决了 Router 的什么限制

考虑一个任务：“分析这些 CSV，找出退款率最高的地区，并生成脚本。”

程序事先不知道：

- 有多少个文件；
- 文件是什么编码；
- 每个文件的列名是否一致；
- 日期和金额是否合法；
- 第一次生成的脚本能否运行。

因此无法在任务开始前写死完整路径。Agent 必须先检查环境，再根据检查结果决定下一步。

[ReAct](https://arxiv.org/abs/2210.03629) 就是这种循环：

```text
判断下一步 Reason
       ↓
调用工具 Action
       ↓
获得结果 Observation
       ↓
根据新结果重新判断
       ↓
完成或继续循环
```

这里的重点不是让模型输出很长的思维链，而是让 Observation 真正参与下一次决策。

### 3.2 一个完整的 ReAct 执行过程

仍以 CSV 任务为例。

#### 第一次循环：先看有哪些文件

```json
{
	"action_summary": "先确认输入文件范围",
	"tool": "list_files",
	"arguments": { "pattern": "input/*.csv" }
}
```

工具返回：

```json
{
	"files": ["orders_a.csv", "orders_b.csv", "customers.csv"]
}
```

#### 第二次循环：检查结构

模型根据文件列表判断，下一步应该读取少量样本，而不是立刻生成代码：

```json
{
	"action_summary": "检查编码、列名和数据类型",
	"tool": "inspect_csv",
	"arguments": {
		"files": ["orders_a.csv", "orders_b.csv", "customers.csv"],
		"sampleRows": 30
	}
}
```

工具返回：

```json
{
	"orders_a.csv": {
		"encoding": "utf-8",
		"columns": ["订单号", "客户编号", "订单金额", "退款金额"]
	},
	"orders_b.csv": {
		"encoding": "gb18030",
		"columns": ["id", "buyer_id", "revenue", "refund_amount"]
	}
}
```

#### 第三次循环：基于事实生成代码

Agent 现在知道两个文件编码不同、列名不同，因此生成别名映射和编码探测逻辑：

```python
ALIASES = {
	"order_id": ["order_id", "订单号", "id"],
	"customer_id": ["customer_id", "客户编号", "buyer_id"],
	"amount": ["amount", "订单金额", "revenue"],
	"refund": ["refund", "退款金额", "refund_amount"],
}

def normalize_columns(frame: pd.DataFrame) -> pd.DataFrame:
	reverse = {alias: name for name, aliases in ALIASES.items() for alias in aliases}
	return frame.rename(columns=lambda column: reverse.get(column.strip(), column.strip()))
```

#### 第四次循环：执行脚本

```json
{
	"action_summary": "在样本数据上运行脚本",
	"tool": "run_python",
	"arguments": { "file": "pipeline.py", "timeoutSeconds": 20 }
}
```

如果返回 `ValueError: could not convert string to float`，Agent 下一步就不是提交结果，而是定位金额列中的非法字符串，补充 `pd.to_numeric(..., errors="coerce")`，然后再次运行。

这就是 ReAct：**不是一次把所有步骤想完，而是让每次工具结果改变后续动作。**

### 3.3 Zero-shot 和 Few-shot 在这里做什么

Zero-shot 不是一句“请一步步思考”，而是告诉模型循环协议和边界：

```text
目标：生成并验证数据处理脚本。
每轮只能调用一个工具或提交最终结果。
未知的编码、列名和数据类型必须先检查，不能猜测。
每次调用工具前输出一句简短的动作说明。
最多调用 8 次工具，总耗时不得超过 90 秒。
```

Few-shot 则提供一两个“输入—动作”示例，教模型如何选工具：

```json
{
	"input": "合并结构未知的 CSV",
	"correct_action": "先调用 inspect_csv，而不是直接 write_file"
}
```

工具简单、命名清楚时用 Zero-shot。多个工具功能相近、模型经常选错时，再加入 Few-shot 示例。

### 3.4 ReAct Runtime 怎么写

```python
def run(goal: str, runtime: Runtime):
	state = State(goal=goal)

	for step in range(runtime.max_steps):
		decision = runtime.decide(
			goal=state.goal,
			recent_observations=state.recent(5),
			available_tools=runtime.tool_schemas,
		)

		if decision.type == "finish":
			return decision.output

		observation = runtime.execute(
			decision.tool,
			decision.arguments,
			timeout=20,
		)
		state.append(decision, observation)

	raise StepLimitExceeded(runtime.max_steps)
```

`max_steps` 不是可选优化，而是架构的一部分。没有步数、时间、Token 和工具权限限制的 ReAct，只是一个可能无限消耗资源的循环。

为了控制上下文窗口，不要每轮都传入全部历史。保留最近几次 Observation；更早的信息压缩成“已确认事实”；大文件和长日志只返回存储地址、摘要和样本。

### 3.5 什么时候用 ReAct

适合 ReAct 的任务满足两个条件：下一步依赖刚获得的外部信息，且任务能在有限轮次内完成。

数据探索、故障排查、网页操作、搜索研究适合 ReAct。需要同时维护几十个文件、多个依赖和全局结构的任务，单纯使用 ReAct 很容易走一步看一步，最后偏离总目标。

## 四、Plan-and-Execute：先画任务地图，再逐项完成

### 4.1 为什么长任务不能只靠 ReAct

假设用户要求：“帮我初始化一个 FastAPI 后端，包含用户注册、登录、订单创建、数据库迁移和测试。”

如果使用纯 ReAct，Agent 可能先写订单接口，写到一半才发现认证模型还不存在；又去补数据库，之后发现目录结构与前面的 Import 对不上。

问题不是模型不会写代码，而是它一直在做局部决策，没有维护全局依赖。

Plan-and-Execute 将任务分成两个阶段：

```text
Planner：先回答“总共要做什么，先后关系是什么”
                         ↓
Executor：每次只回答“当前这个任务具体怎么完成”
```

### 4.2 Planner 应该输出什么

Planner 的输出不能只是一串自然语言 Todo。系统需要知道任务 ID、依赖、产物和完成条件：

```yaml
tasks:
  - id: project_structure
    output: app 目录与启动入口
    depends_on: []

  - id: database
    output: 数据库连接和会话依赖
    depends_on: [project_structure]

  - id: auth
    output: 登录接口和身份验证依赖
    depends_on: [database]

  - id: create_order
    output: POST /orders
    depends_on: [database, auth]

  - id: tests
    output: API 集成测试
    depends_on: [auth, create_order]
```

这是一张 DAG（有向无环图）。箭头表示依赖关系：

```text
project_structure
       ↓
   database
    ↙     ↘
 auth    user_repository
    \       /
   create_order
        ↓
      tests
```

在执行前，程序应先校验：

- 每个依赖指向的任务是否存在；
- 依赖图是否出现循环；
- 两个端点是否使用了相同的 Method 和 Path；
- 每个任务是否定义了输出和验收条件。

### 4.3 Planner 如何设计全局 Routing

除了任务依赖，Planner 还要先确定 API 全貌：

```yaml
routes:
  - method: POST
    path: /users
    handler: app.api.users:create_user
    response_model: UserRead

  - method: POST
    path: /sessions
    handler: app.api.sessions:create_session
    response_model: TokenRead

  - method: POST
    path: /orders
    handler: app.api.orders:create_order
    response_model: OrderRead
    requires: authenticated_user
```

先确定这份路由契约，Executor 才不会在不同文件里创造互相冲突的路径、模型和命名。

### 4.4 Executor 如何执行计划

Executor 不按 YAML 从上到下盲目执行，而是领取所有依赖已经完成的任务：

```python
def get_ready_tasks(plan: Plan, completed: set[str]) -> list[Task]:
	return [
		task
		for task in plan.tasks
		if task.id not in completed
		and set(task.depends_on).issubset(completed)
	]
```

当轮到 `create_order`，Executor 只接收四类上下文：

1. `/orders` 的路由契约；
2. `OrderCreate` 和 `OrderRead` 数据模型；
3. 已完成的数据库与认证接口；
4. 当前任务的验收条件。

然后生成当前端点：

```python
@router.post("/orders", response_model=OrderRead, status_code=201)
async def create_order(
	payload: OrderCreate,
	service: Annotated[OrderService, Depends(get_order_service)],
	user: Annotated[User, Depends(get_current_user)],
) -> OrderRead:
	return await service.create(owner_id=user.id, payload=payload)
```

完成后先检查 Import、类型和当前端点测试，通过才将任务标为 `completed`，随后解锁依赖它的任务。

### 4.5 什么时候重新规划

不是每次报错都重新规划。

- 少写一个 Import：当前任务局部修复；
- 测试断言失败：当前任务局部修复；
- `/orders` 契约从同步改为异步任务：重规划受影响子图；
- 数据库从关系型改为文档型：重新评估数据库之后的依赖。

计划要带版本，并记录它基于哪个代码状态生成：

```json
{
	"plan_version": 3,
	"based_on_commit": "a84c9e1",
	"status": "active"
}
```

### 4.6 什么时候用 Plan-and-Execute

适合它的任务有明确终点，包含多个相互依赖的子任务，并且每个子任务可以单独验收。例如项目初始化、跨文件重构、长报告生成和复杂迁移。

如果任务只有两三步，或者目标随每次 Observation 快速变化，先生成完整计划反而浪费时间，ReAct 更直接。

## 五、Reflection：运行、发现错误、分析原因、再次修复

### 5.1 Reflection 不等于“让模型自我检查”

让模型回答完后再问一句“你确定吗”，通常不会增加新的事实。模型只是对自己的文本再生成一次。

真正的 Reflection 必须有外部验证结果：

```text
生成代码
   ↓
在隔离环境运行
   ↓
测试或编译器给出结果
   ├─ 成功 → 结束
   └─ 失败 → 分析错误 → 生成最小修复 → 再次运行
```

[Reflexion](https://arxiv.org/abs/2303.11366) 的关键思想，是把环境反馈总结成下一次尝试可以使用的经验，而不是重新训练模型。

### 5.2 为什么一定要 Sandbox

如果 Agent 生成的是代码，就不能直接在宿主机上运行。它可能误删文件、无限创建进程、访问网络或读取环境变量。

Sandbox（沙盒）是一个受限制的执行环境。至少要做到：

- 默认关闭网络；
- 只挂载本次任务目录；
- 测试数据只读；
- 限制 CPU、内存和进程数；
- 设置硬超时；
- 不传入生产密钥。

```python
result = sandbox.run(
	image="python-runtime@sha256:固定镜像摘要",
	command=["pytest", "-q"],
	workspace=temporary_workspace,
	network="none",
	cpu=1,
	memory_mb=512,
	timeout_seconds=30,
)
```

Sandbox 返回的不是一句“代码有问题”，而是一组确定数据：

```json
{
	"exit_code": 1,
	"timed_out": false,
	"failed_test": "test_refund_rate_with_null",
	"exception": "TypeError: unsupported operand type",
	"location": "pipeline.py:48"
}
```

### 5.3 反馈如何变成修复

首先由 Verifier（验证器）处理原始日志：删除密钥和绝对路径，截断重复内容，提取失败测试、异常类型和代码位置。

然后 Reflector 生成一个结构化诊断：

```json
{
	"root_cause": "退款列在聚合前仍包含字符串空值",
	"evidence": ["test_refund_rate_with_null", "pipeline.py:48"],
	"repair_scope": ["pipeline.py", "tests/test_pipeline.py"],
	"repair": "先转为数值，再填充空值",
	"success_condition": "失败测试和全量测试全部通过"
}
```

最后由 Generator 根据诊断生成最小补丁。Reflector 负责说明为什么失败、应该改哪里；Generator 负责写代码。把诊断和修改分开，更容易限制修改范围。

### 5.4 完整的反馈循环

```python
seen_errors: set[str] = set()

for attempt in range(MAX_RETRIES + 1):
	workspace = create_workspace_from(baseline)
	patch = generator.create_patch(task, reflection)
	workspace.apply(patch)

	result = sandbox.run_tests(workspace)
	verdict = verifier.evaluate(result)

	if verdict.passed:
		return patch

	if verdict.error_signature in seen_errors:
		raise RepeatedFailure(verdict.error_signature)

	seen_errors.add(verdict.error_signature)
	reflection = reflector.diagnose(verdict, patch)

raise RetryLimitExceeded(MAX_RETRIES)
```

这里有三个容易忽略的细节：

1. 每次都从同一个基线创建临时工作区，失败补丁不能污染下一轮；
2. 相同错误再次出现就停止，不能无限重试；
3. 先跑失败测试，再跑相关测试，最后跑全量测试，避免局部修好、全局退化。

测试目录应默认只读，否则 Agent 可能通过删除测试或放宽断言来制造“测试通过”。

### 5.5 什么时候用 Reflection

只有结果能被客观验证时，Reflection 才可靠。代码可以用测试、编译器和 Lint 验证；SQL 可以用只读数据库和结果约束验证；配置可以用 Schema 验证。

广告文案“是否足够有感染力”没有稳定的外部判定标准。这种任务可以让模型点评，但不应该把它包装成自动纠错闭环。

## 六、四种架构不是四选一

一个代码开发助手可以同时包含四种控制流：

```text
用户请求
   ↓
Router 判断任务类型
   ├─ “解释这段代码” → 固定问答流程
   ├─ “排查这个报错” → ReAct
   └─ “初始化后端项目” → Plan-and-Execute
                                ↓
                         Executor 编写端点
                                ↓
                         Reflection 运行测试并修复
```

这里仍然只有一个主要 Agent 决策主体：

- Router 决定使用哪种控制器；
- ReAct 处理步骤未知的短任务；
- Plan-and-Execute 维护长任务的全局依赖；
- Reflection 只包裹那些存在客观验证方式的执行节点。

架构组合的关键不是多画几个方框，而是明确控制权在什么时候切换，以及切换时传递什么状态。

## 七、如何选型

按下面的顺序判断：

### 问题一：一次分类后，后续路径是否已经确定

如果是，使用 Router 或普通 Workflow。不要为了“更像 Agent”引入循环。

### 问题二：下一步是否依赖刚获得的外部结果

如果是，而且预计几步内可以完成，使用 ReAct。

### 问题三：任务是否包含多个依赖明确的子任务

如果是，使用 Plan-and-Execute。先让 Planner 生成可校验的任务图，再执行节点。

### 问题四：执行结果是否能被客观验证

如果可以，就在对应执行节点外增加 Reflection。如果没有稳定验证器，不要自动重试。

最终可以得到这张表：

| 任务 | 选择 | 原因 |
| --- | --- | --- |
| 客服意图分发 | Router | 路径有限，一次分类即可确定 |
| 查看文件后决定如何清洗数据 | ReAct | 后续动作依赖文件实际结构 |
| 初始化包含多个模块的 FastAPI 项目 | Plan-and-Execute | 模块之间存在明确依赖 |
| 自动修复测试失败的代码 | Reflection | 测试可以提供客观反馈 |

## 八、实现时必须守住的边界

无论采用哪种控制流，生产系统都要满足以下条件：

### 状态不能只存在聊天记录里

至少单独保存 `goal`、`current_step`、`observations`、`artifacts`、`budget` 和 `status`。聊天历史是输入，不是状态数据库。

### 模型不能直接执行工具

模型只输出工具名和参数。Runtime 负责 Schema 校验、权限检查、超时、限流和真正的执行。

### 每个循环都必须有硬停止条件

使用代码限制最大步数、最大重试次数、总耗时和 Token。不要依赖 Prompt 中的“请及时停止”。

### 上下文窗口只放当前决策需要的信息

长日志、大文件和早期轨迹放到外部存储。模型只接收摘要、引用和当前步骤相关内容。

### 每次状态迁移都要可追踪

记录 `trace_id`、原状态、动作、Observation、新状态、耗时、Token、错误签名和终止原因。否则线上失败后只能看到最终一句回答，无法知道 Agent 在哪一步走偏。

## 九、结语

理解单 Agent 架构，可以归结为一句话：**看谁在控制下一步，以及系统如何利用执行结果。**

- Router 做一次选择，将请求送入固定流程；
- ReAct 在每次执行后读取 Observation，再决定下一步；
- Plan-and-Execute 先建立全局任务图，再执行局部节点；
- Reflection 用外部验证结果驱动诊断和修复。

先根据任务的控制流选架构，再选择模型、知识库和工具。只接入 RAG 和 Tool Calling，不会自动得到一个设计良好的 Agent；同样，把所有请求都塞进自主循环，也不会让系统更智能，只会让行为更难预测。

真正可靠的 Agent，一半是模型能力，另一半是状态、预算、权限、验证和终止条件。
