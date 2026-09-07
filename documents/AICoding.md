# Copilot
开发公司：Microsoft，基于OpenAI的大语言模型。
## 安装方法
在Github中开启Copilot  
本地VS Code插件搜索Github Copilot Chat并下载  
VS Code如果登陆了Github账号就会自动启动  
## 功能
代码实时补全。在编写代码时，就像“智能联想”一样自动预测并生成下一行或整段代码，也支持内嵌对话面板询问问题。  
## 费用
每月免费的2000次代码补全（Code Completion），以及额度有限的 Copilot Chat问答。  
可以在Github网站查询当前用了多少次。  
Copilot Pro：$15/月， unlimited chats and code completions。access to current premium models。

# Cursor
开发公司：Anysphere
## 安装方法
Cursor 是基于 VS Code Fork 开发的。  
从 Cursor 官网下载并安装后，它就是一个独立的软件桌面图标。直接打开它就能编写代码，不需要电脑里先有 VS Code。  
电脑上可以同时安装 VS Code 和 Cursor，它们互相独立、互不影响。
## 功能
- 代码智能补全（Cursor Tab）:在编写代码时，Cursor 会根据上下文预测你可能要写的下一段代码甚至多行代码，按 Tab 键即可一键采纳。  
- AI 聊天交互（Chat / Cmd + L）：按下 Cmd + L（Windows 上为 Ctrl + L）开启聊天侧边栏。你可以向 AI 询问代码逻辑、解释报错信息或要求其重构当前文件，支持引用特定的文件或函数（输入 @ 符号触发引用）。
- 部代码修改（Edit / Cmd + K）：在代码编辑器中选中任意代码片段，按下 Cmd + K（Windows 上为 Ctrl + K），直接输入修改指令（如 "改用异步逻辑重构" 或 "增加异常处理"），AI 会在代码原位生成 Diff 修改供你对比确认。
- Agent 模式（Composer / Cmd + I）：按下 Cmd + I（或 Ctrl + I）开启 Composer 模式。在 Composer 中，AI 不仅能修改单文件，还能理解整个项目工程架构，自主创建、编辑多个文件、执行终端命令，跨文件完成复杂的业务功能需求。
- 项目全局索引（Codebase Indexing）：Cursor 会在本地对代码库建立向量索引，让 AI 在回答问题或修改代码时能快速检索整个项目的上下文。
## 费用
- Hobby（免费版）:基础 AI 功能体验;有限的 API / 进阶模型使用额度;包含 Cursor Tab 补全能力
- Pro（专业版）:$20 / 月,包含 $20 API 推理额度（用于调用 Claude、GPT-4o 等前沿模型）;在 Auto 与 Cursor 自研 Tab 模型中包含充足/不限量使用;支持所有主流顶级模型自由切换

# Claude Code
开发公司：Anthropic 
## 安装方法
- 在 VS Code 的扩展市场（Extensions）中搜索 "Claude Code" (番茄图标)并点击安装。
- 终端 CLI 安装（Claude Code 核心形态）  
在系统终端(比如VSCode terminal)中按官方指引安装好独立的 claude 命令行工具。
- 桌面应用安装（Claude Code Desktop）
## 功能
- 代码理解与重构
- 自动修改文件与运行测试
- Git 与 CI/CD 集成
- 多平台与权限模式
## 费用
- Claude Code 免费：有限使用,额度很少  
- Claude Code Pro: $20/月，额度标准  
- Claude Code Max 5x: $100/月，额度标准的五倍  
- Claude Code Max 20x: $200/月，额度标准的二十倍  

目前 Pro 的机制是：  
每 5 小时一个 usage window  
Pro 在高峰期至少提供 免费版的 5 倍 session usage  
达到 5 小时额度后，需要等窗口重置  
另外还有 weekly limit  
Claude 网页版 + Claude Code 共用这个额度，不是 Claude Code 单独给你一份额度  

# Codex
开发公司：OpenAI
## 安装方法
- 桌面应用: Microsoft Store 一键安装（官方推荐）
- CLI: 从 openai/codex 仓库下载二进制文件
- IDE 扩展: VS Code、Cursor、Windsurf 等主流编辑器提供官方 IDE 插件
- Web: 在 chatgpt.com/codex 提交长任务，适合后台运行。
## 功能
- 代码理解与分析
- 自动修改代码
- 自动生成运行测试
## 费用
Codex 本身不单独收费，包含在 ChatGPT 套餐中
- Free: 使用量很低
- Go: $8/月，比 Free 高，轻度coding
- Plus: $20/月，使用量明显更高，日常开发
- Pro: $100/月，使用量非常高，重度开发

# Gemini CLI
开发公司：Google
## 安装方法
在本地命令行中安装 Gemini CLI（支持通过 npm 或直接下载包管理）
## 功能
- 自动工具调用（Tools & Actions）：支持文件读取/写入、系统目录搜索（如 grep）、执行 Shell 命令、网页搜索与 Fetch。
- 安全沙盒与确认机制（Sandboxing）：在执行写文件或删除命令等高危操作前，CLI 会向用户展示具体命令并要求确认。
- 生态兼容（MCP）：支持 Model Context Protocol，可扩展接轨各类外部数据库与第三方 API 服务。
- 云端集成：开箱即用集成在 Google Cloud Shell 中，也可在 macOS、Linux、Windows 本地命令行中安装。
## 费用
- 免费额度 (Google AI Studio)：个人开发者使用来自 AI Studio 的 API Key 时，享有高限额的免费调用额度，足以覆盖日常终端命令交互与轻度开发需求。
- 按量付费 (Pay-As-You-Go)：按 API Token 计费，超出免费额度或绑定 Google Cloud 账单后，按调用的具体模型（如 Gemini Flash / Pro）的输入/输出 Token 计费（例如 Gemini Flash 输入低至 $0.10 - $0.50 / 百万 Token）。
- Google Cloud Shell：免费使用，在 Google Cloud Shell 环境中提供默认免费配额（云环境每周提供 50 小时免费使用时长）。
- Gemini Code Assist订阅：如果你购买或获得了 Gemini Code Assist 许可证，Gemini CLI 与 IDE 插件将共享配额，无需重复支付 API 费用。

# Windsurf
最初开发者：Codeium  
现归属：OpenAI（约 30 亿美元收购）  
## 安装方法
从 windsurf.com 下载对应系统的安装包  
可从 VS Code 或 Cursor 导入配置  
注册 / 登录 Windsurf 账号并开始使用 Windsurf  
## 功能
Windsurf 的核心是 Cascade ——一个真正能“执行任务”的 AI Agent，而不是简单的补全工具。  
Cascade 能：  
- 自动浏览整个代码库（无需你指定文件）
- 执行终端命令（安装依赖、运行测试、修复错误）
- 多文件协作编辑（前端、后端、配置、测试一起改）
- 自动生成项目结构
- 自动规划多步骤任务（AI Flow）
Windsurf 会记住：  
- 项目结构
- 你的编码习惯
- 团队术语
- 历史解决方案
Supercomplete（智能补全）：
- 不仅补全下一行，而是预测你的“下一步意图”。
## 费用
Free：2000次 Supercomplete + 25 次 Cascade Flow/月  
Pro：$15/月，无限 Supercomplete + 500 次 Fast Premium  

# Continue
开发公司：Continue Dev，采用 Apache 2.0 开源协议  
2026年6月中旬，Continue 团队被 Cursor 收购，官方 GitHub 仓库设置为只读状态，但其开源代码库依然开源可用。社区依然在广泛使用它，或者将其无缝迁移至继承其路线的开源项目（如 Cline 或 Roo Code）。  
要实现高效且低成本（甚至免费）的 AI 编程环境，Continue + 免费/开源模型 API 是非常经典的架构方案。  
## 核心大模型：DeepSeek-R1
定位： 强推理/逻辑链模型（Reasoning Model）。  
特点： 在出厂时就经过大规模强化学习训练，回答编码问题前会先进行内部“思考（Chain of Thought）”。擅长解决复杂 Bug、算法设计、重构底座架构等需要深度逻辑推理的场景。  
在 Continue 中的角色： 适合放在 Chat（对话）模式下，当你遇到极其晦涩的代码报错或复杂的逻辑需求时调用。  
## 核心大模型：Llama-3
定位： 全能通用开源大模型。  
特点： 具有极高的响应速度、出色的指令遵循能力和极强的代码生成基础。  
在 Continue 中的角色： 适合作为日常的代码补全（Tab 键自动续写）或常规的代码编写助手。  
## 核心大模型：Qwen2.5
## 核心大模型：GLM 系列 (智谱 AI - Zhipu AI)
## 核心大模型：Kimi / Moonshot 系列开源蒸馏版/轻量版 (月之暗面)
## 大模型运行引擎Ollama
Ollama类似于大模型界的 Docker。在终端输入一行命令（例如 ollama run gemma:7b 或 ollama run phi4）即可自动下载并以 API 形式在本地运行，Continue 等插件可直接无缝对接。  
## 安装方法
### 开源 AI 编程插件Continue
可通过VSCode Extension安装。  
第一次点击左侧 Continue 图标，它会自动引导你进行初始化设置。  
在配置文件中接入你的云端 API Key（如 DeepSeek、OpenRouter）或本地 Ollama 后，即可直接开始使用。  
### 大模型
- Online 模式，通过云端厂商提供的 API Endpoint 运行，不需要下载模型到本地。  
在 Continue 的配置文件（config.yaml 或 config.json）中填入 API Key 即可。  
免费/带赠送额度的厂商： 比如 Google Gemini API（提供免费额度）、DeepSeek 官方 API（价格极其便宜）、Groq / Together.ai / SiliconFlow（硅基流动，经常提供免费额度或低延迟免费模型）。  
OpenRouter： 一个 API 聚合平台，里面包含很多标记为 Free 的模型（包括各种蒸馏版的 R1 或 Llama-3）。    
- Local 模式（本地运行），利用 Ollama 或 LM Studio 等工具，把模型权重下载到你自己的电脑硬盘上，纯离线运行。
本地启动 Ollama 服务后，Continue 自动连通 http://localhost:11434。  
优点： 100% 隐私安全、代码不经过任何第三方服务器、无网络延迟抖动。  
硬件要求取决于你运行的模型参数量大小（以 Q4 量化版为例）：
    - 小模型 (1.5B ~ 8B): 8 GB 内存 / 显存(RTX 3060 (6G+))
    - 中型模型 (14B ~ 32B): 16 GB - 24 GB 显存/内存 (3090 / 4080 (16G-24G))
    - 大型/满血模型 (70B+): 48 GB+ 显存/统一内存 (双卡 RTX 3090/4090 或 Mac Studio (64G-128G))
    - 满血旗舰 (671B 原始 R1): 350 GB+ 显存/内存 (8× A100/H100 节点),不建议本地单机运行，必须走 Online API



