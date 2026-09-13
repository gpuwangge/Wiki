# GPU 功耗模型(Power Modeling)

## 1 动态功耗基础计算 (Dynamic Power Base Calculation)
预处理缩放因子 (Pre-scaling):  
- SCALE_pre = (gpu_active * timeprd) / 1000
- SCALE = (1 / SCALE_pre) * 0.9

基础动态功耗 (Base Dynamic Power):  
- dyn_sc_pre = (dyn1 + dyn2 + dyn3) * 1000

其中 dyn1, dyn2, dyn3 是不同层级计数器的加权和：  
- dyn1 = Σ(PRI_counter * PRI_COEF) (18 个主计数器)
- dyn2 = Σ(SEC_counter * SEC_COEF) (19 个次级计数器)
- dyn3 = Σ(TER_counter * TER_COEF) (6 个三级计数器)

## 2 电压与频率缩放 (Voltage and Frequency Scaling)
这部分用于根据 DVFS(动态电压频率调整)状态调整功耗。  
volt_scale = (volt / volt_org)²  
freq_scale = new_freq / org_freq  

## 3 漏电功耗 (Leakage Power)
单核漏电 (SC Leakage):  
leak_sc = base_leak_coef * (volt / 0.8)³  
(基准电压为 0.8V)  

## 4 总 GPU 功耗 (Total GPU Power)
这是最终的聚合公式，结合了动态功耗、漏电功耗以及核心数量/绑定比例。  
新的单核动态功耗 (New SC Dynamic Power):  
gpu_sc_new = freq_scale * dyn_sc * volt_scale * (new_sc_bound_ratio * new_num_sc) / (org_sc_bound_ratio * org_num_sc)  

新的顶层/缓存动态功耗 (New Top Level Dynamic Power):  
gpu_top_new = freq_scale_top * dyn_igpu * volt_scale_top * (new_l2_bound_ratio / org_l2_bound_ratio)  

## 总功耗 (Total Power):
total_power = gpu_sc_new + sc_leakage + gpu_top_new + igpu_leakage  
