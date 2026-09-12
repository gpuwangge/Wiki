## Bandwidth Validation
带宽一致性校验（Bandwidth Validation）通过比较底层/硬件边缘计数（Ground Truth，基准值）与上层/着色器核心统计（Actual，实测估算值）之间的偏差，来校验数据流量建模或硬件监控（Hardware Counters）的准确性。  

| 符号              | 含义与说明                                                     |
| :-------------- | :-------------------------------------------------------- |
| `N_slice`       | L2 Cache 切片（Slice）总数                                      |
| `N_sc`          | Shader Core（着色器核心）总数                                      |
| `W_AXI`         | AXI 总线宽度（Bits 或 Bytes）                                    |
| `S_beat`        | 单次传输 Beat 的数据大小                                |
| `B_L2_EXT_RD`   | L2 观测到的外部读传输 Beat 数                                       |
| `B_L2_EXT_WR`   | L2 观测到的外部写传输 Beat 数                                       |
| `Σ B_SC_RD_EXT` | 各 Shader 单元（RTU, FTC, LSC, TEX）发起的外部读 Beat 总和             |
| `Σ B_SC_WR`     | 各 Shader 单元（LSC_OTHER, TIB, LSC_WB）发起的写传输 Beat 总和         |
| `M_L2_IN_TOTAL` | L2 接收到的内部请求消息总数                                           |
| `M_NON_DATA`    | 非数据/管理类开销消息（Eviction, Cache Coherency, MMU Table Reads 等） |
| `Σ B_L2_INT_RD` | 各 Shader 单元（FTC, LSC, TEX, OTHER）发起的 L2 内部读 Beat 总和       |

### 1. DDR 读带宽校验 (DDR Read Bandwidth Validation)
跨视角对比内存读带宽：将 **L2 Cache 观测到的外部内存读流量** 与 **Shader Core 各子单元发起的读请求量** 进行交叉校验，以捕获模型中的计数遗漏或接口不一致。
* **Ground Truth (基准值)**
```
BW_DDR_RD_GT = B_L2_EXT_RD × N_slice × W_AXI
```
* **Actual (测量估算值)**
```
BW_DDR_RD_ACT = (Σ B_SC_RD_EXT) × N_sc × S_beat
```
* **Relative Error (相对误差)**
```
Error_DDR_RD = (BW_DDR_RD_ACT - BW_DDR_RD_GT) / BW_DDR_RD_GT
```

### 2. DDR 写带宽校验 (DDR Write Bandwidth Validation)
验证 DDR 写带宽一致性：核对 **L2 写回外部存储的数据量** 与 **Shader Core（如 Tile Buffer/TIB, LSC Writeback 等）刷出的数据量** 是否匹配，确保写通路（Write Path）建模正确。
* **Ground Truth (基准值)**
```
BW_DDR_WR_GT = B_L2_EXT_WR × N_slice × W_AXI
```
* **Actual (测量估算值)**
```
BW_DDR_WR_ACT = (Σ B_SC_WR) × N_sc × S_beat
```
* **Relative Error (相对误差)**
```
Error_DDR_WR = (BW_DDR_WR_ACT - BW_DDR_WR_GT) / BW_DDR_WR_GT
```

### 3. L2 内部带宽校验 (L2 Internal Bandwidth Validation)
评估 L2 缓存内部有效数据吞吐率：排除 Cache 逐出（Eviction）、缓存一致性消息（Coherency）及 MMU 页表查询等非有效数据流量后，验证 **L2 内部真实数据读流量** 的准确性。
* **Ground Truth (基准值)**
```
BW_L2_INT_GT = (M_L2_IN_TOTAL - M_NON_DATA) × N_slice × 512
```
* **Actual (测量估算值)**
```
BW_L2_INT_ACT = (Σ B_L2_INT_RD) × N_sc × S_beat + BUS_READ × S_beat
```