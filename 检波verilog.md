```matlab
`timescale 1ns / 1ps

module RF_Rectifier #(
    // 这里的位宽填入你 DAS_Core 算出来的那个最终位宽
    // 比如 (ADC_WIDTH + WEIGHT_WIDTH + $clog2(CH_NUM))
    parameter DATA_WIDTH = 23 
)(
    input  wire                   clk,
    input  wire                   rst_n,
    
    // 输入接口 (接 DAS_Core 的输出)
    // 注意：这里用 signed 声明，明确它是有符号的 2的补码
    input  wire signed [DATA_WIDTH-1:0] data_in,       
    input  wire                         data_in_valid, 

    // 输出接口 (准备接 FIR 滤波器的输入)
    // 经过绝对值处理后，数据变成了无符号数 (Unsigned)
    output reg         [DATA_WIDTH-1:0] data_out,      
    output reg                          data_out_valid 
);

    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            data_out       <= {DATA_WIDTH{1'b0}};
            data_out_valid <= 1'b0;
        end else begin
            // Valid 信号打一拍，和数据对齐
            data_out_valid <= data_in_valid; 
            
            if (data_in_valid) begin
                // 判断最高位（符号位）。如果是 1，说明是负数
                if (data_in[DATA_WIDTH-1] == 1'b1) begin
                    // 负数求绝对值：按位取反，末位加 1 (补码规则)
                    data_out <= (~data_in) + 1'b1;
                end else begin
                    // 正数：直接输出
                    data_out <= data_in;
                end
            end
        end
    end

endmodule
```

