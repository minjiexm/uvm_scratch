其实我们仔细观察一下testbench.sv。 DUT instance 其实是相对固定的，但是initial block里面的用例我们要不同的修改。uvm_test实际上实现了DUT instance和initial block的分离。testbench里我们只需要例化DUT，在uvm_test我们可以实现对DUT的输入的控制。

做程序员的都知道，我们的第一个程序总是叫hello world。所以我们的第一个uvm_test也不例外，自然也叫hellow_world。大家可以copy下面的源代码，产生一个新的文件hello_world.sv。

```verilog
`ifndef HELLO_WORLD_TEST_SV
`define HELLO_WORLD_TEST_SV

`timescale 1ns/10ps

`include "uvm_macros.svh"
import uvm_pkg::*;

class hello_world extends uvm_test;

  `uvm_component_utils(hello_world)

  function new (string name = "hello_world", uvm_component parent);
    super.new(name, parent);
  endfunction : new

  virtual task run_phase(uvm_phase phase);
    super.run_phase(phase);

    $display("Hello UVM world!");

  endtask : run_phase

endclass : hello_world

`endif //HELLO_WORLD_TEST_SV

```



如果是初学者，估计看到这样代码一下就头晕了。如果没有任何uvm的基础，确实上面的代码看起来有些困难。但是其实上面的这段代码很多部分都是固定不变的。所以让我来变个戏法，来让这个hello_world的uvm_test看起来对初学者友好一些简单一些。

让我们增加一个新的文件叫做clk_gen_uvm_test_define.sv

```verilog
`ifndef CLK_GEN_UVM_TEST_DEFINE_SV
`define CLK_GEN_UVM_TEST_DEFINE_SV

`define clk_gen_uvm_test_define(test_name) \
class test_name extends uvm_test; \
\
  `uvm_component_utils(test_name) \
\
  function new(string name = `"test_name`", uvm_component parent); \
    super.new(name, parent); \
  endfunction : new


`define clk_gen_uvm_test_end(test_name) \
endclass : test_name


`endif //CLK_GEN_UVM_TEST_DEFINE_SV
```



然后重写hello_world.sv：

```verilog
`ifndef HELLO_WORLD_TEST_SV
`define HELLO_WORLD_TEST_SV

`timescale 1ns/10ps

`include "uvm_macros.svh"
import uvm_pkg::*;

`include "clk_gen_uvm_test_define.sv"

`clk_gen_uvm_test_define(hello_world)
  task run_phase(uvm_phase phase);
    $display("Hello UVM world!");
  endtask
`clk_gen_uvm_test_end(hello_world)

`endif //HELLO_WORLD_TEST_SV

```



现在看起来怎么样？是不是简洁多了？你就可以将clk_gen_uvm_test_define当作test的开始，clk_gen_uvm_test_end当做test的结束。至于task run_phase(uvm_phase phase);你暂时就把这个task当成一个task run_phase(); 里面的参数忽略掉，反正我们这里也没用。



下面让我们来运行一下hello_world.sv。

1. 首先将hello_world.sv添加到files.list:

```text
+incdir+.                       #Add current path into search path
+incdir+./uvm-1.1d/src          #Add uvm src

#uvm tests
hellow_world.sv

testbench.sv
```

2. 在testbench.sv的最后添加运行uvm_test的命令：

```verilog
//File Name: testbench.sv
// top testbench module

`include "uvm.sv"
`include "uvm_macros.svh"

....

module testbench();

  ....

  //Run UVM test
  initial begin
    run_test();
  end
 
endmodule : testbench
```



下面是完整的编译和仿真命令：

```shell
#compile command
vlog -64  -timescale "1ps/1ps" -f files.list -mfcu -suppress 2181 +acc=rmb -writetoplevels questa.tops  +define+UVM_NO_DPI

#simulation command
vsim -64 -do "log -r /*; run -all; q" -l questa.log -wlf vsim.wlf -f questa.tops -c +TESTNAME=TC_CHANGE_FREQ +UVM_TESTNAME=hello_world

```

uvm能够根据+UVM_TESTNAME自动找到uvm_test的class并且实例化并执行。



下面是仿真打印：

```shell
# Running test TC_CHANGE_FREQ.
# UVM_INFO ./uvm-1.1d/src/base/uvm_root.svh(370) @ 0.00ns: reporter [NO_DPI_TSTNAME] UVM_NO_DPI defined--getting UVM_TESTNAME directly, without DPI
# UVM_INFO @ 0.00ns: reporter [RNTST] Running test hello_world...
# Hello UVM world!
# UVM_INFO ./uvm-1.1d/src/base/uvm_objection.svh(1267) @ 0.00ns: reporter [TEST_DONE] 'run' phase is ready to proceed to the 'extract' phase
# 
# --- UVM Report Summary ---
# 
# ** Report counts by severity
# UVM_INFO :    4
# UVM_WARNING :    0
# UVM_ERROR :    0
# UVM_FATAL :    0
# ** Report counts by id
# [NO_DPI_TSTNAME]     1
# [RNTST]     1
# [TEST]     1
# [TEST_DONE]     1
# ** Note: $finish    : ./uvm-1.1d/src/base/uvm_root.svh(430)
#    Time: 0 ps  Iteration: 208  Instance: /testbench
# End time: 00:11:28 on Dec 09,2019, Elapsed time: 0:00:02
# Errors: 0, Warnings: 0

```

可以看到hello UVM world！已经成功的被打印出来了。很神奇吧，我们并没有在testbench.sv里面去调用hello_world.sv中任何代码，但是hello_world.sv中的run_phase就自动被执行了。而且uvm会自动接管整个仿真过程，uvm会判断仿真是否可以结束，并不需要我们再手动调用$finish()。

让我们将testbench.sv中的tc_change_freq和tc_enable写成uvm_test的形式。

让我们产生新的文件叫做clk_gen_tests.sv：

```verilog
`ifndef CLK_GEN_TESTS_SV
`define CLK_GEN_TESTS_SV

`timescale 1ns/10ps

`include "uvm_macros.svh"
import uvm_pkg::*;

`include "clk_gen_uvm_test_define.sv"


//Test case 1 : dynamic enable/disable clock
`clk_gen_uvm_test_define(tc_enable)
  task run_phase(uvm_phase phase);
      
    phase.raise_objection(this);  //block terminating simulation

    #100;
    testbench.enable[0] = 0;

    #1000;
    testbench.enable[0] = 1;

    #1000;
    testbench.enable[0] = 0;

    #1000;
    testbench.enable[0] = 1;
      
    phase.drop_objection(this);  //now simulation can be terminated

  endtask
`clk_gen_uvm_test_end(tc_enable)

 
    
  //Test case 2 : dynamic change frequency
`clk_gen_uvm_test_define(tc_change_freq)
  task run_phase(uvm_phase phase);

    phase.raise_objection(this);  //block terminating simulation

    #1000;
    testbench.freq[0] = testbench.freq[0]/2;

    #1000;
    testbench.freq[0] = testbench.freq[0]*2;

    phase.drop_objection(this);  //now simulation can be terminated

  endtask
`clk_gen_uvm_test_end(tc_change_freq)


`endif //CLK_GEN_TESTS_SV

```

 大家先不用深究phase.raise_objection(this);  和phase.drop_objection(this); 这两句话简单来说就是用来控制仿真结束时间的，如果不加仿真会在0ns就结束，我们会在后续的章节再详细介绍。



添加到files.list中：

```text
+incdir+.                       #Add current path into search path
+incdir+./uvm-1.1d/src          #Add uvm src

hellow_world.sv
clk_gen_tests.sv
testbench.sv
```



在testbench.sv我们就可以删除tc_enable和tc_change_freq以及控制仿真时间和结束的代码，修改后代码如下：

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

  //Run UVM test
  initial begin
    run_test();
  end

endmodule : testbench
```



这样我们就可以用+UVM_TESTNAME来控制运行哪个uvm_test。

大家可以试试将+UVM_TESTNAME=hello_world改成+UVM_TESTNAME=tc_enable以及+UVM_TESTNAME=tc_change_freq。看看是不是看到了熟悉的打印结果？

相信大家已经能够感觉到了，uvm_test将test case从testbench.sv彻底剥离出来。testbench.sv也变的简单起来。复用性和可读性也都大大提高了。而且这仅仅是uvm给大家带来第一个好处。后面还有更多惊喜等着大家呢。
