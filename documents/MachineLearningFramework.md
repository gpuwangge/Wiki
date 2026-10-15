# AI与ML/DL知识树
```
AI
├── 基础概念与方法
│   ├── 机器学习（ML）
│   └── 深度学习（DL）
│
├── 工程工具与框架
│   ├── ML 框架
│   │   ├── PyTorch
│   │   └── TensorFlow
│   └── 推理引擎与部署
│       ├── ncnn
│       ├── MNN
│       └── ExecuTorch
│
└── GPU 计算与执行后端
    ├── CUDA 相关
    └── Vulkan Compute 相关
```

# ML Framework软件栈
```
模型与应用
例如：图像分类、目标检测、语言模型
                    │
                    ▼
ML framework
PyTorch / TensorFlow
定义模型、自动求导、训练、组织计算
                    │
         ┌──────────┴──────────┐
         ▼                     ▼
框架直接执行                 导出并部署模型
例如 CPU / CUDA 路径         例如 ncnn / MNN / ExecuTorch
                               │
                               ▼
                         Vulkan 执行后端
                         管理资源、管线、命令、同步
                               │
                               ▼
                         Compute shaders
                         执行具体 ML 算子
                               │
                               ▼
                              GPU
```
| 名称                         | 属于什么                     | 主要职责                         | 与其他概念的关系                                     |
| -------------------------- | ------------------------ | ---------------------------- | -------------------------------------------- |
| ML framework               | 软件类别：机器学习框架              | 提供张量计算、模型构建、自动求导、训练和推理能力     | PyTorch、TensorFlow 都属于这一类                    |
| PyTorch                    | 具体的 ML framework         | 用 Tensor 和模型模块表达计算，支持自动求导与训练 | 可以使用 CPU、CUDA 等执行后端；端侧部署可使用 ExecuTorch       |
| TensorFlow                 | 具体的 ML framework / ML 平台 | 构建、训练、部署机器学习模型               | 与 PyTorch 是同类框架，不是上下层关系                      |
| Vulkan                     | 底层 GPU API               | 管理 GPU 资源、管线、命令提交和同步         | 可以作为 ML 推理引擎的 GPU 执行后端                       |
| Vulkan compute shader      | GPU 计算程序                 | 执行矩阵乘法、卷积等具体计算               | 被 Vulkan 后端通过 compute pipeline 和 dispatch 调用 |
| ncnn / MNN                 | ML 推理框架或引擎               | 加载模型、组织算子执行、管理部署             | 提供 Vulkan GPU 推理后端                           |
| ExecuTorch Vulkan delegate | PyTorch 生态中的端侧执行后端       | 执行分配给 Vulkan 的模型子图           | 使用 Vulkan 和 compute shaders 加速推理             |

# PyTorch 和 TensorFlow 的设备管理
PyTorch默认使用 CPU。即使电脑有 GPU、安装的 PyTorch 也支持 CUDA，只要你没有指定设备或修改默认设备，普通新建的 Tensor 和模型参数仍然在 CPU 上。  
TensorFlow 与 PyTorch 不一样：如果 TensorFlow 能识别到可用 GPU，并且某个算子有 GPU 实现，它默认会 优先 把这个算子放到 GPU 上执行；没有可用 GPU，则在 CPU 上执行。  

| 情况      | PyTorch                   | TensorFlow                |
| ------- | ------------------------- | ------------------------- |
| 不显式指定设备 | 普通新建 Tensor 默认在 CPU       | 自动选择设备，支持 GPU 的算子优先用 GPU  |
| 想使用 GPU | 通常显式把模型和输入放到 CUDA         | 环境配置正确时，通常不需要额外修改代码       |
| 手动控制设备  | .to("cuda")、device="cuda" | with tf.device("/GPU:0"): |

PyTorch 默认创建 CPU Tensor，通常由开发者显式管理设备；TensorFlow 默认进行自动设备放置，对于有 GPU 实现的算子，会优先使用可用 GPU。但这不代表整个 TensorFlow 程序的所有操作都在 GPU 上执行。  

# Vulkan的ML Framework
部署模型的工具
| 项目                        | 定位                      | Vulkan 在其中做什么                                          |
| ------------------------- | ----------------------- | ------------------------------------------------------ |
| ncnn                      | 高性能神经网络推理框架，面向移动端和跨平台部署 | 使用 Vulkan 加速 GPU 推理                                    |
| MNN                       | 轻量级深度学习框架，支持端侧推理与训练     | Vulkan 是其 GPU 推理后端之一，另有 OpenCL、Metal 等                 |
| ExecuTorch Vulkan backend | PyTorch 生态中的端侧推理后端      | 使用 GLSL compute shaders 实现 PyTorch 算子，在 Vulkan GPU 上执行 |

建议：ncnn 入门 Vulkan ML 算子执行，再用 ExecuTorch 理解 PyTorch 模型如何接入硬件后端；MNN 可以作为另一种 runtime 架构参考。这个顺序的依据是：ncnn 有直接、明确的 Vulkan layer 开发流程，而 ExecuTorch 涉及模型导出、图划分和 delegate 执行，更贴近框架与后端之间的连接。  

| 对比项              | ncnn                                 | MNN                            | ExecuTorch                                         |
| ---------------- | ------------------------------------ | ------------------------------ | -------------------------------------------------- |
| 核心定位             | 高性能神经网络推理框架                          | 轻量级 ML 框架与推理引擎                 | PyTorch 的端侧模型部署方案                                  |
| 与 PyTorch 的关系    | 可以部署转换后的 PyTorch 模型，不属于 PyTorch 本体   | 可以部署转换后的模型，有自己的执行体系            | 属于 PyTorch 生态，承接模型导出与端侧执行                          |
| Vulkan 的角色       | GPU 推理后端                             | 多种计算后端之一                       | delegate：承接分配给 Vulkan 的模型子图                        |
| 适合重点研究           | Layer → pipeline → shader → dispatch | Runtime / Backend 抽象、设备管理、算子实现 | Export → partition → lowering → delegate → runtime |
| 对 Vulkan 工程师的切入点 | 给一个 layer 增加 Vulkan 实现               | 理解并实现后端算子                      | 给 Vulkan delegate 增加算子支持与 shader                   |
| 我建议的学习用途         | 第一站：看懂并修改 GPU 算子                     | 第二种 runtime 设计参考               | 第二站：连接 PyTorch 与硬件后端                               |

## ncnn工作流程
ncnn 文档把“给一个 layer 增加 Vulkan 支持”分成这些步骤：
```
添加 compute shader
        ↓
上传权重数据
        ↓
创建 compute pipeline
        ↓
在 forward 中记录执行命令
        ↓
执行并获取结果
```
一个简单算子读源码：
```
输入 Tensor
    ↓
检查 shape、数据类型、packing
    ↓
选择 shader / pipeline
    ↓
绑定资源与参数
    ↓
dispatch
    ↓
输出 Tensor
```

## ExecuTorch工作流程
```
PyTorch 模型
    ↓
导出计算图
    ↓
Partitioner 判断哪些节点由 Vulkan 支持
    ↓
将支持的子图交给 Vulkan 后端转换
    ↓
序列化部署产物
    ↓
ExecuTorch runtime
    ↓
调用 Vulkan delegate
    ↓
Compute shaders 执行
```

## MNN 工作流程
MNN 通过 Runtime / Backend 抽象计算设备，支持 CPU、Vulkan、OpenCL、Metal、CUDA 等后端；文档也区分“增加新设备后端”和“给已有后端增加算子”。  
```
模型与计算图
    ↓
Runtime / Backend 抽象
    ↓
算子执行实现
    ↓
具体设备与资源管理
```
它适合帮你建立“不能把整个 runtime 写死为 Vulkan”的架构意识。不过，如果当前目标是最快做出一个 Vulkan ML 算子实验，我仍建议先选择 ncnn，不必同时理解两套相近但不同的接口体系。

## 具体实现建议
### 第一阶段：只看一个简单算子
选择 elementwise add 或 ReLU：  
找到 CPU 实现，明确输入、输出和计算语义。  
找到 Vulkan 实现。  
跟踪 Tensor 如何变成 GPU 资源。  
跟踪 pipeline 创建与 dispatch。  
对照 CPU 结果验证正确性。  

### 第二阶段：进入矩阵乘法
用一个小型 Linear / MatMul 工作负载，研究：  
Tensor shape 与实际内存布局。  
Shader 中如何计算索引。  
Workgroup、tiling、数据复用。  
数据传输与设备计算分别耗时多少。  

### 第三阶段：把同一模型接入 ExecuTorch
再关注：  
哪些算子被 Vulkan delegate 接收。  
模型被划分成多少个执行区域。  
导出产物和 runtime 的职责边界。  
算子支持不足时，怎样定位问题。  

这是我建议的实践路线；它把学习范围控制在“一个算子 → 一个小模型 → 一套后端接入流程”，避免直接陷入大模型的复杂性。  

完整流程  
用 PyTorch 开发或获取训练好的模型，通过 pnnx 转换，再用 ncnn 的 Vulkan 后端部署推理。  
```
PyTorch / TensorFlow
定义网络、训练参数、验证模型
        ↓
导出并转换模型
转换成 ncnn 支持的模型结构与权重
        ↓
ncnn 加载模型
组织各个算子的执行
        ↓
Vulkan 后端
通过 compute shaders 执行受支持的 GPU 算子
        ↓
得到推理结果
```
| 组件                   | 职责                         |
| -------------------- | -------------------------- |
| PyTorch / TensorFlow | 定义模型，利用训练数据学习参数；也可以直接执行推理  |
| 模型转换工具               | 把源模型的计算图和权重转换成 ncnn 能识别的表示 |
| ncnn                 | 加载模型、管理数据与算子、组织推理执行        |
| Vulkan 后端            | 管理 GPU 资源、pipeline、命令与同步   |
| Compute shaders      | 执行具体算子，例如矩阵乘法、卷积和激活        |

同时记住三个限制：
- 不一定要自己训练：可以使用别人已经训练好的模型，再转换部署。
- 不一定要用 ncnn：PyTorch / TensorFlow 本身也能推理，ncnn 是另一种部署选择。
- 不一定全部在 Vulkan 上执行：需要启用 Vulkan，并检查模型所用算子的后端支持范围。


