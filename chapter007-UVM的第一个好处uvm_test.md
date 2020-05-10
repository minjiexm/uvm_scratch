也许大家都有点着急了，怎么啰啰嗦嗦讲了这么多篇幅，还没进入到UVM的介绍呀？别急别急，心急吃不了热豆腐。下面我们就开始介绍并且使用点uvm的东西，给大家点开胃菜。

在之前给时钟模块添加测试用例的过程中，不知道大家有没有发现什么问题？那就是切换不同的测试用例。我们可以发现每次如果要运行第二个更改时钟频率的用例，还需要把第一个enable的用例先注释掉。反之，要运行第一个用例就需要手动注释掉第二个用力，都需要先修改testbench.sv。在我们这个小环境当然没什么问题，编译快的很。可是大家真正进入到公司里，里面都是实际的项目，项目越大，代码越多，编译所消耗的时间就越久。如果每次选择一个用例都要去修改代码，这肯定是不能接受的。另外我们还需要进行测试用例的回归（regression），我们需要环境自动的将所有的用例并行提交到server farm。

为了解决这个问题，我们可以将用例写成独立的task，然后再用command line argument来控制运行哪个用例。大家可以去访问 https://www.chipverify.com/systemverilog/systemverilog-command-line-input 获得详细的command line argument的使用方法。



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
  task tc_enable();
    #100;
    enable[0] = 0;
    
    #1000;
    enable[0] = 1;

    #1000;
    enable[0] = 0;
    
    #1000;
    enable[0] = 1;
  endtask
 
    
  //Test case 2 : dynamic change frequency
  task tc_change_freq();
    #1000;
    freq[0] = freq[0]/2;
    
    #1000;
    freq[0] = freq[0]*2;
  endtask
    
  initial begin
    if ($value$plusargs("TESTNAME=%s", testname)) begin
      $display("Running test %0s.", testname);

      if (testname == "TC_ENABLE")
        tc_enable();
      else if (testname == "TC_CHANGE_FREQ")
        tc_change_freq(); 
    end
  end

endmodule : testbench

```



这样我们就可以用+TESTNAME来控制运行哪个用例：

```shell
vlog -64  -timescale "1ps/1ps" -f files.list -mfcu -suppress 2181 +acc=rmb -writetoplevels questa.tops

vsim -64 -do "log -r /*; run -all; q" -l questa.log -wlf vsim.wlf -f questa.tops -c +TESTNAME=TC_ENABLE

vsim -64 -do "log -r /*; run -all; q" -l questa.log -wlf vsim.wlf -f questa.tops -c +TESTNAME=TC_CHANGE_FREQ

```



这种写法现在看上去应付2个测试用例的情况，感觉还不错。但是如果测试用例有上千条怎么办？我们不可能将所有的测试用例都放到testbench.sv这一个文件中吧？而且这种写法看上去就是一种不好的代码风格，重用性很差。假设另外一个项目要复用时钟的测试用例，还需要先修改自己的testbench.sv,加上 else if (testname == "TC_CHANGE_FREQ") 这样的代码。

下面让我们引入UVM的uvm_test来帮我们解决这个问题。大家可以亲身感受一下uvm_test带来的好处。
