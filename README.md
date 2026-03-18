# 64bit-ALU-with-Barrel-Shifter-Verilog

##  Overview

This project implements a **64-bit Arithmetic Logic Unit (ALU)** in Verilog integrated with a **high-speed barrel shifter**. The design supports arithmetic, logical, shift, and comparison operations with proper flag generation.

##  Features

* 64-bit datapath design
* Modular architecture (ALU + Barrel Shifter)
* Supports:

  * Addition & Subtraction
  * Bitwise AND, OR, XOR
  * Logical Shift Left (SHL)
  * Logical Shift Right (SHR)
  * Arithmetic Shift Right (ASR)
  * Set Less Than (SLT)
* Status Flags:

  * Zero
  * Negative
  * Carry
  * Overflow

##  Barrel Shifter Design

* Logarithmic shifter using 6 stages
* Shift range: 0–63 bits
* Supports both logical and arithmetic shifts
* Time complexity: O(log N)

##  ALU Operations

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



##  Simulation

Simulated using xilinx Vivado.

##  Applications

* CPU datapath design
* Digital signal processing
* Embedded systems
  
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

## 64-bit ALU Design

```verilog
// ============================================================
// 64-bit ALU with Barrel Shifter Integration
// ============================================================

module alu_64(

input [63:0] A,
input [63:0] B,
input [3:0] alu_sel,

output reg [63:0] Y,

output Zero,
output Negative,
output Carry,
output Overflow

);

// -------- Operation Decode --------
wire add_en = (alu_sel == 4'b0000);
wire sub_en = (alu_sel == 4'b0001);
wire and_en = (alu_sel == 4'b0010);
wire or_en  = (alu_sel == 4'b0011);
wire xor_en = (alu_sel == 4'b0100);

wire shl_en = (alu_sel == 4'b0101);
wire shr_en = (alu_sel == 4'b0110);
wire asr_en = (alu_sel == 4'b0111);

wire slt_en = (alu_sel == 4'b1000);

// -------- Adder / Subtractor --------
wire [63:0] B_mod;
wire Cin;

assign B_mod = sub_en ? ~B : B;
assign Cin   = sub_en ? 1'b1 : 1'b0;

wire [64:0] adder_out;

assign adder_out = {1'b0,A} + {1'b0,B_mod} + Cin;

wire [63:0] add_out =
       (add_en | sub_en) ? adder_out[63:0] : 64'b0;

// -------- Logic Unit --------
wire [63:0] logic_out =
       and_en ? (A & B) :
       or_en  ? (A | B) :
       xor_en ? (A ^ B) :
       64'b0;

// -------- Barrel Shifter Control --------
wire dir;
wire arth_shift;

assign dir = shr_en | asr_en;
assign arth_shift = asr_en;

wire [63:0] shift_out;

// Barrel Shifter Instance
barrel_shifter SHIFT1(
    .din(A),
    .shift(B[5:0]),
    .dir(dir),
    .arth_shift(arth_shift),
    .dout(shift_out)
);

// -------- Set Less Than --------
wire [63:0] slt_out;

assign slt_out = slt_en ? (A < B ? 64'd1 : 64'd0) : 64'd0;

// -------- Output Selection --------
always @(*) begin
    case(alu_sel)

    4'b0000,
    4'b0001: Y = add_out;

    4'b0010,
    4'b0011,
    4'b0100: Y = logic_out;

    4'b0101,
    4'b0110,
    4'b0111: Y = shift_out;

    4'b1000: Y = slt_out;

    default: Y = 64'b0;

    endcase
end

// -------- Flags --------
assign Zero = ~|Y;

assign Negative = Y[63];

assign Carry =
      (add_en | sub_en) ? adder_out[64] : 1'b0;

assign Overflow =
(add_en) ? ((A[63] & B[63] & ~Y[63]) |
           (~A[63] & ~B[63] & Y[63])) :

(sub_en) ? ((A[63] & ~B[63] & ~Y[63]) |
           (~A[63] & B[63] & Y[63])) :

1'b0;

endmodule
```

## ALU Testbench

```verilog
`timescale 1ns/1ps

module alu_tb;

reg [63:0] A;
reg [63:0] B;
reg [3:0] alu_sel;

wire [63:0] Y;
wire Zero;
wire Negative;
wire Carry;
wire Overflow;

// Instantiate ALU
alu_64 DUT(
    .A(A),
    .B(B),
    .alu_sel(alu_sel),
    .Y(Y),
    .Zero(Zero),
    .Negative(Negative),
    .Carry(Carry),
    .Overflow(Overflow)
);

initial begin

    // Display header
    $display("time\tA\tB\tALU\tY\tZ\tN\tC\tV");

    // Monitor values continuously
    $monitor("%0t\t%d\t%d\t%b\t%d\t%b\t%b\t%b\t%b",
             $time,A,B,alu_sel,Y,Zero,Negative,Carry,Overflow);

    // -------- Arithmetic Operations --------
    A=15; B=10; alu_sel=4'b0000; #10;   // ADD
    A=20; B=5;  alu_sel=4'b0001; #10;   // SUB

    // -------- Logic Operations --------
    A=64'hF0F0; B=64'h0FF0; alu_sel=4'b0010; #10; // AND
    A=64'hF0F0; B=64'h0FF0; alu_sel=4'b0011; #10; // OR
    A=64'hAAAA; B=64'h5555; alu_sel=4'b0100; #10; // XOR

    // -------- Shift Operations --------
    A=64'h1;  B=2; alu_sel=4'b0101; #10;     // SHL
    A=64'h10; B=2; alu_sel=4'b0110; #10;     // SHR
    A=-64'd16; B=2; alu_sel=4'b0111; #10;    // ASR

    // -------- Comparison --------
    A=5; B=10; alu_sel=4'b1000; #10;         // SLT

    // -------- Edge Cases --------
    A=5; B=5; alu_sel=4'b0001; #10;          // Zero result
    A=5; B=10; alu_sel=4'b0001; #10;         // Negative result

    // -------- Overflow Test --------
    A=64'h7FFFFFFFFFFFFFFF; B=1; alu_sel=4'b0000; #10;

    $finish;

end

endmodule
```
