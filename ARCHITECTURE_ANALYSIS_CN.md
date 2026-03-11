# RS(255,239) 译码器模块工作机制解析（基于 Berlekamp-Massey）

本文档对应本仓库中的 RTL 代码，说明各模块输入输出关系、时序关系与关键代码含义。

## 1. 顶层数据通路（RSdecoder）

顶层 `RSdecoder` 串接 6 个核心功能块：

1. `SyndromeCalc`：对 255 个接收码元计算 16 个伴随子。
2. `Reformulated_BM`：根据伴随子迭代求误差位置多项式 `Lambda(x)`。
3. `Chien_Search`：对 `Lambda(x)` 做 Chien 搜索，输出当前位置是否为错误位置。
4. `Forney`：按 Forney 公式计算错误值（幅度）`Error_approx`。
5. `Shift_Reg`：将原始输入延时到和错误定位/估值时序对齐。
6. `ErrorCorrect`：按错误位置与错误值对延时后的数据逐符号纠正。

顶层连接关系见 `RTL/RSdecoder.v`：`Scalc_done` 触发 BM，`BM_done` 触发 Chien/Forney，最终 `Error_symbol + Error_approx` 驱动纠错输出。 

---

## 2. 伴随子计算模块（SyndromeCalc）

### 2.1 核心思路

- 使用 Horner 形式迭代：
  \[
  S_j \leftarrow r_i + \alpha^j S_j
  \]
- 其中 `j=1..16`，对应 RS(255,239) 的 `2t=16` 个伴随子。

### 2.2 代码对应

- `Serial_machine_cnt` 在 `sync` 拉低后从 1 计到 255，用来标记一帧 255 符号处理进度。
- 每个时钟对 `Syndrome[1..16]` 执行 `data_in ^ data_alpha_pow_j[j]`，其中 `data_alpha_pow_j` 由 `GF_constMul` 完成常数乘法。
- 当计数到 `n` 时，置位 `Scalc_done` 表示伴随子计算完成。
- 通过 `Syndrome_shift_reg + shift_out_cnt` 将 16 个伴随子并转串输出到 `Syndrome_out`，供 BM 模块逐拍读入。

### 2.3 设计特点

- 使用常数乘法器（`GF_constMul`）代替通用乘法器，硬件代价低。
- 伴随子输出采用串行化，减少跨模块总线宽度。

---

## 3. 改进 BM 模块（Reformulated_BM）

### 3.1 功能

由串行输入的伴随子计算误差位置多项式：
\[
\Lambda(x)=1+\lambda_1x+...+\lambda_8x^8
\]
并生成判别量 `Delta`（失配值/差值）。

### 3.2 关键寄存器语义

- `Lambda_temp[0..8]`：当前迭代的 `\Lambda` 系数。
- `T[0..8]`：BM 中的辅助多项式。
- `gamma`：比例因子（用于无除法更新）。
- `polynomial_degree`：当前 `\Lambda` 阶数。
- `Syndrome_buffer[1..9]`：伴随子移位缓存，用于与 `Lambda` 卷积求 `Delta`。

### 3.3 关键流程

1. `Scalc_done` 到来后初始化 `Lambda=1`、`T=1`、`gamma=1`、`L=0`。
2. 连续 16 拍迭代（`iteration_cnt<=15`）。
3. 通过 9 路 `GF_mul` + XOR 求得 `Delta`。
4. 满足更新条件 `(Delta!=0 && 2L<=r)` 时更新 `gamma/L/T`；否则 `T` 左移。
5. `Lambda` 按 `gamma*Lambda + Delta*x*T` 更新。
6. 第 15 次迭代后 `BM_done=1`，将 `Lambda0..8` 输出有效化。

---

## 4. Chien 搜索模块（Chien_Search）

### 4.1 功能

逐符号测试 `\Lambda(\alpha^{-i})` 是否为 0，若为 0 则当前位置为错误位置。

### 4.2 代码实现逻辑

- BM 结束后把 `Lambda0..8` 装入本地寄存器。
- 每拍对 `Lambda[1..8]` 分别乘固定常数（相当于下一点的递推评估），由 `GF_constMul` 完成。
- 将 `Lambda[0..8]` 全部 XOR；若结果为 0，则 `flag=1`（检测到错误符号位置）。
- `ErrSymbol_ShiftReg` 将检测标志延时对齐后输出为 `Error_symbol`。
- `End_Error_symbol` 用 `end_operation_cnt` 在末符号处给出最终位置标志。

---

## 5. Forney 错误值计算模块（Forney）

### 5.1 功能

计算每个错误位置处的错误幅值：
\[
E_i = \frac{\Omega(\alpha^{-i})}{\Lambda'(\alpha^{-i})}
\]

### 5.2 模块分解

- `Omega` 生成：利用 `Delta` 的时序值构造误差评估多项式 `\Omega(x)` 系数。
- `Omega_reg/Omega_alpha`：递推评估 `\Omega(\alpha^{-i})`。
- `Lambda_odd/diff_Lambda`：仅取奇次项形成形式导数 `\Lambda'(x)` 的评估链。
- `ROM_INV`：查表求逆 `1/\Lambda'(\alpha^{-i})`。
- `GF_mul_clk`：将 `Omega` 与逆元相乘得到 `Error_approx`。

### 5.3 时序配合

- `BM_done` 触发一次帧内初始化。
- `end_operation_cnt` 计到 14 时锁存 `Error_approx_latch`，用于末符号补偿纠错。

---

## 6. 纠错输出模块（ErrorCorrect）

### 6.1 功能

在对齐后的数据流上做条件异或：

- 若当前位置 `Error_symbol=1`，输出 `data_shifted ^ Error_approx`。
- 否则透传 `data_shifted`。
- 对最后一个符号使用 `End_Error_symbol` 与锁存的 `Error_approx_latch`。

### 6.2 时序

- 由 `Forney` 的 `end_operation_cnt==14` 触发 `Serial_machine_cnt` 从 1 开始，持续覆盖 255 个符号纠错窗口。

---

## 7. 有限域算术基础模块

### 7.1 `GF_mul`

- 组合逻辑乘法器。
- 流程：`mul_B` 多项式基 → 弱对偶基扩展 → 与 `mul_A` 内积得 `prod_dual` → 转回多项式基。

### 7.2 `GF_mul_clk`

- `GF_mul` 的时钟版（寄存输出），便于打拍与时序收敛。

### 7.3 `GF_constMul`

- 乘以固定域常数的专用乘法器。
- `dual_const` 直接给出常数在弱对偶基下系数，适合大量 `\alpha^j` 常数乘场景。

---

## 8. 仿真辅助模块（Simulation_tb）

- `RSEncoder.v`：系统 RS 编码器（前 239 信息 + 后 16 校验）。
- `data_generate.v`：生成帧同步与测试数据。
- `Interleaver.v/Deinterleaver.v`：交织/解交织（更贴近链路场景）。
- `ErrorInjection.v`：人为注错。
- `RSdecoder_fixed_data_tb.v`：固定向量激励的译码测试入口。

这些文件用于验证链路，不属于译码核心 datapath，但可帮助观察纠错性能。

---

## 9. 模块协同总结（按时间顺序）

1. `sync` 拉低后，`SyndromeCalc` 对一帧 255 符号累计伴随子。
2. `Scalc_done` 拉高，BM 开始 16 拍迭代输出 `Lambda(x)` 与 `BM_done`。
3. `BM_done` 后，Chien 与 Forney 同步运行：
   - Chien 给出每拍错误位置 `Error_symbol`；
   - Forney 给出每拍错误值 `Error_approx`。
4. 输入数据经 `Shift_Reg` 延时后与上述结果对齐。
5. `ErrorCorrect` 在错误位置执行异或纠正，输出 `data_out`。

整体属于**流式串行译码架构**：以较少并行资源换取稳定吞吐和较低硬件面积。

## 10. 代码阅读注意点

- 工程依赖若干未在 `RTL` 目录直接给出的模块/IP（如 `Shift_Reg`、`ErrSymbol_ShiftReg`、`ROM_INV`），通常由 FPGA IP Catalog 或其他文件生成。
- `Simulation_tb/RSdecoder_tb/RSdecoder_tb.v` 的端口连接与 `RTL/RSdecoder.v` 版本并不完全一致，实际仿真建议以 `fixed_data_sim` 或修正后的 testbench 为准。
