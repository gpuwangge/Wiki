# Raytracing Context 包含什么

| 类别 | RCE Context 内容 |
|---|---|
| 射线参数 | `Origin`、`Direction`、`tMin/tMax`、`Ray Flags`、`Hit Mask` |
| 追踪状态机 | 当前命中/未命中阶段、`Hit/Miss Shader Index`、`Payload`、缓冲区引用、`Ray Query` 状态（`Proceed/Committed`） |
| 加速结构引用 | `TLAS/BLAS` 引用 ID、`SBT`（Shader Binding Table）偏移、`Instance/Geometry ID` |
| 遍历栈与常数 | 递归深度、`BVH` 遍历栈指针、`Push Constants` / `Tunable Parameters` |
| 线程组管理 | 线程 ID 分发状态、屏蔽掩码（`Enable Mask`）、预取队列指针 |

