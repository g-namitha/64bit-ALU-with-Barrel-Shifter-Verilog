#  Pipelined 64-bit ALU with Barrel Shifter (Verilog)

##  Overview

This project implements a **high-performance 64-bit pipelined ALU** integrated with a **logarithmic barrel shifter**. The design uses a **multi-stage pipeline** to improve throughput and enable high-frequency operation.

---

##  Key Features

*  64-bit datapath
*  3-stage pipelined architecture
*  Integrated barrel shifter (O(log N) delay)
*  Supports arithmetic, logic, shift, and comparison
*  Valid signal propagation (valid_in → valid_out)

---

##  Supported Operations

| alu_sel | Operation |
| ------- | --------- |
| 0000    | ADD       |
| 0001    | SUB       |
| 0010    | AND       |
| 0011    | OR        |
| 0100    | XOR       |
| 0101    | SHL       |
| 0110    | SHR       |
| 0111    | ASR       |
| 1000    | SLT       |

---

##  Pipeline Stages

###  Stage 1: Input Register Stage

* Latches inputs (`A`, `B`, `alu_sel`)
* Synchronizes with clock
* Stores `valid_in`

---

###  Stage 2: Execution Stage

* Performs:

  * Addition/Subtraction
  * Logic operations
  * Barrel shifting
  * Comparison
* Stores intermediate results

---

###  Stage 3: Output Stage

* Selects final output
* Generates flags:

  * Zero
  * Negative
  * Carry
  * Overflow
* Outputs `valid_out`

---

##  Barrel Shifter

* 6-stage logarithmic structure:

  * 32, 16, 8, 4, 2, 1 shifts
* Supports:

  * Logical left shift
  * Logical right shift
  * Arithmetic right shift

---

##  Pipeline Advantage

* Increased throughput
* Reduced critical path delay
* Suitable for high-frequency processors

---

##  Simulation

Tested using xilinx Vivado

##  Applications

* CPU datapath design
* DSP processors
* FPGA-based accelerators

 ## Verilog Design

```verilog
// ============================================================
// 64-bit Barrel Shifter (Bidirectional with Arithmetic Support)
// ============================================================

module barrel_shifter(
    input  [63:0] din,
    input  [5:0]  shift,
    input         dir,
    input         arth_shift,
    output [63:0] dout
);

wire [63:0] s1, s2, s3, s4, s5, s6;
wire [63:0] d1, d2, d3, d4, d5, d6;

// Arithmetic shift enable (only for right shift)
wire arith = arth_shift & dir;

// Sign extension (replicates MSB)
wire [63:0] sign_ext = {64{din[63]}};

// -------- Stage 1: Shift by 32 --------
wire [63:0] l1 = {din[31:0], 32'b0};
wire [63:0] r1 = {(arith ? sign_ext[31:0] : 32'b0), din[63:32]};
mux m1(d1, dir, l1, r1);
mux m2(s1, shift[5], din, d1);

// -------- Stage 2: Shift by 16 --------
wire [63:0] l2 = {s1[47:0], 16'b0};
wire [63:0] r2 = {(arith ? sign_ext[15:0] : 16'b0), s1[63:16]};
mux m3(d2, dir, l2, r2);
mux m4(s2, shift[4], s1, d2);

// -------- Stage 3: Shift by 8 --------
wire [63:0] l3 = {s2[55:0], 8'b0};
wire [63:0] r3 = {(arith ? sign_ext[7:0] : 8'b0), s2[63:8]};
mux m5(d3, dir, l3, r3);
mux m6(s3, shift[3], s2, d3);

// -------- Stage 4: Shift by 4 --------
wire [63:0] l4 = {s3[59:0], 4'b0};
wire [63:0] r4 = {(arith ? sign_ext[3:0] : 4'b0), s3[63:4]};
mux m7(d4, dir, l4, r4);
mux m8(s4, shift[2], s3, d4);

// -------- Stage 5: Shift by 2 --------
wire [63:0] l5 = {s4[61:0], 2'b0};
wire [63:0] r5 = {(arith ? sign_ext[1:0] : 2'b0), s4[63:2]};
mux m9(d5, dir, l5, r5);
mux m10(s5, shift[1], s4, d5);

// -------- Stage 6: Shift by 1 --------
wire [63:0] l6 = {s5[62:0], 1'b0};
wire [63:0] r6 = {(arith ? sign_ext[0] : 1'b0), s5[63:1]};
mux m11(d6, dir, l6, r6);
mux m12(s6, shift[0], s5, d6);

// Final Output
assign dout = s6;

endmodule


// ============================================================
// 2:1 Multiplexer (64-bit)
// ============================================================

module mux(
    output [63:0] y,
    input         sel,
    input  [63:0] a,
    input  [63:0] b
);

assign y = sel ? b : a;

endmodule
```

## Barrel Shifter Testbench

```verilog
`timescale 1ns/1ps

module tb_barrel_shifter;

reg  [63:0] din;
reg  [5:0]  shift;
reg         dir;
reg         arth_shift;
wire [63:0] dout;

// Instantiate Barrel Shifter
barrel_shifter uut (
    .din(din),
    .shift(shift),
    .dir(dir),
    .arth_shift(arth_shift),
    .dout(dout)
);

initial begin
    // Display format
    $display("time\t din\t\t\t shift dir arth dout");
    $monitor("%0t\t %h\t %d\t %b\t %b\t %h",
              $time, din, shift, dir, arth_shift, dout);

    // -------- Test 1: Left Shift --------
    din = 64'h123456789ABCDEF0; shift = 0; dir = 0; arth_shift = 0;
    #5 shift = 6'd4;
    #5 shift = 6'd8;
    #10;

    // -------- Test 2: Logical Right Shift --------
    din = 64'h123456789ABCDEF0; dir = 1; arth_shift = 0;
    shift = 6'd2;
    #5 shift = 6'd6;
    #5 shift = 6'd10;
    #10;

    // -------- Test 3: Arithmetic Right Shift --------
    din = 64'hF23456789ABCDEF0; dir = 1; arth_shift = 1;
    shift = 6'd1;
    #5 shift = 6'd4;
    #5 shift = 6'd12;
    #10;

    // -------- Test 4: Edge Case (Sign Bit) --------
    din = 64'h8000000000000001; dir = 1; arth_shift = 1;
    shift = 6'd1;
    #5 shift = 6'd3;
    #10;

    // -------- Test 5: All Ones --------
    din = 64'hFFFFFFFFFFFFFFFF; dir = 1; arth_shift = 1;
    shift = 6'd8;
    #5 shift = 6'd16;
    #10;

    // -------- Test 6: Alternating Pattern --------
    din = 64'hAAAAAAAAAAAAAAAA; dir = 0; arth_shift = 0;
    shift = 6'd1;
    #5 shift = 6'd5;
    #5 shift = 6'd20;
    #10;

    $finish;
end

endmodule
```
## Pipelined 64-bit ALU (3-Stage Pipeline)

```verilog
// ============================================================
// 3-Stage Pipelined 64-bit ALU with Barrel Shifter
// ============================================================

module alu_64(

input clk,
input rst,

input valid_in,

input [63:0] A,
input [63:0] B,
input [3:0] alu_sel,

output reg valid_out,

output reg [63:0] Y,
output reg Zero,
output reg Negative,
output reg Carry,
output reg Overflow

);

// ============================================================
// Stage 1: Input Register Stage
// ============================================================

reg [63:0] A_s1, B_s1;
reg [3:0] sel_s1;
reg valid_s1;

always @(posedge clk or posedge rst) begin
    if (rst) begin
        A_s1 <= 0;
        B_s1 <= 0;
        sel_s1 <= 0;
        valid_s1 <= 0;
    end else begin
        A_s1 <= A;
        B_s1 <= B;
        sel_s1 <= alu_sel;
        valid_s1 <= valid_in;
    end
end

// ============================================================
// Operation Decode
// ============================================================

wire add_en = (sel_s1 == 4'b0000);
wire sub_en = (sel_s1 == 4'b0001);
wire and_en = (sel_s1 == 4'b0010);
wire or_en  = (sel_s1 == 4'b0011);
wire xor_en = (sel_s1 == 4'b0100);

wire shl_en = (sel_s1 == 4'b0101);
wire shr_en = (sel_s1 == 4'b0110);
wire asr_en = (sel_s1 == 4'b0111);

wire slt_en = (sel_s1 == 4'b1000);

// ============================================================
// Stage 2: Execution Stage
// ============================================================

// -------- Adder/Subtractor --------
wire [63:0] B_mod = sub_en ? ~B_s1 : B_s1;
wire Cin = sub_en ? 1'b1 : 1'b0;

wire [64:0] adder_out = {1'b0,A_s1} + {1'b0,B_mod} + Cin;
wire [63:0] add_out = adder_out[63:0];

// -------- Logic Unit --------
wire [63:0] logic_out =
       and_en ? (A_s1 & B_s1) :
       or_en  ? (A_s1 | B_s1) :
       xor_en ? (A_s1 ^ B_s1) :
       64'b0;

// -------- Barrel Shifter --------
wire dir = shr_en | asr_en;
wire arth_shift = asr_en;

wire [63:0] shift_out;

barrel_shifter SHIFT1(
    .din(A_s1),
    .shift(B_s1[5:0]),
    .dir(dir),
    .arth_shift(arth_shift),
    .dout(shift_out)
);

// -------- SLT --------
wire [63:0] slt_out = (A_s1 < B_s1) ? 64'd1 : 64'd0;

// -------- Pipeline Registers --------
reg [63:0] add_r, logic_r, shift_r, slt_r;
reg [64:0] adder_out_r;

reg add_en_r, sub_en_r, and_en_r, or_en_r, xor_en_r;
reg shl_en_r, shr_en_r, asr_en_r, slt_en_r;
reg valid_s2;

always @(posedge clk or posedge rst) begin
    if (rst) begin
        add_r <= 0; logic_r <= 0; shift_r <= 0; slt_r <= 0;
        adder_out_r <= 0;
        add_en_r <= 0; sub_en_r <= 0;
        and_en_r <= 0; or_en_r <= 0; xor_en_r <= 0;
        shl_en_r <= 0; shr_en_r <= 0; asr_en_r <= 0;
        slt_en_r <= 0;
        valid_s2 <= 0;
    end else begin
        add_r <= add_out;
        logic_r <= logic_out;
        shift_r <= shift_out;
        slt_r <= slt_out;
        adder_out_r <= adder_out;

        add_en_r <= add_en;
        sub_en_r <= sub_en;
        and_en_r <= and_en;
        or_en_r <= or_en;
        xor_en_r <= xor_en;
        shl_en_r <= shl_en;
        shr_en_r <= shr_en;
        asr_en_r <= asr_en;
        slt_en_r <= slt_en;

        valid_s2 <= valid_s1;
    end
end

// ============================================================
// Stage 3: Output Selection Stage
// ============================================================

reg [63:0] Y_exec;
reg valid_s3;

// Operation Selection
always @(*) begin
    if (add_en_r | sub_en_r)
        Y_exec = add_r;
    else if (and_en_r | or_en_r | xor_en_r)
        Y_exec = logic_r;
    else if (shl_en_r | shr_en_r | asr_en_r)
        Y_exec = shift_r;
    else if (slt_en_r)
        Y_exec = slt_r;
    else
        Y_exec = 64'b0;
end

// Valid Signal Pipeline
always @(posedge clk or posedge rst) begin
    if (rst)
        valid_s3 <= 0;
    else
        valid_s3 <= valid_s2;
end

// ============================================================
// Final Output + Flags
// ============================================================

always @(posedge clk or posedge rst) begin
    if (rst) begin
        Y <= 0;
        Zero <= 0;
        Negative <= 0;
        Carry <= 0;
        Overflow <= 0;
        valid_out <= 0;
    end else begin
        Y <= Y_exec;

        Zero <= ~|Y_exec;
        Negative <= Y_exec[63];

        Carry <= (add_en_r | sub_en_r) ? adder_out_r[64] : 1'b0;

        Overflow <=
        (add_en_r) ? ((A_s1[63] & B_s1[63] & ~Y_exec[63]) |
                     (~A_s1[63] & ~B_s1[63] & Y_exec[63])) :
        (sub_en_r) ? ((A_s1[63] & ~B_s1[63] & ~Y_exec[63]) |
                     (~A_s1[63] & B_s1[63] & Y_exec[63])) :
        1'b0;

        valid_out <= valid_s3;
    end
end

endmodule
```
## Pipelined ALU Testbench

```verilog
`timescale 1ns/1ps

module tb_alu_64;

reg clk;
reg rst;
reg valid_in;

reg [63:0] A, B;
reg [3:0] alu_sel;

wire [63:0] Y;
wire Zero, Negative, Carry, Overflow;
wire valid_out;

// Instantiate DUT
alu_64 dut(
    .clk(clk),
    .rst(rst),
    .valid_in(valid_in),
    .A(A),
    .B(B),
    .alu_sel(alu_sel),
    .valid_out(valid_out),
    .Y(Y),
    .Zero(Zero),
    .Negative(Negative),
    .Carry(Carry),
    .Overflow(Overflow)
);

// Clock generation (10ns period)
always #5 clk = ~clk;

// Display signals
initial begin
    $display("Time\tvalid_in\tA\tB\tSEL\tvalid_out\tY\tZ\tN\tC\tV");
    $monitor("%0t\t%b\t%d\t%d\t%b\t%b\t%d\t%b\t%b\t%b\t%b",
             $time, valid_in, A, B, alu_sel,
             valid_out, Y, Zero, Negative, Carry, Overflow);
end

// Test stimulus
initial begin
    clk = 0;
    rst = 1;
    valid_in = 0;
    A = 0;
    B = 0;
    alu_sel = 0;

    // Apply reset
    #10 rst = 0;

    // -------- TEST CASES --------

    @(posedge clk);
    valid_in = 1; A = 10; B = 5; alu_sel = 4'b0000; // ADD

    @(posedge clk);
    A = 10; B = 5; alu_sel = 4'b0001; // SUB

    @(posedge clk);
    A = 12; B = 10; alu_sel = 4'b0010; // AND

    @(posedge clk);
    A = 12; B = 10; alu_sel = 4'b0011; // OR

    @(posedge clk);
    A = 12; B = 10; alu_sel = 4'b0100; // XOR

    @(posedge clk);
    A = 4; B = 1; alu_sel = 4'b0101; // SHL

    @(posedge clk);
    A = 16; B = 2; alu_sel = 4'b0110; // SHR

    @(posedge clk);
    A = -8; B = 1; alu_sel = 4'b0111; // ASR

    @(posedge clk);
    A = 3; B = 5; alu_sel = 4'b1000; // SLT

    // Stop input
    @(posedge clk);
    valid_in = 0;

    // Wait for pipeline to flush
    repeat(10) @(posedge clk);

    $finish;
end

endmodule
```


