---
title: SystemVerilog 下多路复用器的简单优化
date: 2021-05-02 20:58:38
tags: [Verilog]
categories: "硬件"
---

本文基于知乎文章[《多路选择器的一个简单优化》](https://zhuanlan.zhihu.com/p/368348208)，聊一聊如何使用 SystemVerilog 下的新关键词 `unique` 帮助化简代码，优化逻辑资源消耗。

## 考虑如下问题

>输入二维数组 [7:0][63:0] data_i 和 [7:0] valid_i
>输出一维数组 [63:0] data_o
>输出为 valid_i 为 1 所在 bit 对应 index 上的 data_i
>输入的 valid_i 为 one-hot 编码

很明显问题需要我们实现一个 8-in-1 多路选择器，也称为多路复用器 (MUX) 。

<!-- more -->

## 实现方案

在不考虑优化的情况下，可以写出如下代码 1:

```verilog
always_comb begin
    data_o = '0;
    for (int i=0; i<8; i++) begin
        if (valid_i[i] == 1'b1) begin
            data_o = data_i[i];
        end
    end
end
```

利用 valid 信号独热的性质，可以实现为代码 2:

```verilog
always_comb begin
    data_o = '0;
    for (int i=0; i<8; i++) begin
        if (valid_i[i] == 1'b1) begin
            data_o |= data_i[i];
        end
    end
end
```

代码 1 保证了多 bit valid 下的选择安全，它总会选择优先级最高（index最大）的有效数据；代码 2 的选出逻辑则会在不独热的情况下失效。

但代码 2 得益于 valid 预知的独热性质，对逻辑有化简作用。在 Vivado 2020.1 上，代码 1 综合完共消耗 3 个 LUT 6, 1 个 LUT 3, 1 个 LUT 4，而代码 2 仅需要 3 个 LUT6 。（1 bit data_o）

## 引入 SystemVerilog 特性

在 Verilog 中，由于语言本身的类型系统较弱，类似的手工优化魔法还有很多。

但 SystemVerilog 为我们提供了 `unique` 和 `priority` 两个新关键词，分别用于标识独热并行和有优先逻辑，让综合工具自动完成更深层次的逻辑重组和优化。

利用 `unique` 关键词，我们可以把之前的代码改写为代码 3:

```verilog
always_comb begin
    data_o = '0;
    for (int i=0; i<8; i++) begin
        unique if (valid_i[i] == 1'b1) begin
            data_o = data_i[i];
        end else begin

        end
    end
end
```

在 Vivado 上也可以使用 `unique0` 关键词替代 `unique` ，`unique0` 是 `unique` 的语法糖，它允许后缀的 if 声明内的块全部无效，也即可以省略空的 else 块。

使用 `unique` 的代码综合后消耗 3 个 LUT6 和 1 个 LUT3 ，比代码 1 好，但还是多于代码 2 ，这是为什么呢？

让我们简短回顾一下 Verilog 的类型系统，因为实际硬件电路中存在四种状态：0、1、X、Z，所以 Verilog 中的线网类型默认都是四态逻辑。

代码 2 显式地将输入取或，暗示输入不可以为 Z 态或者 X 态，综合工具可以直接利用第一层的或出结果进行第二层选择。但当输入为 Z 态时，0 与 Z 态取或会导致输出变为 X 态。

代码 3 没有对输入做运算处理，所以当输入为 Z 态时也可以正确传播，但综合工具就不能再利用输入数据来重组逻辑了。

**TL;DR, 代码 2 不允许 Z 态传播，代码 3 允许 Z 态传播，所以需要更多资源。**

所以其实还是各种资源的 trade-off.
