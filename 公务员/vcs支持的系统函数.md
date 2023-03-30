### 一、vcs支持的系统函数

#### 1、$lsi_dumpports

在verilog建模方面（在使用ATPG自动生成测试模型时）可以使用$lsi_dumpports函数

本节旨在提供将\$lsi_dumpports与自动测试模式生成(ATPG)工具一起使用时的指导。有时，ATPG工具严格遵循端口方向，不允许从设备内部驱动单向端口。如果在编写测试夹具时不小心，\$lsi_dumpports的结果会给ATPG工具带来问题。

###### 1.1处理未分配的线网类型

```
module test(A);
input A; 
wire A; 
DUT DUT_1 (A); 
// assign A = 1'bz; 
initial 
$lsi_dumpports(DUT_1,"dump.out"); 
endmodule

module DUT(A); 
input A; 
wire A; 
child child_1(A); 
endmodule

module child(A); 
input A; 
wire Z,A,B; 
and (Z,A,B); 
endmodule
```

 可以检测出未被赋值的线网类型。上述程序将会返回一个值F。它在DUT_1的端口A的任何一侧都找不到驱动器，因此给出了一个F，三态（输入和输出未连接）的代码。当导线从外部而不是内部驱动时，$lsi_dumpports返回一个z代码。

正确的使用方法 

###### 1.2 时间0的代码值

另一个问题可能发生在时间0，即按照您的预期将值分配给端口之前。因此，\$lsi_dumpports在未完成所有用户指定的任务时对驱动程序进行评估。为了纠正这种情况，您需要提前模拟时间，以便完成任务。这可以通过在\$lsi_dumpports之前添加#1来实现，如下所示

```
initial 
begin 
#1 $lsi_dumpports(instance,"dump.out"); 
end
```

###### 1.3

```
module test; 
initial 
begin 
force top.u1.a = 1'b0; 
$lsi_dumpports(top.u1,"dump.out"); 
end 
endmodule

module top; 
middle u1 (a); 
endmodule

module middle(a); 
input a; 
wire b; 
buf(b,a); 
endmodule
```

存在两个问题，问题一：在顶层test没有实例化top,lsi_dumpports调用实例的名称。问题二：跨模块调用，force的使用，是test指定了top中u1的a信号。用户希望u1的这个端口a是一个输入，但当$lsi_dumpports计算驱动程序的端口时，它发现实例u1的端口a是从内部驱动的，因此返回一个L代码。

```
要纠正这两个问题，需要实例化top inside测试，并在测试中驱动信号a。这是通过以下方式完成的
module test; 
wire a; 
initial 
begin 
force a = 1'b0; 
$lsi_dumpports(test.u0.u1,"dump.out2"); 
end 
top u0 (a); 
endmodule
module top(a); 
input a; 
middle u1 (a); 
endmodule
module middle(a);
input a; 
wire b; 
buf(b,a); 
endmodule
```

