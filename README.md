# UVM Training from Scratch

This training course is for a beginner who wants to get familiar with UVM.

We will use Mentor Modelsim/Questa as simulator.



## Stage 1: Preparing the LAB environment

Print "Hello world from Systemverilog!"

Create a systemverilog module called testbench.sv

```
module testbench();

  initial begin
    $display("Hello world from systemverilog!");
  end

endmodule : testbench
```

Compile/run then check the simulation log.

You should see "Hellow world from systemverilog!" in the simulation log.



## Stage 2: Compile UVM LIB

Download uvm-1.1d lib from github

git clone https://github.com/hbq/uvm-1.1d.git

Add UVM lib to compile list.



## Stage 3: Hello world from UVM!

Create a uvm test called myFirstTest.sv.

```
class myFirstTest extends uvm_test;

  `uvm_component_utils(myFirstTest)

   function new(string name, uvm_component parent);
     super.new(name, parent);
   endfunction : new
   
   task main_phase(uvm_phase phase);
     $display("Hellow world from UVM!");
   endtask : main_phase
   
endclass : myFirstTest
```

Add below codes into testbench.sv

```
module testbench();

  initial begin
    $display("Hello world from systemverilog!");
    run_test(); //Add this will run UVM simulation.
  end

endmodule : testbench
```

Also add a simulation arg to vsim command:

+UVM_TESTNAME=myFirstTest

Compile/run then check the simulation log.

You should see "Hellow world from UVM!" in the simulation log.