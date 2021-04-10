---
title: TopK 算法的 Verilog 实现
date: 2020-12-26 16:19:34
updated: 2020-12-26 16:19:34
tags: [Hardware]
categories: "硬件"
---

### 算法

TopK 的软件实现一般使用小顶推排序算法来实现，每次数据进入需要不定次数的比较和交换，不利于硬件实现。

在软件算法的硬件实现上，应该避免使用这类带有不确定性的算法，防止逻辑过于复杂。硬件算法实现可以更充分的利用硬件并行化的优势，所以采用冒泡排序，冒泡过程在硬件中即为位移运算，可以并行完成，速度较快。

### 实现

以下贴出了 TopK 算法实现的 verilog 代码。

<!-- more -->

```verilog
module topk #
(
    parameter integer TOTAL_WIDTH   = 48,       // 数据总位数
    parameter integer COMP_WIDTH    = 8 ,       // 需要比较的位数
    parameter integer K_NUM         = 10,       // 保留多少个K
    parameter integer DATA_WIDTH    = 32        // 输出数据位宽
)
(
    input                           clk,
    input                           rst_n,
    input                           clk_gating_en,
    input       [TOTAL_WIDTH-1:0]   indata,
    input                           indata_valid,
    input                           indata_over,
    output      [DATA_WIDTH-1:0]    outdata,
    output                          outdata_valid,
    output                          outdata_over
);

localparam cnt_total = TOTAL_WIDTH * K_NUM / DATA_WIDTH - 1;

reg [TOTAL_WIDTH*K_NUM-1:0] d;
wire over;
reg shift;

wire [K_NUM+1:0] comp;          // [1,    11000,     0]
assign comp[0]       = 1'b0;
assign comp[K_NUM+1] = 1'b1;

generate
    genvar j;
        for (j=0;j<K_NUM;j=j+1) begin: COMPARE_
            assign comp[j+1] = (indata[COMP_WIDTH-1:0] > d[(j*TOTAL_WIDTH)+:COMP_WIDTH]) ? 1'b1 : 1'b0;
        end
endgenerate

generate
    genvar i;
        for(i=0;i<K_NUM;i=i+1)begin: SHIFT_
            always @ (posedge clk or negedge rst_n) begin
                if (!rst_n) d[(i * TOTAL_WIDTH) +: TOTAL_WIDTH] <= {TOTAL_WIDTH{1'b0}};
                else begin
                    if (clk_gating_en) begin
                        if (over) begin
                            d[(i * TOTAL_WIDTH) +: TOTAL_WIDTH] <= {TOTAL_WIDTH{1'b0}};
                        end else if (shift && i<K_NUM-1) begin
                            d[(i * TOTAL_WIDTH) +: TOTAL_WIDTH] <= d[(i*TOTAL_WIDTH+DATA_WIDTH)+:TOTAL_WIDTH];
                        end else if (indata_valid) begin
                            if (comp[i] == 1'b0 && comp[i+1] == 1'b1) begin
                                d[(i * TOTAL_WIDTH) +: TOTAL_WIDTH] <= indata;
                            end else if (comp[i+1] == 1'b1 && i>0) begin
                                d[(i * TOTAL_WIDTH) +: TOTAL_WIDTH] <= d[( (i-1) * TOTAL_WIDTH ) +: TOTAL_WIDTH];
                            end else begin
                                d[(i * TOTAL_WIDTH) +: TOTAL_WIDTH] <= d[(i * TOTAL_WIDTH) +: TOTAL_WIDTH];
                            end
                        end
                    end
                end
            end
        end
endgenerate

reg [$clog2(cnt_total):0] cnt;
always @ (posedge clk or negedge rst_n) begin
    if (!rst_n) cnt <= {$clog2(cnt_total){1'b0}};
    else begin
        if (clk_gating_en) begin
            if (over)       cnt <= {$clog2(cnt_total){1'b0}};
            else if (shift) cnt <= cnt + {{($clog2(cnt_total)-1){1'b0}},1'b1};
            else            cnt <= cnt;
        end
    end
end

assign over = cnt == cnt_total;

assign outdata = d[TOTAL_WIDTH-1:0];
assign outdata_valid = shift;
assign outdata_over  = over;

reg current, next;
always @ (posedge clk or negedge rst_n) begin
    if (!rst_n)             current <= 1'b0;
    else if (clk_gating_en) current <= next;
end

always @ (*) begin
    case (current)
        1'b0:   if (indata_over)    next = 1'b1;    else next = 1'b0;
        1'b1:   if (over)           next = 1'b0;    else next = 1'b1;
        default: next = 1'b0;
    endcase
end

always @ (*) begin
    case(current)
        1'b0:    shift = 1'b0;
        1'b1:    shift = 1'b1;
        default: shift = 1'b0;
    endcase
end        

endmodule
```

实现首先对输入数据进行并行比较，产生一个类似 [11100] 的序列，0为小于，1为大于。前一位为0且所在位也为0时，数据保持不变；前一位为0且所在位为1时，则为数据插入点；前一位为1且所在位也为1时，接受前置的数据移位。

实现采用了的 Verilog 语言的 for 循环功能来对长寄存器组分块处理，也实现了一个小状态机来在输入/输出状态间进行切换。

需要特别注意的是， for 循环的作用在 Verilog 语言里相当于重复生成了 N 次相同的语句，而不是类似常规软件语言的顺序执行N次。
