```verilog
`timescale 1ns / 1ps

module LED(
    input clk,
    output led
    );

parameter T1MS = 26'd50_000_000;//表示一个26位宽的十进制常量
reg [25:0]time_count=26'd0;  // 如果是寄存器 需要指定向量
reg leg_r = 1'b0;
always@(poedge clk)
    if (time_count>=T1MS)begin
        time_count<=26'd0;
        led_r<=~led_r;
    end
    else time_count <= time_count +1'b1;
assign led=led_r;
endmodule 


```

### `timescale 1ns / 1ps

**1ns** —— 时间单位（time unit）。
 你写的 `#1` 延时，就表示 **1 纳秒**。
 `#10` 就是 **10 纳秒**。

**1ps** —— 时间精度（time precision）。
 表示仿真器在计算延时时，最小能分辨的精度是 **1 皮秒**。
 也就是说，如果你写 `#0.49ns`，会四舍五入到 0；
 如果写 `#0.51ns`，会四舍五入举个例子：

```
`timescale 1ns / 1ps

module tb;
    reg clk = 0;
    always #10 clk = ~clk; // 每 10ns 翻转一次 -> 20ns 一个周期
endmodule
```

这里 `#10` 就是 **10ns**，因为时间单位是 `1ns`。

如果改成：

```
`timescale 1us / 1ns
```

那 `#10` 就表示 **10 微秒**。

所以，`timescale` 的作用主要在 **仿真环境**，对综合（真正硬件实现）没有影响。到 1ps。



### <=

- = 阻塞赋值 

- <= 非阻塞赋值
  在时序逻辑中常用 
  告诉综合器  在时钟上升沿

  - time_count清零
  - led_r反转

  注意 二者是并行运行的,如果你写成 `time_count = 26'd0;` 就变成阻塞赋值了，可能在仿真时表现顺序不同，不适合用在时钟驱动的寄存器赋值场景

#### *有没有>=?? 这个和大小比较的小于等于是否冲突 为什么是<=这个符号?*

先看 **比较运算符**：

- `>=` ：大于等于（greater than or equal）
- `<=` ：小于等于（less than or equal）
- `==` ：等于
- `!=` ：不等于

这些出现在 **if 判断里**，例如：

```
if (time_count >= T1MS) begin
    ...
end
```

这里的 `>=` 就是大小比较，意思是 **当计数器值大于等于 T1MS 时**。
 这和数学里的用法一模一样，不会冲突。

------

再看 **赋值运算符**：

- `=` ：阻塞赋值（blocking assignment）
- `<=` ：非阻塞赋值（non-blocking assignment）

这些只能出现在 **过程赋值里**（always、initial 等）。例如：

```
time_count <= 26'd0;
```

这里的 `<=` 是“非阻塞赋值”，和小于等于没有关系。只是恰好长得一样。

------

所以你看到的 `<=` 有两种完全不同的语境：

1. 在 `if (...)` 里：`<=` 表示 **小于等于**。
2. 在 `always` 里的赋值语句里：`<=` 表示 **非阻塞赋值**。

不会冲突，因为编译器可以根据上下文分辨。

----



>d —— decimal（十进制）
>b —— binary（二进制）
>h —— hexadecimal（十六进制）
>o —— octal（八进制）