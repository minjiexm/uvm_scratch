让我们回过头看看到目前为止，我们写过的代码。

首先我们写的第一个文件是testbench.v。然后将testbench.v里面产生时钟的那一行代码进行了扩充，变成了一个独立的文件clkgen.sv。之后为了能够自动的检测clkgen的输出时钟的频率，我们写一个clk_freq_checker.sv。在testbench.sv中我们还需要控制仿真的总时间（退出机制），时钟模块的初始化。

在控制时钟的那一节，我罗列了时钟控制模块的几个特性（feature）：

1. 启动时钟。
2. 停止时钟。
3. 动态改变时钟频率。

如果我们将时钟控制模块作为DUT来看待的话，显然我们需要分别测试上面的3个特性。首先我们测试时钟的启动和停止功能，因此我们需要动态的将enable设置为0或者1。下面的testbench代码中，我们将clk_num设置为1，仅仅实例化一个clock generator module， 并增加了一个inital的模块，用来设置enable为0，等待1000ns之后又设置为1。


<details>
<summary>Testbench</summary>

```verilog
//File Name: testbench.sv
// top testbench module

`include "clkgen.sv"
`include "clk_freq_checker.sv"

`timescale 1ns/10ps

module testbench();
  parameter clk_num = 1;

  logic [clk_num-1:0]  clk;
  logic [clk_num-1:0]  enable;
  logic [clk_num-1:0]  init_value;

  real    freq[clk_num];
  integer jitter[clk_num];
  integer duty_cycle[clk_num];
  string  clk_name[clk_num];
  string  clk_check_name[clk_num];

  //automatic instance 16 clock generator
  genvar i;
  generate
    // The for-loop creates 16 clkgen instance
    for (i=0; i < clk_num; i++) begin : clkGen
      clkgen inst(clk[i], clk_name[i], enable[i], freq[i], jitter[i], duty_cycle[i], init_value[i]);
    end

    // The for-loop creates 16 clock frequence checker instance
    for (i=0; i < clk_num; i++) begin : clkCheck
      clk_freq_checker inst(clk[i], clk_check_name[i], freq[i], jitter[i]);
    end
  endgenerate

  //Control simulation length
  initial begin
   //shall print %t with scaled in ns (-9), with 2 precision digits, and would print the "ns" string after time
   $timeformat(-9, 2, "ns", 20);

    #10000;
    $finish();
  end

  //setting for clock generator
  initial begin
    for (int unsigned i=0; i < clk_num; i++) begin
      enable[i] = 1;
      init_value[i] = 1;

      jitter[i] = 0;  //default jitter set to 0%
      duty_cycle[i] = 50;  //default duty cycle is 50%  
      freq[i] = 100_000 + 10_000*i;  //KHz, 10MHz incr

      clk_name[i].itoa(i);
      clk_name[i] = {"clk[", clk_name[i], "]"};

      clk_check_name[i].itoa(i);
      clk_check_name[i] = {"clk_check[", clk_check_name[i], "]"};
    end
  end

  //Test case 1 : dynamic enable/disable clock
  initial begin
    #100;
    enable[0] = 0;
    
    #1000;
    enable[0] = 1;

    #1000;
    enable[0] = 0;
    
    #1000;
    enable[0] = 1;
  end
    
endmodule : testbench

```
</details>



下面是仿真结果的打印：

```shell
# clock clk_check[0] Frequency = 100000.00KHz
# clock clk_check[0] period =   10.00ns
# clock clk_check[0] Frequency =  990.10KHz
# clock clk_check[0] period = 1010.00ns
# [ERROR] clock clk_check[0] Expect Frequency = 100000.00KHz, Real Frequency =  990.10KHz
# [ERROR] clock clk_check[0] Expect jiter = 0%, Real jitter = 10000%
# clock clk_check[0] Frequency = 100000.00KHz
# clock clk_check[0] period =   10.00ns
# clock clk_check[0] Frequency =  990.10KHz
# clock clk_check[0] period = 1010.00ns
# [ERROR] clock clk_check[0] Expect Frequency = 100000.00KHz, Real Frequency =  990.10KHz
# [ERROR] clock clk_check[0] Expect jiter = 0%, Real jitter = 10000%
# clock clk_check[0] Frequency = 100000.00KHz
# clock clk_check[0] period =   10.00ns

```

从ERROR的打印可以看出，我们成功的让时钟停止了1000ns，所以时钟周期变成了1010ns。所以我们并不需要观察波形就可以判断enable的功能工作正常。

下面我们还需要测试一下动态改变时钟频率的功能。我们将测试enable的inital模块先注释掉，然后增加一段动态修改频率的inital代码：


<details>
<summary>Testbench</summary>

```verilog
//File Name: testbench.sv
// top testbench module

`include "clkgen.sv"
`include "clk_freq_checker.sv"

`timescale 1ns/10ps

module testbench();
  parameter clk_num = 1;

  logic [clk_num-1:0]  clk;
  logic [clk_num-1:0]  enable;
  logic [clk_num-1:0]  init_value;

  real    freq[clk_num];
  integer jitter[clk_num];
  integer duty_cycle[clk_num];
  string  clk_name[clk_num];
  string  clk_check_name[clk_num];

  //automatic instance 16 clock generator
  genvar i;
  generate
    // The for-loop creates 16 clkgen instance
    for (i=0; i < clk_num; i++) begin : clkGen
      clkgen inst(clk[i], clk_name[i], enable[i], freq[i], jitter[i], duty_cycle[i], init_value[i]);
    end

    // The for-loop creates 16 clock frequence checker instance
    for (i=0; i < clk_num; i++) begin : clkCheck
      clk_freq_checker inst(clk[i], clk_check_name[i], freq[i], jitter[i]);
    end
  endgenerate

  //Control simulation length
  initial begin
   //shall print %t with scaled in ns (-9), with 2 precision digits, and would print the "ns" string after time
   $timeformat(-9, 2, "ns", 20);

    #10000;
    $finish();
  end

  //setting for clock generator
  initial begin
    for (int unsigned i=0; i < clk_num; i++) begin
      enable[i] = 1;
      init_value[i] = 1;

      jitter[i] = 0;  //default jitter set to 0%
      duty_cycle[i] = 50;  //default duty cycle is 50%  
      freq[i] = 100_000 + 10_000*i;  //KHz, 10MHz incr

      clk_name[i].itoa(i);
      clk_name[i] = {"clk[", clk_name[i], "]"};

      clk_check_name[i].itoa(i);
      clk_check_name[i] = {"clk_check[", clk_check_name[i], "]"};
    end
  end

  //Test case 1 : dynamic enable/disable clock
  /*
  initial begin
    #100;
    enable[0] = 0;
    
    #1000;
    enable[0] = 1;

    #1000;
    enable[0] = 0;
    
    #1000;
    enable[0] = 1;
  end
  */
    
  //Test case 2 : dynamic change frequency
  initial begin
    #1000;
    freq[0] = freq[0]/2;
    
    #1000;
    freq[0] = freq[0]*2;
  end

endmodule : testbench

```

</details>



下面是仿真的打印信息：

```shell
# clock clk_check[0] Frequency = 100000.00KHz
# clock clk_check[0] period =   10.00ns
# clock clk_check[0] Frequency = 50000.00KHz
# clock clk_check[0] period =   20.00ns
# clock clk_check[0] Frequency = 100000.00KHz
# clock clk_check[0] period =   10.00ns
```

大家可以从打印信息中直接看到频率的变化，和我们的设定也是一致的。

时钟产生模块其实还有2个参数，jitter和占空比。这里我们就留给读者自己去测试了。
