# 时钟控制

在功能仿真阶段，其实我们很少会去控制时钟，大部分情况下时钟从0时刻就产生好了，我们也不需要做什么改变。但是有的时候，我们总有点特别的小需求，比如我想在仿真的中间修改输入的时钟频率（降频），或者想在仿真的中间将时钟停止一段时间，然后再启动时钟（比如为了调整两个异步时钟之间的相位Phase）。

让我将需求提取出来，罗列如下：

1. 启动时钟。

2. 停止时钟。

3. 动态改变时钟频率。

   

对于1和2来说，实现起来还比较简单，给clkgen模块增加一根使能信号就解决了。

对于3来说，我们也需要将时钟频率作为输入送给module，修改后的代码如下：

```verilog
//File Name: clkgen.v
// timescale
`timescale 1ns/10ps

module clkgen(output reg clk, input wire enable, input real freq);
  //freq unit is KHz
  real period;

  always@(*) begin
    if(freq == 0)
      period = 10; //default 100MHz
    else
      period = 1000_000/freq;
  end


  always #(period/2) if(enable) clk = ~clk;

  initial begin
    clk = 0;
  end

endmodule : clkgen

```



下面的代码是控制时钟频率和使能时钟的例子：

```verilog
//File Name: testbench.sv
// top testbench module

`include "clkgen.v"

module testbench();

  wire clk;
  real freq;
  reg enable;

  clkgen clkGen_inst(clk, enable, freq);
 
  initial begin
    freq = 100_000;  //KHz
    enable = 1;

    //Change frequence
    #1000ns;
    freq = 200_000;  //double the freq
    
    //Disable clock
    #10ns;
    enable = 0;
    #10ns;
    enable = 1;  //enable clock
  end

endmodule : testbench

```


## 再增加一些时钟控制

时钟除了频率之外，其实还有几个特别重要的属性，比如jitter，skew。

Clock Skew: The spatial variation in arrival time of a clock transition on an integrated circuit;

Clock jitter: The temporal vatiation of the clock period at a given point on the chip;

简言之，skew通常是时钟相位上的不确定，而jitter是指时钟频率上的不确定。造成skew和jitter的原因很多。

由于时钟源到达不同寄存器所经历路径的驱动和负载的不同，时钟边沿的位置有所差异，因此就带来了skew。

而由于晶振本身稳定性，电源以及温度变化等原因造成了时钟频率的变化，就是jitter。

此外时钟的起始值也是常见的控制，比如我们需要产生2个反相的时钟。

clock skew由于是延迟导致的，所以产生时钟我们只能控制时钟的频率抖动（jitter）。

占空比是指高电平在一个周期之内所占的时间比率。方波的占空比为50％，占空比为0.5，说明正电平所占时间为0.5个周期。
若信号的周期为T，每周期高电平时间为t1,低电平时间为t2,T=t1+t2,则占空比D=t1/T.



我将时钟需要控制的属性完整的罗列如下：

1. 启动时钟。
2. 停止时钟。
3. 动态改变时钟频率。
4. 控制时钟频率抖动的范围，按照百分比控制。
5. 控制时钟的起始值, 0或者1。
6. 控制时钟的占空比，按照百分比控制，比如60%表示高电平占比为60%。



```verilog
//File Name: clkgen.sv
// timescale
`timescale 1ns/10ps

module clkgen(
    output logic   clk,
    input  string  clk_name,   //this input is used to make print more readable
    input  logic   enable, 
    input  real    freq,
    input  integer jitter,     //percentage, 10 means jitter is 10%
    input  integer duty_cycle, //pos puls width percentage, 50 means 50%
    input  logic   init_value
);

  //freq unit is KHz
  real period;
  real first_half_period;
  real second_half_period;
  integer first_puls_duty;

  initial begin
    #0;  //very important, otherwise clk can not get right init_value in questa.

    clk = init_value;

    forever begin
      if(freq == 0) begin
        $display("[ERROR] clock %s frequence can not be zero!", clk_name); 
        break;
      end

      if(jitter >= 100) begin
        $display("[ERROR] clock %s jitter %0d can not larger than 99!", clk_name, jitter); 
        break;
      end

      period = 1000_000/freq;
      period = period + period*$urandom_range(2*jitter)/100 - period*jitter/100;

      if(period == 0) begin //if half_period == 0 forever loop will happen
        $display("[ERROR] Clock %s period is zero!", clk_name); 
        break; 
      end

      if(duty_cycle == 0 || duty_cycle > 99) begin
          $display("[ERROR] clock %s duty_cycle can not be %0d! duty_cycle should within 0 to 99!", clk_name, duty_cycle); 
        break; 
      end

      if(clk == 1)
        first_puls_duty = duty_cycle;
      else
        first_puls_duty = 100-duty_cycle;

      first_half_period  = period*first_puls_duty/100;
      second_half_period = period-first_half_period;
 
      #(first_half_period)
      if(enable) clk = ~clk;

      #(second_half_period)
      if(enable) clk = ~clk;

    end  //forever
  
  end //initial
 
endmodule : clkgen

```

增加这么多控制之后，时钟模块的输入有7个之多。

其实这里介绍时钟产生模块的目的，并不是要大家聚焦时钟产生模块本身的功能。像我罗列的6个控制时钟的变量，在实际工作中90%大家仅仅是在初始时刻设置一个固定的频率，仅此而已。 比如像jitter的控制估计之后在后仿真带延时信息的时候才会使用到。这里介绍时钟产生模块的原因是我并不想引入一个大家并不熟悉的设计，时钟产生模块相信是所有从事或者想从事IC相关工作的工程师都能理解和掌握的。我们介绍时钟产生模块的方法并不是说一定要让大家应用到工作中去，而是希望利用时钟产生模块产生一种复杂的应用场景，然后能够让大家能够感性的认识到如果不采用UVM验证方法学的话，我们的工作中会多出很多繁琐的，无意义的重复劳动，而且效率低下。然后对比采用UVM验证方法学之后，大家能够对效率的提升有切身的体会。

就像我刚刚说过的一样，目前的代码如果对只产生单个时钟，其实已经不算难用了。那让我们才增加一些难度。下面是产生16个时钟的testbench的代码。频率最低的时钟频率为100MHz，然后依次频率增加10MHz。此处我使用了systemverilog generate语法用for loop的方式例化了16个clkgen模块。

关于generate的具体语法，大家可以去访问 https://www.systemverilog.io/generate 。 里面有比较详细的介绍。



```verilog
//File Name: testbench.sv
// top testbench module

`include "clkgen.sv"

module testbench();
  parameter clk_num = 16;

  logic [clk_num-1:0]  clk;
  logic [clk_num-1:0]  enable;
  logic [clk_num-1:0]  init_value;

  real    freq[clk_num];
  integer jittclk_num[clk_num];
  integer duty_cycle[clk_num];
  string  clk_name[clk_num];

  // The for-loop creates 16 clkgen instance
  genvar i;
  generate
    for (i=0; i < clk_num; i++) begin : clkgen
      clkgen inst(clk[i], clk_name[i], enable[i], freq[i], jitter[i], duty_cycle[i], init_value[i]);
    end
  endgenerate    
  
  //Control simulation length
  initial begin
    #10000;
    $finish();
  end
  
  //inital setting for clock generator
  initial begin
    for (int unsigned i=0; i < clk_num; i++) begin
      enable[i] = 1;
      init_value[i] = 1;

      jitter[i] = 0;  //default jitter set to 0%
      duty_cycle[i] = 50;  //default duty cycle is 50%  
      freq[i] = 100_000 + 10_000*i;  //KHz, 10MHz incr

      clk_name[i].itoa(i);
      clk_name[i] = {"clk[", clk_name[i], "]"};
    end
  end

endmodule : testbench

```

大家可以跑一下仿真，看看结果。

