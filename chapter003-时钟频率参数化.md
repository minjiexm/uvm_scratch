# 时钟频率参数化

上面我们已经完成了一个完整时钟产生模块的代码。实际工作中其实一般情况下也是用这样的方式来产生时钟的。下面又有一个问题了，如果要产生其他频率的时钟，比如135MHz的时钟频率，怎么办？ 100MHz我们是很好计算时钟的周期的，10ns。那么135MHz时钟频率就要非一些功夫了。
$$
Period = 1/135MHz = 7.4074ns
$$


```verilog
// timescale
`timescale 1ns/10ps

// top testbench module
module testbench();

  reg clk100M;
  reg clk135M;

  always #5ns      clk100M = ~clk100M;
  always #3.7037ns clk135M = ~clk135M;

  initial begin
    clk100M = 0;
    clk135M = 0;

    #1000000;  //max sim time
    $finish();
  end
    
  initial begin
    $printtimescale($root.testbench);// prints the timescale dut_i module instance
  end 

endmodule : testbench

```



每次新添加一个时钟还需要计算一下半周期，还挺麻烦的，我是一个懒惰的人，能不能让我直接输入频率，自动产生时钟呢？我们可以将产生时钟的代码封装到module里面，然后使用parameter的方式输入频率，内部计算出真实的周期。代码如下



```verilog
// timescale
`timescale 1ns/10ps

module clkgen(output reg clk);
  parameter freq = 100_000; //(unit is KHz)
  parameter period = 1000_000/freq;

  always #(period/2) clk = ~clk;

  initial begin
    clk = 0;
  end
endmodule : clkgen

// top testbench module
module testbench();

  wire clk100M;
  wire clk135M;

  clkgen clkGen100M #(parameter freq = 100_000) (clk100M);
  clkgen clkGen135M #(parameter freq = 135_000) (clk135M);

endmodule : testbench

```

但是这样的写法的问题在于没有办法加入ns的单位，所以必须固定timescale，否则无法保证时钟频率。
因此我们必须把module clkgen放到一个单独的文件，在这个文件的开头声明timescale.

```verilog
//File Name: clkgen.v
// timescale
`timescale 1ns/10ps

module clkgen(output reg clk);
  parameter freq = 100_000; //(unit is KHz)
  parameter period = 1000_000/freq;

  always #(period/2) clk = ~clk;

  initial begin
    clk = 0;
  end
endmodule : clkgen

```

我们将timescale固定为1ns/10ps，也就是说输出的时钟的最大频率为1GHz。
