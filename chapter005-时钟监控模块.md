# 时钟监控（clock monitor）

之前我们产生时钟之后，都是直接测量波形来判断时钟产生的是否正确。但是真实的工作中，我们不可能精力和时间在每次simulation之后都去检查波形来判断功能是否正确与否。观察波形应该是一种调试问题的手段，而不是确定正确与否的手段。

能否有一种方法能够将波形转换成打印信息，让我们可以直接判断功能正确与否呢？答案当然是肯定的。下面我们就来实现一个自动检查时钟频率的检查器，功能点如下：

1. 自动打印时钟频率和周期信息。这样我们就不需要再去打开波形来测量和计算了。
2. 当时钟频率不满足预期，自动打印错误报警。
3. 当频率发生变化的时候，自动打印最新的频率信息。

思路其实也很简单，自动计算两个时钟正沿的时间间隔，获取时钟的周期信息，然后换算成时钟频率信息就可以了。
请大家先自己写出自己的代码，如果需要参考，可以点击“时钟监控”。

<details>
<summary>时钟监控</summary>



```verilog
//File Name: clk_freq_checker.sv
// Clock frequency checker

`timescale 1ns/10ps

module clk_freq_checker(input logic clk, input string clk_name, input real expect_freq, input integer jitter);

  real pos_edge_t;  //first posedge time
  real last_pos_edge_t;  //second posedge time

  real frequency;
  real last_cycle_freq = 0;
  real period;

  initial begin
    #1; //wait a little bit then start to check freq.

    forever begin
      @ (posedge clk) pos_edge_t = $realtime;

      period = pos_edge_t - last_pos_edge_t;

      frequency = 1000_000 / period;

      if(freq_change(frequency, last_cycle_freq, jitter)) begin //frequency change
        $display("clock %s Frequency = %7.2fKHz", clk_name, frequency);
        $display("clock %s period = %7.2fns", clk_name, period);   //step is ns
        void'( check_freq(frequency, expect_freq, jitter) );
        last_cycle_freq = frequency;
      end

      last_pos_edge_t = pos_edge_t;
    end
  end
    
  //This function is used to check whether frequency changed
  function logic freq_change_detect(input real cur_freq, real last_freq, integer jitter);
    real delta_freq;
    integer real_jitter;

    if(cur_freq == last_freq)
      return 0;  //freq not changed

    if(cur_freq > last_freq)
      delta_freq = cur_freq - last_freq;
    else
      delta_freq = last_freq - cur_freq;

    real_jitter = 100*delta_freq/cur_freq;

    if(real_jitter > jitter) begin
      return 1;  //freq changed and out of range
    end

    return 0;
  endfunction : freq_change

    
  function logic check_freq(input real cur_freq, real expect_freq, integer jitter);
    real delta_freq;
    integer real_jitter;

    if(cur_freq == expect_freq)
      return 1; //freq is ok

    if(cur_freq > expect_freq)
      delta_freq = cur_freq - expect_freq;
    else
      delta_freq = expect_freq - cur_freq;

    real_jitter = 100*delta_freq/cur_freq;

    if(real_jitter > jitter) begin
      $display("[ERROR] clock %s Expect Frequency = %7.2fKHz, Real Frequency = %7.2fKHz", clk_name, expect_freq, cur_freq);
      $display("[ERROR] clock %s Expect jiter = %0d%%, Real jitter = %0d%%", clk_name, jitter, real_jitter);
      return 0;
    end

    return 1;
  endfunction : check_freq
    
endmodule : clk_freq_checker
```



```verilog
//File Name: testbench.sv
// top testbench module

`include "clkgen.sv"
`include "clk_freq_checker"

module testbench();
  parameter clk_num = 16;

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
    #10000ns;
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

endmodule : testbench

```

</details>



下面是仿真打印信息，可以看到通过clk_freq_checker我们将频率信息直接打印出来，非常的直观。

```shell
# Starting sim at 0
# clock clk_check[15] Frequency = 250000.00KHz
# clock clk_check[15] period =    4.00ns
# clock clk_check[14] Frequency = 240384.62KHz
# clock clk_check[14] period =    4.16ns
# clock clk_check[13] Frequency = 230414.75KHz
# clock clk_check[13] period =    4.34ns
# clock clk_check[12] Frequency = 220264.32KHz
# clock clk_check[12] period =    4.54ns
# clock clk_check[11] Frequency = 210084.03KHz
# clock clk_check[11] period =    4.76ns
# clock clk_check[10] Frequency = 200000.00KHz
# clock clk_check[10] period =    5.00ns
# clock clk_check[9] Frequency = 190114.07KHz
# clock clk_check[9] period =    5.26ns
# clock clk_check[8] Frequency = 179856.12KHz
# clock clk_check[8] period =    5.56ns
# clock clk_check[7] Frequency = 170068.03KHz
# clock clk_check[7] period =    5.88ns
# clock clk_check[6] Frequency = 159744.41KHz
# clock clk_check[6] period =    6.26ns
# clock clk_check[5] Frequency = 150150.15KHz
# clock clk_check[5] period =    6.66ns
# clock clk_check[4] Frequency = 140056.02KHz
# clock clk_check[4] period =    7.14ns
# clock clk_check[3] Frequency = 129870.13KHz
# clock clk_check[3] period =    7.70ns
# clock clk_check[2] Frequency = 119904.08KHz
# clock clk_check[2] period =    8.34ns
# clock clk_check[1] Frequency = 109890.11KHz
# clock clk_check[1] period =    9.10ns
# clock clk_check[0] Frequency = 100000.00KHz
# clock clk_check[0] period =   10.00ns
# End of sim at 5050000

```
