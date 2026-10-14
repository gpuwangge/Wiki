# 当我们谈到CPU Architecture时，我们在谈论什么？
## CPU Core
目的：把指令执行拆成多个阶段，让多条指令重叠执行，提高吞吐量；不代表单条指令更快。  
基本过程：取指 → 解码/读寄存器 → 执行 → 访存 → 写回。这是经典五级模型。  
什么会让它停下来：数据依赖、资源冲突、分支；对应 forwarding / stall、资源调度、分支预测。  

教学用的经典五级 RISC 模型，不代表现代 CPU 只有五级，也不是每条指令在每一级都做有用工作  
| 阶段                          | 解释                     |
| --------------------------- | -------------------------- |
| IF — Instruction Fetch      | 根据 PC 取指令                  |
| ID — Decode / Register Read | 解码，读取源寄存器                  |
| EX — Execute                | ALU 运算，或计算访存地址；分支处理位置依实现而定 |
| MEM — Memory Access         | Load / Store 访问数据存储层级      |
| WB — Write-back             | 将结果写回寄存器                   |

Hazards
| 类型         | 原因            | 常见处理                           |
| ---------- | ------------- | ------------------------------ |
| Structural | 多条指令竞争同一个硬件资源 | 增加资源/端口、调度或 stall              |
| Data       | 指令依赖尚未可用的结果   | Forwarding、等待；renaming 消除假依赖   |
| Control    | 分支结果或目标尚未确定   | Branch prediction、投机执行；预测错误时恢复 |

数据依赖的例子  
```
ADD r1, r2, r3
SUB r4, r1, r5
```
SUB 需要 ADD 产生的 r1，这是 RAW — Read After Write  
Forwarding / bypassing（转发 / 旁路）：ADD 已经算出结果，但还没写回寄存器，就直接把结果送到 SUB 的 ALU 输入，不必等写回后再读取。  

另一个例子  
```
LOAD r1, [r2]
ADD  r3, r1, r4
```
load 数据于 MEM 末尾可用的模型中，下一条指令立即在 EX 使用它，即使有 forwarding，通常仍需要一个 stall。  

## 非CPU Core辅助组件
Core 负责执行指令；其余部件负责给它供数据、连接其他核心和设备，以及管理整颗芯片。
| 部件                     | 作用                  |
| ---------------------- | ------------------- |
| 共享缓存，如 L3 / LLC        | 缓存数据，减少访问内存         |
| 片上互连，Ring / Mesh / NoC | 连接各个 core、缓存和控制器    |
| 内存控制器                  | 管理对外部 DRAM 的读写      |
| 缓存一致性逻辑                | 保证多个 core 的缓存数据保持一致 |
| I/O 控制器，如 PCIe         | 连接 GPU、SSD 等设备      |
| 电源与时钟管理                | 管理频率、电压和功耗          |


## CPU Fixed Function单元举例
CPU 是通用可编程处理器，但内部也可以有专用硬件加速某些操作；通用不代表所有工作都靠普通 ALU 完成。  
最典型的例子是 AES 加密硬件：CPU 通过 AES-NI 指令调用硬件，直接完成 AES 的特定计算步骤，不用靠普通加减、移位指令逐步实现。  


# CPU Architecture Pipeline 知识整理

## 1. CPU 架构与实现相关术语

| 方向 | 术语 | 中文解释 | 主要考察什么 |
|---|---|---|---|
| Pipeline | Fetch | 取指，从指令缓存或内存中获取下一条指令。 | 是否理解指令进入流水线的过程 |
| Pipeline | Decode | 译码，解析指令的操作类型、源寄存器和目标寄存器等信息。 | 是否理解指令译码 |
| Pipeline | Execute | 执行，使用 ALU 等执行单元完成算术、逻辑或其他操作。 | 是否理解指令执行 |
| Pipeline | Retire | 提交已完成且可以按程序顺序生效的指令，正式更新架构状态。 | 是否理解乱序执行中的提交机制 |
| 指令调度 | Out-of-Order Execution | 乱序执行。处理器可以先执行操作数已准备好的后续指令，不必等待前面尚未就绪的指令。 | 是否理解乱序执行机制 |
| 指令调度 | ROB (Reorder Buffer) | 重排序缓冲区，用于跟踪在途指令，并支持按程序顺序提交结果及处理异常或错误预测。 | 是否理解指令状态管理和按序提交 |
| 指令调度 | Reservation Station | 保留站，保存等待执行的操作及其操作数状态；当操作数就绪时，可以将操作发送给执行单元。 | 是否理解指令等待、唤醒和调度 |
| 数据依赖 | RAW (Read After Write) | 读后写依赖：后面的指令需要读取前面指令将要写入的值，也称真实数据依赖。 | 是否理解指令之间的数据依赖 |
| 数据依赖 | Register Renaming | 寄存器重命名：将架构寄存器映射到不同的物理寄存器，消除 WAR 和 WAW 等名称依赖。 | 是否理解如何提高指令级并行度 |
| 数据依赖 | Forwarding | 数据前递：将执行结果直接传递给需要它的后续流水线阶段，避免等待结果写回寄存器后再读取。 | 是否理解如何减少数据相关导致的停顿 |
| 分支处理 | Branch Prediction | 分支预测：根据历史信息等预测条件分支的走向，提前获取和执行可能需要的指令。 | 是否理解分支对性能的影响 |
| 分支处理 | Speculative Execution | 推测执行：在分支结果确定之前，先执行预测路径上的指令；如果预测错误，需要清除错误路径上的推测状态。 | 是否理解预测执行和错误恢复 |
| 微架构 | ALU (Arithmetic Logic Unit) | 算术逻辑单元，负责加减法、逻辑运算、比较等操作，具体能力取决于设计。 | 是否了解基本执行单元 |
| 微架构 | Load/Store Unit (LSU) | 加载/存储单元，负责处理读写内存的指令，并协调地址计算、缓存访问和内存操作。 | 是否理解 CPU 与内存子系统的交互 |
| 微架构 | Execution Port | 执行端口，连接调度逻辑与一个或多个执行单元，决定哪些操作可以被派发到哪些单元。 | 是否理解执行资源分配和吞吐量限制 |
| 仿真建模 | Cycle-accurate Model | 周期精确模型，模拟硬件事件发生的时钟周期，适合分析时序、流水线停顿和性能。 | 是否有详细微架构建模经验 |
| 仿真建模 | Functional Model | 功能模型，主要模拟指令或硬件行为是否正确，通常不精确模拟每个操作的周期时序。 | 是否理解功能正确性与时序精度的区别 |

## 2. 核心关系

### 2.1 指令执行流程

典型的简化流程可以表示为：

`Fetch → Decode → Dispatch / Issue → Execute → Writeback → Retire`

- **Fetch / Decode**：获取并解析指令。
- **Dispatch / Issue**：将指令分配给相应的调度结构，并在条件满足时发往执行单元。具体术语和阶段边界依 CPU 设计而异。
- **Execute**：执行实际操作。
- **Writeback**：将结果写入物理寄存器或其他结果存储结构。
- **Retire**：按程序顺序提交指令，使其架构状态正式生效。

注意：不同 CPU 的流水线阶段划分并不完全相同。尤其在乱序 CPU 中，执行可以乱序，但通常仍需要按程序顺序提交。

### 2.2 数据依赖与寄存器重命名

假设有两条指令：

```asm
ADD R1, R2, R3
SUB R4, R1, R5
```

第二条指令需要读取第一条指令生成的 `R1`，因此存在 RAW 真实数据依赖。

- **Forwarding** 可以在结果产生后直接传递数据，减少等待。
- **Register Renaming** 可以消除 WAR 和 WAW 名称依赖，但不能消除 RAW 真实数据依赖。

### 2.3 乱序执行与提交

乱序执行让处理器在某条指令等待数据时，先执行其他已经准备好的指令。

- **Reservation Station**：等待操作数就绪并参与调度。
- **ROB(Reorder Buffer)**：跟踪在途指令，支持按序提交。
- **Retire**：按程序顺序提交已完成的指令。

这些机制相互配合，让 CPU 在提高执行并行度的同时，尽可能维持正确的架构行为。

### 2.4 功能模型与周期精确模型

| 对比 | Functional Model | Cycle-accurate Model |
|---|---|---|
| 主要目标 | 验证功能是否正确 | 验证功能并分析周期级时序 |
| 执行时间 | 通常不代表真实硬件延迟 | 模拟硬件事件的周期时序 |
| 性能分析 | 通常能力有限 | 可以分析停顿、吞吐量和延迟 |
| 实现复杂度 | 通常较低 | 通常较高 |

**重点：** 功能正确不代表时序正确；周期精确模型也不一定自动代表真实硬件的全部性能，模型精度仍取决于模拟的微架构细节。

# What happens during an instruction fetch

1. PC（Program Counter）：记录下一条待获取指令的地址。
2. Instruction Fetch：根据 PC 地址，从指令缓存（I-cache）中获取指令。
3. Cache Miss：如果 I-cache 未命中，需要从更低层级的缓存或内存获取指令。
4. Instruction Decode：将取到的指令交给译码阶段。
5. PC Update：更新 PC，指向后续指令；如果发生分支，则可能跳转到新的地址。

Does fetch always retrieve one instruction per cycle?  
No. Modern CPUs may fetch multiple instructions per cycle, depending on the architecture and available resources.  

What happens after fetch?  
The instruction is decoded, and its operands and execution requirements are identified.  

现代 CPU 中的几个概念：
- Branch Prediction：预测分支是否跳转以及目标地址，让 CPU 能提前获取后续指令。
- Instruction Queue / Buffer：暂存已获取、等待译码的指令。
- Out-of-Order CPU：Fetch 通常先获取指令，之后指令可能经过译码、重命名和调度，再乱序执行。
- Pipeline：Fetch、Decode、Execute 等阶段可以同时处理不同的指令。




