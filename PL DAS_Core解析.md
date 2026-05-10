```verilog
`timescale 1ns / 1ps
// ============================================================================
// 模块名称:    DAS_Core
// 模块描述:    参数化延迟求和 (DAS) 波束合成核心模块
//              实现了用于通道时间对齐、变迹加权 (Apodization) 和信号求和的
//              实时硬件流水线 (Real-time Pipeline)。
// ============================================================================

module DAS_Core #(
    // --- 系统级参数 (实例化时可自由修改) ---
    parameter CH_NUM       = 8,      // 接收通道的总数量
    parameter ADC_WIDTH    = 12,     // ADC 输入数据的位宽 (默认12-bit)
    parameter DELAY_WIDTH  = 10,     // 延迟指针的位宽 (决定了最大延迟深度，必须是 log2(BRAM_DEPTH))
    parameter BRAM_DEPTH   = 1024,   // 环形缓存的物理深度 (必须是 2 的 DELAY_WIDTH 次方)
    parameter WEIGHT_WIDTH = 8       // 变迹权重 (抑制权重) 的位宽
)(
    input  wire clk,                 // 系统时钟 (驱动流水线运转的心脏)
    input  wire rst_n,               // 异步复位信号 (低电平有效)
    input  wire adc_valid,           // 数据有效标志位 (高电平时表示当前射频流数据有效)
    
    // --- 展平的数据接口 (Flattened Data Interfaces) ---
    // 注意: 为了在标准 Verilog 中支持参数化的端口，我们将多维数组"拍扁"成了单根极宽的总线。
    // 内部逻辑会再将这根宽总线切片分配给各个通道。
    input  wire [CH_NUM * ADC_WIDTH - 1 : 0]    adc_data_in, // 包含所有通道ADC数据的总线
    input  wire [CH_NUM * DELAY_WIDTH - 1 : 0]  delay_in,    // 包含所有通道延迟参数的总线
    input  wire [CH_NUM * WEIGHT_WIDTH - 1 : 0] weight_in,   // 包含所有通道权重参数的总线
    
    // --- 输出接口 ---
    // 输出位宽在经过乘法和加法累加后会变宽，防止数据溢出。
    // 计算公式: 最终位宽 = ADC位宽 + 权重位宽 + log2(通道数)
    output reg  [(ADC_WIDTH + WEIGHT_WIDTH + 3) - 1 : 0] beamformed_out, // 波束合成最终输出数据
    output reg                                           out_valid       // 输出数据有效标志位
);

    // ========================================================================
    // 1. 全局写指针逻辑 (所有通道共享一个时间基准)
    // ========================================================================
    reg [DELAY_WIDTH - 1 : 0] wr_ptr;
    
    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            wr_ptr <= {DELAY_WIDTH{1'b0}}; // 复位时，写指针清零
        end else if (adc_valid) begin
            wr_ptr <= wr_ptr + 1'b1;       // 数据有效时，每个时钟周期写指针加1 (溢出后自动绕回)
        end
    end

    // ========================================================================
    // 2. 通道处理流水线 (利用 generate 语句根据 CH_NUM 动态批量生成)
    // ========================================================================
    // 定义一个内部数组，用于暂时存放各个通道乘法器算出来的结果，等待进入求和树
    wire [(ADC_WIDTH + WEIGHT_WIDTH) - 1 : 0] mult_result_array [0 : CH_NUM - 1];
    
    genvar i; // generate 专属的循环变量
    generate
        for (i = 0; i < CH_NUM; i = i + 1) begin : CHANNEL_PROCESS_BLOCK
            
            // --- 从展平的宽总线中，精准切出属于当前通道(第i个)的数据 ---
            // 语法说明: [起始位置 +: 位宽] 是 Verilog-2001 的切片语法
            wire [ADC_WIDTH - 1 : 0]    ch_adc_data = adc_data_in [i * ADC_WIDTH +: ADC_WIDTH];
            wire [DELAY_WIDTH - 1 : 0]  ch_delay    = delay_in    [i * DELAY_WIDTH +: DELAY_WIDTH];
            wire [WEIGHT_WIDTH - 1 : 0] ch_weight   = weight_in   [i * WEIGHT_WIDTH +: WEIGHT_WIDTH];
            
            // --- 环形缓存 (综合器会自动将其推断为 FPGA 内部的 BRAM 硬核) ---
            reg [ADC_WIDTH - 1 : 0] ring_buffer [0 : BRAM_DEPTH - 1];
            
            // --- 读指针计算 (利用无符号减法的自然溢出，实现固定距离的回溯) ---
            wire [DELAY_WIDTH - 1 : 0] rd_ptr = wr_ptr - ch_delay;
            
            // --- 流水线寄存器 (用于锁定数据，满足时序要求) ---
            reg [ADC_WIDTH - 1 : 0] delayed_data;                      // 存放对齐后的历史数据
            reg [(ADC_WIDTH + WEIGHT_WIDTH) - 1 : 0] mult_data;        // 存放乘法结果
            
            // 同步的存储器读写与乘法操作
            always @(posedge clk) begin
                if (adc_valid) begin
                    // 步骤A: 将当前时刻的新数据写入环形缓存
                    ring_buffer[wr_ptr] <= ch_adc_data;
                    
                    // 步骤B: 抽出对齐后的历史数据
                    delayed_data <= ring_buffer[rd_ptr];
                    
                    // 步骤C: 变迹乘法 (综合器会自动推断并调用 DSP48E 乘法器硬核)
                    mult_data <= delayed_data * ch_weight;
                end
            end
            
            // 将当前通道的乘法结果，通过物理连线接入到全局的求和数组中
            assign mult_result_array[i] = mult_data;
            
        end
    endgenerate

    // ========================================================================
    // 3. 通道求和逻辑 (加法树 Summation Logic)
    // ========================================================================
    // 架构师注: 
    // 对于较少的通道数 (如 8 通道)，使用 behavioral 的 for 循环累加，综合器能够较好地处理。
    // 但如果要扩展到 64 或 128 通道，这种串行加法会导致极长的组合逻辑延迟，无法满足时序。
    // 届时必须将其替换为严格打拍的二叉树加法网络 (Pipelined Binary Adder Tree)。
    
    integer j;
    reg [(ADC_WIDTH + WEIGHT_WIDTH + 3) - 1 : 0] sum_acc; // 累加寄存器
    
    always @(posedge clk) begin
        if (adc_valid) begin
            sum_acc = 0; // 每次计算前，累加器清零
            
            // 将 8 个通道的乘法结果累加
            for (j = 0; j < CH_NUM; j = j + 1) begin
                sum_acc = sum_acc + mult_result_array[j];
            end
            
            // 将累加结果打入最终的输出寄存器
            beamformed_out <= sum_acc;
            out_valid      <= 1'b1;
        end else begin
            out_valid      <= 1'b0;
        end
    end

endmodule
```
#### 第一大块：引脚定义与“展平总线” (Flattened Bus)

Verilog

```
    input  wire [CH_NUM * ADC_WIDTH - 1 : 0]    adc_data_in,
```

- **痛点**：如果你有 8 个通道，每个通道 12 根线，直观的做法是定义一个二维数组（比如 8 行 12 列）。但在 FPGA 的 IP 核封装标准中（如 AXI Stream 接口），**所有数据必须走一根线缆（总线）**。
    
- **做法**：把这 8 个 12 位的线束，并排绑在一起，变成一根宽达 `8 * 12 = 96` 位的超级排线。
    
- **好处**：当你在 Vivado 的 Block Design 里画图连线时，8 个通道的数据只需要连一条线，界面极其清爽。
    

#### 第二大块：全局写指针 (Global Write Pointer)

Verilog

```
    reg [DELAY_WIDTH - 1 : 0] wr_ptr;
    // ... wr_ptr <= wr_ptr + 1'b1;
```

- **物理意义**：这是一个“全局节拍器”。既然 8 个通道是一起采样的，那么它们存放新数据的时间基准必须是绝对统一的。所以我们只用**一个写指针**，指挥 8 个 BRAM 同步存放数据。
    

#### 第三大块：核心并发流水线 (`generate for` 块)

这是整段代码最精妙、最具硬件思维的地方。Vivado 会把这部分代码**像 3D 打印一样，并排复制出 8 个一模一样的物理车间**。

**1. 极度优雅的切片解包语法 `+:`**

Verilog

```
wire [ADC_WIDTH - 1 : 0] ch_adc_data = adc_data_in [i * ADC_WIDTH +: ADC_WIDTH];
```

- 刚才我们把 96 根线绑成了一捆。现在分到了各个通道，需要把属于自己的那 12 根线抽出来。
    
- `+:` 是硬件描述语言独有的切片语法。它的意思是：**从 `i * ADC_WIDTH` 这个位置开始，往上截取 `ADC_WIDTH` 位的数据。**
    
- 举例：当 `i=0` 时，截取 `[0 到 11]`；当 `i=1` 时，截取 `[12 到 23]`。这样，每个通道就极其精准地拿到了自己的专属数据。
    

**2. 同步存储与变迹乘法**

Verilog

```
    ring_buffer[wr_ptr] <= ch_adc_data;       // 存入当前点
    delayed_data        <= ring_buffer[rd_ptr]; // 抽出历史点
    mult_data           <= delayed_data * ch_weight; // 乘以权重
```

- 在这一个 `always @(posedge clk)` 块中，这三句话是**同时发生**的。
    
- 在同一个时钟上升沿：新的数据刚存进 BRAM，对齐好的历史数据刚被读出来，而上一个读出来的数据正送进 DSP48E 乘法器中进行权重计算。这就叫**全流水线操作 (Fully Pipelined)**，它能确保每个时钟周期都能吐出一个有效数据，吞吐量达到极限。
    

#### 第四大块：求和网络 (Summation Logic)

Verilog

```
    for (j = 0; j < CH_NUM; j = j + 1) begin
        sum_acc = sum_acc + mult_result_array[j];
    end
```

- **注意这里的 `=`（阻塞赋值）**：前面用的都是 `<=`（非阻塞赋值，代表同时触发的寄存器）。这里在 `always` 块里用了 `=`，意思是让综合器把这 8 个加法动作**串联**起来，生成一串组合逻辑加法器。
    
- **物理映射**：电流会依次穿过 加法器0 -> 加法器1 -> ... -> 加法器7，最后算出一个总和 `sum_acc`。
    
- **为什么要打入寄存器 `beamformed_out <= sum_acc;`**：算出来的总和如果直接输出，信号会极度不稳定（毛刺极多）。所以最后必须用一个 `<=` 把结果牢牢锁在一个寄存器里，确保输出给下游的数据干干净净、四平八稳。