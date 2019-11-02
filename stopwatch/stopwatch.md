# 基于 **Verilog** 和 **FPGA/CPLD** 的多功能秒表设计 **-** 实验报告 

- 学号： 516072910091
- 姓名： 张万强

## 实验目的

1. 初步掌握利用 Verilog 硬件描述语言进行逻辑功能设计的原理和方法。 
2. 理解和掌握运用大规模可编程逻辑器件进行逻辑设计的原理和方法。 
3. 理解硬件实现方法中的并行性，联系软件实现方法中的并发性。 
4. 理解硬件和软件是相辅相成、并在设计和应用方法上的优势互补的特点。 
5. 本实验学习积累的 Verilog 硬件

## 实验仪器与平台

- 硬件：DE1-SoC 实验板 
- 软件：Altera Quartus II 13.1 

## 实验内容与任务

1.  运用 Verilog 硬件描述语言，基于 DE1-SoC 实验板，设计实现一个具有较多功能的计时秒表。 
2. 要求将 6 个数码管设计为具有“分：秒：毫秒”显示，按键的控制动作有：“计时复位”、“计数/暂停”、“显示暂停/显示继续”等。功能能够满足马拉松或长跑运动员的计时需要。
3. 利用示波器观察按键的抖动，设计按键电路的消抖方法。 
4. 在实验报告中详细报告自己的设计过程、步骤及 Verilog 代码。 

## 实验过程和相关代码

### 实验过程：

1. 首先根据秒表设计要求设计并编写代码
2. 接着通过**PinPlanner** 完成针脚与代码输入输出的连接
3. 最后编译并且上板子执行

### 主模块代码： 

```verilog
module my_watch(clk, key_reset, key_start_pause, key_display_stop,
    hex0, hex1, hex2, hex3, hex4, hex5
    );
    
    input clk, key_reset, key_start_pause, key_display_stop;
    output [6:0] hex0, hex1, hex2, hex3, hex4, hex5;
    
    reg display_work;
    reg counter_work;
    parameter DELAY_TIME     =   10000000; // 200ms
    
    reg [3:0] minute_display_high;
    reg [3:0] minute_display_low;
    reg [3:0] second_display_high;
    reg [3:0] second_display_low;
    reg [3:0] msecond_display_high;
    reg [3:0] msecond_display_low;
    
    
    reg [3:0] minute_counter_high;
    reg [3:0] minute_counter_low;
    reg [3:0] second_counter_high;
    reg [3:0] second_counter_low;
    reg [3:0] msecond_counter_high;
    reg [3:0] msecond_counter_low;
    
    reg [31:0] counter_50M;
    
    reg 	    reset_1_time;
    reg [31:0]  counter_reset;
    reg		    start_1_time;
    reg [31:0]  counter_start;
    reg		    display_1_time;
    reg [31:0]  counter_display;
    

    // sevenseg 模块为 4 位的 BCD 码至 7 段 LED 的译码器，
    // 下面实例化 6 个 LED 数码管的各自译码器。
    sevenseg LED8_minute_display_high(minute_display_high,hex5);
    sevenseg LED8_minute_display_low(minute_display_low,hex4);
    
    sevenseg LED8_second_display_high(second_display_high,hex3);
    sevenseg LED8_second_display_low(second_display_low,hex2);
    
    sevenseg LED8_msecond_display_high(msecond_display_high,hex1);
    sevenseg LED8_msecond_display_low(msecond_display_low,hex0);
    
    // initializer
    initial
    begin
        counter_reset   = 0;
        counter_work    = 0;
        counter_display = 0;
        counter_50M     = 0;
        
        display_work      = 1;
        msecond_display_high = 0;
        msecond_display_low  = 0;
        
        second_display_high = 0;
        second_display_low  = 0;
        
        minute_display_high = 0;
        minute_display_low  = 0;
        
        msecond_counter_high = 0;
        msecond_counter_low  = 0;
        
        second_counter_high = 0;
        second_counter_low  = 0;
        
        minute_counter_high = 0;
        minute_counter_low  = 0;
    end
    
    always@(posedge clk)
    begin
        // key start
        if (!key_reset && !counter_reset) counter_reset            = 1;
        if (!key_start_pause && !counter_start) counter_start      = 1;
        if (!key_display_stop && !counter_display) counter_display = 1;

        if (counter_display) counter_display                       = counter_display+1;
        if (counter_reset) counter_reset                           = counter_reset+1;
        if (counter_start) counter_start                           = counter_start+1;
        
        // debounce actions start
        if (counter_reset == DELAY_TIME)
        begin
            if (key_reset)
            begin
                counter_reset        = 0;
                counter_work         = 0;
                display_work         = 1;
                counter_50M          = 0;
                
                msecond_display_high = 0;
                msecond_display_low  = 0;
                second_display_high  = 0;
                second_display_low   = 0;
                minute_display_high  = 0;
                minute_display_low   = 0;

                msecond_counter_high = 0;
                msecond_counter_low  = 0;
                second_counter_high  = 0;
                second_counter_low   = 0;
                minute_counter_high  = 0;
                minute_counter_low   = 0;
            end
            else counter_reset = 1;
        end

        if (counter_start == DELAY_TIME)
        begin
            if (key_start_pause)
            begin
                counter_start = 0;
                counter_work  = !counter_work;
            end
            else counter_start = 1;
        end

        if (counter_display == DELAY_TIME)
        begin
            if (key_display_stop)
            begin
                counter_display = 0;
                display_work    = !display_work;
            end
            else counter_display = 1;
        end
        
        // debounce actions end
        
        // 计时开始后根据状态调整时间显示
        if (counter_work)
        begin
            counter_50M = counter_50M+1;
            if (counter_50M == 500000)
            begin
                counter_50M         = 0;
                msecond_counter_low = msecond_counter_low+1;
                if (msecond_counter_low == 10)
                begin
                    msecond_counter_low  = 0;
                    msecond_counter_high = msecond_counter_high+1;
                end;
                
                if (msecond_counter_high == 10)
                begin
                    msecond_counter_high = 0;
                    second_counter_low   = second_counter_low+1;
                end

                if (second_counter_low == 10)
                begin
                    second_counter_low  = 0;
                    second_counter_high = second_counter_high+1;
                end

                if (second_counter_high == 6)
                begin
                    second_counter_high = 0;
                    minute_counter_low  = minute_counter_low+1;
                end

                if (minute_counter_low == 10)
                begin
                    msecond_counter_low = 0;
                    minute_counter_high = minute_counter_high+1;
                end

            end
        end
           
        // 显示设置
        if (display_work)
        begin
            minute_display_high  = minute_counter_high;
            minute_display_low   = minute_counter_low;
            second_display_high  = second_counter_high;
            second_display_low   = second_counter_low;
            msecond_display_high = msecond_counter_high;
            msecond_display_low  = msecond_counter_low;
        end
    end
endmodule
            
```

**各按钮状态**： 

- key0：暂停/恢复时间显示
- key1：暂停/开始时间计数器
- key2：重置所有状态。（若当前为计时状态则停止计时。）



**去抖操作：** 

去抖是通过一个**200ms（DELAY_TIME）**的延时计数器实现的，每次按下按钮后（对于任意按钮都是这样），**xx_counter**会进入计时状态，来记录当前状态发生后经过的时间，当某个按钮状态的记录时间达到**DELAY_TIME**之后，会检查一下此时按钮的状态，如果按钮状态已经恢复为**1**，则触发按钮的事件，否则（说明一直在按着按钮）就重置按钮状态计时器开始计时。



### 七段译码模块： 

```verilog
module sevenseg(data,
               ledsegments);
    input [3:0] data;
    output ledsegments;
    reg [6:0] ledsegments;
    
    always@(*)
        case(data)
            0: ledsegments       = 7'b100_0000;
            1: ledsegments       = 7'b111_1001;
            2: ledsegments       = 7'b010_0100;
            3: ledsegments       = 7'b011_0000;
            4: ledsegments       = 7'b001_1001;
            5: ledsegments       = 7'b001_0010;
            6: ledsegments       = 7'b000_0010;
            7: ledsegments       = 7'b111_1000;
            8: ledsegments       = 7'b000_0000;
            9: ledsegments       = 7'b001_0000;
            default: ledsegments = 7'b111_1111;
        endcase
endmodule
```

## 实验总结

### 实验结果

实验代码经过编译综合，载入到开发板后，能正常完成预期的秒表功能。按键消抖效果良好，未出现按键不响应或响应多次的现象。

### 经验教训

最开始做去抖操作的时候是通过按下按钮的稳定时长（时长设置为2ms）来判断的，后来发现这样的结果就是我需要可以延长自己按下按钮的时长来保证防抖操作，觉得有悖常理，于是又查了查资料，并且请教了一下同学，才换成了如今的防抖操作。

### 感受

作为一名软件学院的学生，在本次实验中我亲身体验了让自己设计的逻辑直接在硬件电路上运行的过程，更清晰地理解了硬件的结构、原理及与软件的联系。同时，调试硬件也需要很多耐心，想办法定位问题之所在，必要时和老师、同学沟通能更有效地解决问题。