# How to Build a Local AI Agent

## VSCode插件工具：Continue
Continue是开源 AI 编程插件，它不能独立使用，必须配合大模型。  
开发公司：Continue Dev，采用 Apache 2.0 开源协议  
2026年6月中旬，Continue 团队被 Cursor 收购，官方 GitHub 仓库设置为只读状态，但其开源代码库依然开源可用。社区依然在广泛使用它，或者将其无缝迁移至继承其路线的开源项目（如 Cline 或 Roo Code）。  
要实现高效且低成本（甚至免费）的 AI 编程环境，Continue + 免费/开源模型 API 是非常经典的架构方案。  

## Steps to Build a AI Chatbot and AI Coding Complete System
### 首先是安装Continue
可通过VSCode Extension安装。  
第一次点击左侧 Continue 图标(一个空心六边形)，它会自动引导你进行初始化设置。  
在配置文件中接入你的云端 API Key（如 DeepSeek、OpenRouter）或本地 Ollama 后，即可直接开始使用。  
有两种模式运行大模型：  
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
### Online模式：使用云端API Key
- 如何获得API Key：以Google Gemini API为例，首先准备好google账号，前往 Google AI Studio，登陆后同意服务条款，然后点击屏幕左侧Dashboard界面，会看到API Key已经生成好了，默认状态是Free tier。(同时会看到自动生成了一个Default Gemini Project)  
Free tier的Gemini API Key没有时间过期，但有配额限制：具体的每分钟请求数（RPM）和每天请求数（RPD）会根据你调用的模型（如 Flash 或 Pro 系列）以及官方当时的实时容量政策而变化。  
Free tier的数据可能会被 Google 记录以用于改进其产品和服务。  
- Gemini Project有什么用：Google 对 Gemini API 的免费使用限制（如每分钟请求数 RPM、每天请求数 RPD）是按项目（Project）统计的，而非单纯按 API 密钥计算。  
即使是免费调用，也需要在 Google Cloud 底层启用 Generative Language API 等核心服务，项目充当了这些服务的容器。  
将 API 密钥绑定在项目中，如果你以后决定从免费层升级到按量付费（Pay-as-you-go）方案，只需直接为该项目关联账单（Billing）即可无缝提升配额，无需重新更换密钥。  
- Gemini的模型分为Flash/Pro，若使用免费的Project，则具体限制如下：
    - Flash: 每分钟请求数 (RPM): 15 次/分钟, 每日请求数 (RPD): 1500 次/天, 每分钟 Token 数 (TPM): 1,000,000 (100万) tokens。
    - Pro: 每分钟请求数: 只有 2 次/分钟，每日请求数: 只有 50 次/天，每分钟 Token 数: 只有 32000 tokens。
- Gemini的模型付费层级 (Pay-as-you-go)
    - Flash: $0.075 / 每百万 tokens ~ $0.60 / 每百万 tokens
    - Pro: $3.50 / 每百万 tokens ~ $21.00 / 每百万 tokens
    - 数据不会被用于训练模型。
    - Flash 的 RPM 通常提升到 2,000 次，且没有每日上限。
    - Pro 的 RPM 通常提升到 360 次，且没有每日上限。
### Local模式：使用Ollama调用本地大模型运行引擎
Ollama类似于大模型界的 Docker。在终端输入一行命令（例如 ollama run gemma:7b 或 ollama run phi4）即可自动下载并以 API 形式在本地运行，Continue 等插件可直接无缝对接。  
首先在以下网站下载Ollama(图标是一只羊驼)  
```
https://ollama.com/download  
```
安装后，第一次运行会被要求创建账号(可选)。会被要求Connect账号和设备(比如笔记本电脑)(这时候会自动生成一个ssh号码)。  
设定完成后，会弹出Apps页面，告诉你Ollama支持哪些Terminal。  
接下来打开一个terminal比如Command Prompt,拉取需要的模型。  
以模型qwen2.5-coder:7b为例，运行(和拉取)命令如下：  
```
ollama run qwen2.5-coder:7b
```
第一次执行这个命令会在网上拉取(下载)qwen2.5-coder:7b模型(4.7GB)。以后执行就不需要再下载了。  
Terminal显示success后，就可以直接在Terminal里跟模型聊天了。  
为了保证VSCode也能使用，在浏览器中输入以下网址：  
```
http://localhost:11434
```
11434是OLLAMA_HOST指定的默认TCP端口。改成其他的未占用端口也是可以的。  
若显示“Ollama is running”，则表示本地服务已就绪，VSCode可以通过这个端口来连接模型。  
(此时可以关闭Terminal，端口11434的服务仍然会继续有效)  
(如果要关闭服务，使用ollma stop, 或者在任务管理器中强制关闭ollama.exe。如果要重新打开，就重新执行ollama run)  
回到VSCode，点击Continue的空心六边形图标，再点击右上角齿轮，再点击Configs图标，再点击"Main Config"右边的六边形齿轮图标，会打开config.yaml文件。  
在config.yaml文件内加入如下字段：  
```
models:
    - name: "Qwen2.5 Coder 7B"
        provider: ollama
        model: qwen2.5-coder:7b
        apiBase: http://localhost:11434
        roles:
            - chat
            - autocomplete
```
这时候在Continue底部就可以选择"Qwen2.5 Coder 7B"作为对话模型了。  
roles这一栏如果不写，默认就是chat。写了autocomplete并且模型也支持autocomplete功能，就可以在代码编辑器中使用Tab键进行代码补全。  
测试方法是在VSCode里随便打开一个文档，输入void quicksort，按tab，看看是否出现自动补全。  
一般来讲7b做补全没有问题，但如果想提升响应速度，可以用1.5b的qwen2.5模型。  
到这里我们就用Continue+Ollama+Qwen2.5 Coder 7B搭建了一个**免费的无限使用的不依赖网络的**AI聊天和代码补全环境。  

 
## Steps to Build a AI Agent
首先AI Agent对模型有更强的需求。这里选取Llama-3.1:8b作为基础模型。  
大模型介绍：Llama-3  
定位： 全能通用开源大模型。  
特点： 具有极高的响应速度、出色的指令遵循能力和极强的代码生成基础。  
其中llama3.1:8b是入门级AI Agent模型。(Agent Mode需要tool/function calling)  
先抓取模型(4.9GB)：  
```
ollama run llama3.1:8b
```
然后修改config.yaml文件，加入llama3.1:8b的配置信息。  
```
models:
    - name: "Llama3.1:8B"
        provider: ollama
        model: llama3.1:8b
        apiBase: http://localhost:11434
        roles:
            - chat
            - autocomplete
        capabilities:
            - tool_use
```
然后确认在continue底部左下角的mode选中的是Agent。  
根据Continue的说明，如果模型有某个工具调用的能力，就可以直接调用，不需要你教他怎么使用。  
可以先让模型读一个文件测试一下，然后写入文件，然后读一个工作区所有文件并列出目录，确认功能正常后再赋予复杂的任务。  
到这里**免费的无限使用的不依赖网络的**通用AI Agent做完成了。  

## 大模型介绍：DeepSeek-R1
定位： 强推理/逻辑链模型（Reasoning Model）。  
特点： 在出厂时就经过大规模强化学习训练，回答编码问题前会先进行内部“思考（Chain of Thought）”。擅长解决复杂 Bug、算法设计、重构底座架构等需要深度逻辑推理的场景。  
在 Continue 中的角色： 适合放在 Chat（对话）模式下，当你遇到极其晦涩的代码报错或复杂的逻辑需求时调用。 
## 大模型介绍：Qwen2.5
WIP
## 大模型介绍：GLM 系列 (智谱 AI - Zhipu AI)
WIP
## 大模型介绍：Kimi / Moonshot 系列开源蒸馏版/轻量版 (月之暗面)
WIP

## Ollama使用Notes
### 自动加载模型
当VSCode+Continue+Ollama+模型的链路设定完成后，每次时候就不需要在terminal里面运行ollama run了。  
可以直接在chatbot上选择模型名字，然后随便说点什么，这个模型就会自动被加载。  
甚至开启了autocomplete之后，只要在VSCode文档里输入东西触发补全，就会自动在后台加载模型。  
这里会出现一些小问题，不如不小心触发了模型加载，之后玩游戏就会出现显存不够用的麻烦。  

### 多模型加载
如果Ollma设定了不只一个模型，那么每次用ollma run或使用chatbot时候或触发补全的时候，这些模型都会被同时加载。  
这些模型都共用同一个端口，比如11434。  
但是，每次加载一个模型，GPU内存都会被占用一部分。比如: 正常情况下，笔记本5080(16GB)的显存占用为0.7/16GB。  
- 加载qwen2.5-coder:7b之后占用为5.7/16GB。  
- 加载llama3.1:8b之后占用为6.4/16GB。  
- 同时加载两者之后占用为11.4/16GB。  

### 常用Ollama命令
除了
```
ollama run model_name
```
用如下命令可以查看当前加载的模型：
```
ollama ps
```
用如下命令可以卸载模型：
```
ollama stop model_name
```



