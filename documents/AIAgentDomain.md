# Domain AI Agent
Domain AI Agent也称作Vertical AI Agent, 比如SWE Agent跟普通AI Agent的区别是，SWE Agent把“解决一个软件工程工作项”作为最小交付单位，强调可验证的代码变更和工程闭环。  

Domain AI Agent跟AI Coding, AI Agent的区别如下：  

| 维度 | AI Coding | AI Agent | Domain AI Agent（垂直领域 AI Agent） |
|---|---|---|---|
| 核心定义 | 以生成、补全、解释和修改代码为主要能力的 AI 工具或助手 | 能围绕目标进行规划、调用工具、执行动作并根据结果迭代的通用智能体 | 面向某个专业领域或业务职能，整合领域知识、专用工具和工作流的 AI Agent |
| 关注中心 | “如何写出这段代码” | “如何完成这个目标” | “如何在特定领域中可靠地完成目标” |
| 覆盖范围 | 代码片段、函数、文件级修改；部分产品可扩展到仓库级 | 可跨代码、文档、浏览器、数据库、企业系统和 API 执行多步骤任务 | 聚焦一个垂直领域，如软件工程、医疗、法律、金融、芯片设计或客户支持 |
| 是否必须写代码 | 通常是 | 不必须 | 取决于领域；SWE Agent 通常需要，医疗或法务 Agent 通常不需要 |
| 是否必须调用外部工具 | 不一定；可仅在 IDE 对话或代码补全中工作 | 通常需要；工具调用是其完成复杂任务的重要手段 | 通常需要；且往往接入领域专属工具、数据源、系统和验证机制 |
| 是否具备自主规划 | 通常较弱或由用户主导 | 通常具备；可将目标拆成多个步骤并动态调整 | 通常具备；规划会受到领域规则、流程和风险约束 |
| 是否维护状态/记忆 | 通常只保留当前代码上下文、会话或仓库上下文 | 可维护任务状态、历史观察、中间结果和长期记忆 | 除任务状态外，通常还需维护领域实体、审计记录、案例上下文或规范版本 |
| 典型输入 | 自然语言需求、选中的代码、报错信息、代码注释、测试失败 | 高层目标、任务说明、约束条件、外部事件或用户请求 | 领域任务、领域数据、规则、专业文档、业务系统记录和合规要求 |
| 典型输出 | 代码补全、函数实现、重构建议、测试代码、代码解释、diff | 报告、检索结果、表单更新、邮件、工单、代码修改或其他工具执行结果 | 经领域流程处理的决策建议、可审计结果、领域文档、系统更新或经验证的交付物 |
| 验证方式 | 编译、lint、单元测试、类型检查、人工 code review | 任务完成状态、工具返回值、用户确认、规则校验 | 领域特有验证：例如 CI/测试、法规校验、诊断规则、交易风控、仿真或专家复核 |
| 主要风险 | 生成错误代码、幻觉 API、引入安全漏洞或回归 | 错误规划、错误工具调用、权限滥用、错误的外部副作用 | 在通用 Agent 风险之外，还可能有领域合规、专业准确性、安全性、责任归属和审计风险 |
| 人类角色 | 主导开发；AI 多数作为副驾驶或加速器 | 设置目标、授予权限、审查关键步骤和处理异常 | 定义领域边界、审批高风险决策、审核证据链并承担最终责任 |
| 代表性能力 | Copilot 式补全、代码问答、生成函数、重构、生成测试 | 浏览网页、查询系统、规划任务、调用 API、执行多轮工作流 | SWE Agent、法务审阅 Agent、医疗文档 Agent、金融研究或风控 Agent |
| 与 SWE Agent 的关系 | SWE Agent 通常包含 AI Coding 能力，但远不止代码生成 | SWE Agent 是 AI Agent 的一个子类 | SWE Agent 是面向软件工程领域的 Domain AI Agent |
| 简单例子 | “为这个 Vulkan buffer allocator 补一个 RAII 封装，并生成单元测试。” | “调查三个 GPU 厂商的最新架构信息，整理成比较报告并发到知识库。” | “读取 GitHub Issue，定位 Vulkan TLAS 更新错误，修改 C++/GLSL，运行测试，并生成可 review 的 PR。” |
| 一句话概括 | 帮你写代码 | 替你完成多步骤任务 | 在一个专业领域中，按该领域的知识、工具和规则完成任务 |


在有AI Agent的情况下，用什么语言实现Domain AI Agent:

| 模块 | 推荐语言 | 原因 |
|---|---|---|
| Agent 编排、Prompt、RAG、模型调用、评测 | **Python** | AI SDK、工具生态、异步服务、数据处理和快速迭代最成熟；SWE-agent 与 OpenHands SDK 都以 Python 作为重要实现基础 |
| Web UI、CLI、IDE 插件、服务端 API | **TypeScript** | VS Code 扩展、Web 前端、Node.js 服务和 GitHub App 集成方便 |
| 高性能领域工具、现有引擎/编译器/驱动集成 | **C++** | 适合直接复用 Vulkan engine、图形调试工具、仿真器、编译工具链与性能敏感模块 |
| 安全沙箱、执行器、并发 worker、基础设施 | **Go** 或 **Rust** | 适合构建稳定的命令执行服务、容器调度、权限隔离和高并发后端 |
| Tool 协议 | **JSON Schema + HTTP/REST、MCP 或 gRPC** | 将 Agent runtime 与领域工具解耦；工具可独立测试、替换、审计和部署 |


描述AI Coding, AI Agent for Coding, SWE Agent的关系：

| 类别 | 一句话定义 | 是否自主调用工具 | 典型任务范围 | 典型交付物 | 是否必须验证 |
|---|---|---:|---|---|---:|
| AI Coding | 用 AI 辅助人类理解、生成、修改或解释代码 | 不一定 | 代码补全、函数实现、问答、重构建议、生成测试 | 代码片段、建议、局部 diff | 不一定；通常由开发者自行验证 |
| AI Agent for Coding | 能围绕编码目标规划并调用代码工具的 Agent | 通常需要 | 搜索代码、读写文件、运行命令、修复局部错误、生成测试 | 多文件 diff、命令结果、修复建议 | 应该有，但可能只做局部 build/test |
| SWE Agent | 面向完整软件工程任务闭环的 Domain AI Agent | 必须或基本必须 | Issue → 定位 → 修改 → build/test → 迭代 → PR/review | 可审查 patch、测试结果、commit 或 PR | 必须依赖外部工程证据验证 |

# 如何实现Domain AI Agent
项目实践：  
https://github.com/gpuwangge/AIAgentSandbox/tree/main   

## 连接模型的方式
Native 连接模型和通过 OpenAI 方式连接模型，它们都能实现“用户输入一句话 → 模型回复一句话”。  
真正差别主要体现在：代码通用性和能否使用某家模型的特色功能。  

### Native API
直接按每家模型厂商自己的“母语”去调用它。  
你分别学日语、法语、意大利语，直接按每家餐厅自己的菜单和规则点菜。  
想完整发挥某一家模型的能力：选 native API。  

通过 native API 连接模型的本质，就是你的程序按模型提供商定义的网络协议，构造 HTTP 请求（header + JSON body），发送给模型服务；然后接收并解析它定义格式的 JSON 响应或流式数据。  

以本地 Ollama 为例，它默认把 API 暴露在 http://localhost:11434/api，其中聊天接口是 POST /api/chat。Ollama 官方也说明其接口可用于运行和交互模型，并默认提供流式响应。  

```
┌───────────────────────┐
│ VS Code + Continue    │
│ 或你自己的 C++ 程序    │
└───────────┬───────────┘
            │
            │ 1. 构造 HTTP Request
            │    - URL
            │    - HTTP method: POST
            │    - headers
            │    - JSON request body
            ▼
┌───────────────────────┐
│ Ollama HTTP Server    │
│ localhost:11434       │
└───────────┬───────────┘
            │
            │ 2. 校验并解析 JSON
            │    model、messages、options、stream...
            │
            ▼
┌───────────────────────┐
│ Ollama Runtime        │
│ 模型加载、KV cache、   │
│ tokenization、采样     │
└───────────┬───────────┘
            │
            │ 3. CPU / GPU 做推理
            ▼
┌───────────────────────┐
│ 本地 LLM 模型          │
│ Qwen / Llama / ...    │
└───────────┬───────────┘
            │
            │ 4. 返回 JSON 或一串流式 JSON
            ▼
┌───────────────────────┐
│ Continue / 你的程序    │
│ 解析结果、累积文本、    │
│ 更新 VS Code UI        │
└───────────────────────┘
```

### OpenAI-compatible API
大家都假装讲同一种“OpenAI 口音”，你的聊天机器人用一套代码就能连不同家的模型。  
OpenAI-compatible API：这些餐厅都额外提供了一份英文菜单。你不一定能点到所有当地隐藏菜，但基本菜都能统一地点。  
想最快支持多个模型、方便切换：选 OpenAI-compatible。  

OpenAI-compatible 是事实上的行业标准（de facto standard），不是由 ISO、IETF、W3C 等正式标准组织制定的强制标准。  
它之所以像“标准”，是因为大量模型托管商、本地推理框架和 Agent 应用，都愿意兼容 OpenAI 的请求/响应格式。这样开发者能复用已有 SDK、代码和工具链，只改少量配置就更换模型或推理后端。  

以 VS Code + Continue + Ollama 为例：模型仍由本地 Ollama 在 CPU/GPU 上运行，只是 Continue 不再使用 Ollama 的 /api/chat 原生协议，而改用 OpenAI 风格的 /v1/chat/completions 协议。  

```
┌───────────────────────┐
│ VS Code + Continue    │
│ 或你自己的 C++ 程序    │
└───────────┬───────────┘
            │
            │ 1. 构造 OpenAI-compatible HTTP Request
            │    - URL: /v1/chat/completions
            │    - HTTP method: POST
            │    - headers: Content-Type / Authorization
            │    - OpenAI 风格 JSON request body
            ▼
┌──────────────────────────────────┐
│ Ollama OpenAI-compatible API 层  │
│ http://localhost:11434/v1        │
│                                  │
│ 接收 OpenAI 风格请求并做协议适配  │
└───────────┬──────────────────────┘
            │
            │ 2. 转换/映射为 Ollama runtime 所需的请求
            │    model、messages、sampling、stream...
            ▼
┌───────────────────────┐
│ Ollama Runtime        │
│ 模型加载、KV cache、   │
│ tokenization、采样     │
└───────────┬───────────┘
            │
            │ 3. CPU / GPU 做推理
            ▼
┌───────────────────────┐
│ 本地 LLM 模型          │
│ Qwen / Llama / ...    │
└───────────┬───────────┘
            │
            │ 4. Ollama 将结果包装为 OpenAI 风格 response
            │    - 非流式：完整 JSON
            │    - 流式：SSE chunks
            ▼
┌───────────────────────┐
│ Continue / 你的程序    │
│ 解析 OpenAI schema：   │
│ choices[].message      │
│ choices[].delta        │
│ 并更新 VS Code UI      │
└───────────────────────┘
```


### 对比两种API
Native API
```
VS Code
  │  提问或要求修改
  ▼
Continue
  │
  │  POST http://localhost:11434/api/chat
  │  使用 Ollama 原生请求格式
  ▼
Ollama 本地模型
  │
  │  调 qwen2.5-coder / deepseek-coder / llama 等
  ▼
Continue 在 VS Code 展示回答或 diff
```
OpenAI-compatible API
```
VS Code
  │
  ▼
Continue
  │  认为它连接的是“OpenAI 风格服务”
  │
  │  POST http://localhost:11434/v1/chat/completions
  │  使用 OpenAI Chat Completions JSON
  ▼
Ollama 的 OpenAI-compatible 层
  │  将兼容请求交给 Ollama runtime
  ▼
Ollama 本地模型
  │
  ▼
Continue 在 VS Code 展示回答或 diff
```

## LangChain
LangChain 是一个帮助你搭建 LLM 应用和 AI Agent 的开源开发框架。它把“调用模型、组织上下文、调用工具、循环执行、管理状态”等常见工作封装起来；但它不是实现 AI Agent 的必须组件。你完全可以直接用 Ollama/OpenAI/Anthropic 的 HTTP API，自己写一个几十到几百行的 Agent loop。  

对你正在理解的 VS Code + Continue + Ollama 这类本地模型链路来说，可以把 LangChain 看成“位于 Chatbot/Agent 与模型 API 中间的一套通用 runtime 工具箱”。LangChain 官方把其 Agent 定义概括为：Agent = Model + Harness；这里的 harness 就是模型循环周围的 prompt、tools 和控制逻辑。  

最简单的 Chatbot 只需做一件事：
```
用户输入
  ↓
发 HTTP JSON 给 Ollama / OpenAI
  ↓
收到模型回答
  ↓
显示回答
```
这不需要 LangChain。
```
┌───────────┐        HTTP + JSON        ┌────────────┐
│ Chat UI   │ ─────────────────────────▶│ LLM Server │
│ / 后端    │◀───────────────────────── │ Ollama等   │
└───────────┘         回答 JSON         └────────────┘
```
但你一旦希望它成为 Agent，事情开始增加：
- 需要查本地文件、数据库或网页。
- 需要调用 shell、编译器、Git、CI、内部 REST API。
- 需要由模型决定“这一步该调用哪个工具”。
- 工具执行失败后，希望模型看到错误并自行重试。
- 要做 RAG，把代码库或文档检索结果塞进上下文。
- 要管理多轮对话、token budget、状态、日志、追踪和人工确认。
- 复杂任务要拆成规划、执行和验证步骤。

LangChain 的目标就是减少这些“模型周边胶水代码”。它提供面向模型、消息、tools、retrieval、middleware 和 Agent 的统一抽象，并对接多个模型提供商。  

下面这张图能说明 LangChain 不等于模型，也不等于工具；它更像一层 Agent runtime /  
```
┌─────────────────────────────┐
│ 用户 / VS Code / Web Chat UI │
└──────────────┬──────────────┘
               │ 用户任务
               ▼
┌───────────────────────────────────────────┐
│ LangChain                                  │
│                                           │
│ - 组织 system prompt / 历史消息 / 上下文   │
│ - 把 tools 描述交给模型                    │
│ - 收到 tool call 后执行工具                │
│ - 将 tool result 回传模型                  │
│ - 重复循环，直到任务完成                    │
│ - 可加入 memory、RAG、guardrail、日志      │
└───────┬──────────────────────┬────────────┘
        │                      │
        │ 调模型               │ 调工具
        ▼                      ▼
┌─────────────────┐   ┌─────────────────────┐
│ Ollama / OpenAI │   │ 文件、Git、数据库、  │
│ Claude / Gemini │   │ 搜索、shell、CI/API  │
└─────────────────┘   └─────────────────────┘
```
模型通常只负责两种决策：
```
A. 直接回复用户
B. 说“请调用某个工具，并给出参数”
```
LangChain 或你自己写的 Agent loop 负责后续动作：
```
模型返回 tool call
  ↓
执行对应的真实函数/API
  ↓
把执行结果放回对话
  ↓
再次请求模型
  ↓
模型决定下一步
```
官方文档也将 Agent 描述为“模型在循环中调用工具，直至任务完成”；工具本质是有明确输入输出的可调用函数，模型依据上下文决定何时调用和提供什么参数。  

LangChain 是否必须?  
完全不必须。
AI Agent 的最小定义不是“用了 LangChain”，而是：
+ 可调用工具
+ 调用工具后的观察结果
+ 必要时反复决策的循环

你可以用任意语言、任意模型 SDK、直接 HTTP，甚至用 C++ 自己实现。

| 需求                                   | 是否建议 LangChain | 原因                       |
| ------------------------------------ | -------------- | ------------------------ |
| 简单 Chatbot                           | 否              | 直接请求模型 API 即可            |
| 单个模型、单个简单工具                          | 通常否            | 手写循环非常短，依赖少、可控性高         |
| VS Code 内部小型 coding helper           | 视情况而定          | 先手写 adapter/loop 往往更利于调试 |
| 快速验证 RAG + 多种模型 + 多工具                | 是              | 可少写大量连接与编排代码             |
| 要同时支持 Ollama、OpenAI、Anthropic、Gemini | 常常值得           | provider 抽象能减少适配工作       |
| 工作流有显式状态机、重试、审批、长期任务                 | 更应考虑 LangGraph | 这类问题需要更强的状态与编排能力         |
| 产品级高风险写操作                            | 不能只靠 LangChain | 必须额外做权限、审批、审计、幂等和策略层     |

LangChain 可以快速连接不同模型和工具；LangChain 自身也强调其用途是从 model、tools、prompt、middleware 组合出合适的 agent，而不是规定唯一 Agent 架构。  

## LangGraph
LangChain = 偏高层、方便快速搭 Agent 的抽象和组件库  
LangGraph = 偏底层、显式管理 Agent 状态、节点、边、恢复和人工介入的编排框架  

对于“模型调用工具形成循环”的一般 Agent，LangChain 足够；对于长期运行、可暂停恢复、有审批节点、失败分支明确的业务 Agent，LangGraph 更接近状态机/任务图的思路。  
LangGraph 官方定位就是面向可靠 Agent 的低层 orchestration runtime，强调状态、memory、human-in-the-loop 等能力。  


