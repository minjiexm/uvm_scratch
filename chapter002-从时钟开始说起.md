# 从时钟开始说起

​        验证方法学的目的就是为了应付日益复杂的数字集成电路的验证需求。数字集成电路的核心其实就是同步时序，在时钟的驱动下完成复杂的功能。所以我打算从时钟开始我们的旅程。

​         让我抛出我的一个问题吧。请产生一个频率为100MHz的时钟信号。我相信很多朋友就开始不屑一顾起来，这又什么难的？确实是很简单的，下面让我贴上我的答案：

```verilog
//testbench.v
module testbench();

  reg clk100M;
 
  always #5 clk100M = ~clk100M;

  //Control simulation length
  initial begin
    #10000;
    $finish();
  end

endmodule : testbench
```

​         怎么样，跟你们的答案一样么？请问我的答案是正确答案么？大家可以把我的程序保存成testbench.v, 然后用modelsim跑一下仿真看看波形。

首先我们在testbench.v的所在目录下打开一个files.list文件，内容如下：

```text
+incdir+.   #Add current directory into serach path

testbench.sv
```

下面是编译和仿真命令：

```shell
#compile command
vlog -64 -f files.list -mfcu -suppress 2181 +acc=rmb -writetoplevels questa.tops

#simulation command
vsim -64 -do "log -r /*; run -all; q" -l questa.log -wlf vsim.wlf -f questa.tops -c

#open waveform
vsim -64 -view vsim.wlf
```



打开波形文件之后，相信有的朋友一下就能发现我代码有问题，我没有给时钟信号赋初始值，所以clk100M的初始状态就是X（不定态），不定态取反还是不定态，所以我的代码时钟信号会一直是不定态（X）。所以我们要给时钟一个起始值，下面是修正后的代码：

```verilog
//testbench.v
module testbench();

  reg clk100M;
 
  always #5 clk100M = ~clk100M;

  //Control simulation length
  initial begin
    #10000;
    $finish();
  end

  initial begin
    clk100M = 0;  //assign inital value for clock
  end
    
endmodule : testbench
```

​        再次进行仿真，我们发现这下时钟信号正确的产生了。大家可以测量一下时钟的频率，正好是100MHz。那是否这段代码就万无一失呢？答案是否定的。让我们用下面的命令再来运行一下仿真：

```shell
vlog -64  -timescale "1ps/1ps" testbench.v -mfcu -suppress 2181 +acc=rmb -writetoplevels questa.tops

vsim -64 -do "log -r /*; run -all; q" -l questa.log -wlf vsim.wlf -f questa.tops -c

vsim -64 -view vsim.wlf
```

​        现在再检查时钟频率，请问还是100MHz么？关键的问题是在于单位， always #5 clk100M = ~clk100M;的#5并没有指明是5ns，单位是随着timescale发生变化的。#5的单位是timescale的timeunit。一旦timeunit发生变化，那时钟的频率也就发生变化了。所以为了确保100MHz的频率，我们必须固定时间单位。一种写法是直接写成 always #5ns clk100M = ~clk100M; 但是我们发现modelsim编译会报错，因为我们文件后缀是.v, verilog不支持绝对时间延迟，所以我们需要将文件后缀改为.sv, Systemverilog支持绝对时间单位。

```verilog
//testbench.sv
module testbench();

  reg clk100M;
 
  always #5ns clk100M = ~clk100M;

  initial begin
    clk100M = 0; 
   
    #1000000;  //max sim time
    $finish();
  end
    
endmodule : testbench
```

更改一下仿真命令：

```shell
vlog -64  -timescale "1ps/1ps" testbench.sv -mfcu -suppress 2181 +acc=rmb -writetoplevels questa.tops

vsim -64 -do "log -r /*; run -all; q" -l questa.log -wlf vsim.wlf -f questa.tops -c

vsim -64 -view vsim.wlf
```

我们可以发现时钟频率又回到了100MHz了。
