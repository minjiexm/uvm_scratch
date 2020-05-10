第一步让我们先编译UVM库。

1. 下载标准uvm库。

   ```shell
   git clone https://github.com/minjiexm/uvm-1.1d.git
   ```

2. 添加uvm库目录和uvm入口文件到files.list中。

   ```text
   +incdir+.                       #Add current path into search path
   +incdir+./uvm-1.1d/src          #Add uvm src
   
   testbench.sv
   ```

3. 在testbench.sv中添加 uvm相关的文件

   ```verilog
   //File Name: testbench.sv
   // top testbench module
   
   `include "uvm.sv"
   `include "uvm_macros.svh"
   
   ....
   
   module testbench();
   
     ....
       
   endmodule : testbench
   ```

   

4. 在questa编译命令中添加 +define+UVM_NO_DPI

   ```shell
   #compile command
   vlog -64 -f files.list -mfcu -suppress 2181 +acc=rmb -writetoplevels questa.tops +define+UVM_NO_DPI
   
   ```

   正常uvm是有一些DPI的函数需要调用的，所如果需要调用DPI函数的话就需要将questa下面的预编译的dpi库函数添加到编译命令中。我们后续也不需要使用到DPI相关的函数，所以此处简化一下，添加+define+UVM_NO_DPI从而不需要添加动态链接库到仿真器。在实际工作中可以参阅所使用的仿真器的用户手册添加uvm编译命令。
