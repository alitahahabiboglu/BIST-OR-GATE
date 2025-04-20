# BIST-OR-GATE
BIST implementation to an OR gate

//VERILOG CODE

`timescale 1ns / 1ps

// OR Gate Module
module ORGate (
    input wire a,
    input wire b,
    output wire y
);
    assign y = a | b;
endmodule

// Scan-Enabled Flip-Flop
module ScanFlipFlop (
    input wire clk,
    input wire reset,
    input wire d,
    input wire scan_in,
    input wire scan_enable,
    output reg q
);
    always @(posedge clk or posedge reset) begin
        if (reset) q <= 0;
        else if (scan_enable) q <= scan_in;
        else q <= d;
    end
endmodule

// Top Level with BIST
module TopLevelWithBIST (
    input wire clk,
    input wire reset,
    input wire d1,
    input wire d2,
    input wire scan_in1,
    input wire scan_enable,
    input wire bist_enable,
    output wire or_output,
    output wire [1:0] scan_outs // observable scan chain outputs
);
    wire q1, q2;
    wire scan_input_mux;

    assign scan_input_mux = (bist_enable) ? 1'b1 : scan_in1; // BIST injects pattern '1'

    ScanFlipFlop ff1 (
        .clk(clk),
        .reset(reset),
        .d(d1),
        .scan_in(scan_input_mux),
        .scan_enable(scan_enable),
        .q(q1)
    );

    ScanFlipFlop ff2 (
        .clk(clk),
        .reset(reset),
        .d(d2),
        .scan_in(q1),
        .scan_enable(scan_enable),
        .q(q2)
    );

    ORGate or1 (
        .a(q1),
        .b(q2),
        .y(or_output)
    );

    assign scan_outs = {q2, q1};
endmodule


//TEST BENCH

`timescale 1ns / 1ps

module BIST_Testbench;

    reg clk;
    reg reset;
    reg d1, d2;
    reg scan_enable;
    reg scan_in1;
    reg bist_enable;
    wire or_output;
    wire [1:0] scan_outs;

    TopLevelWithBIST uut (
        .clk(clk),
        .reset(reset),
        .d1(d1),
        .d2(d2),
        .scan_in1(scan_in1),
        .scan_enable(scan_enable),
        .bist_enable(bist_enable),
        .or_output(or_output),
        .scan_outs(scan_outs)
    );

    // Clock generation
    initial begin
        clk = 0;
        forever #5 clk = ~clk;
    end

    initial begin
        $display("=== BIST-Enabled Scan Chain Test ===");

        reset = 1;
        d1 = 0; d2 = 0;
        scan_enable = 0;
        scan_in1 = 0;
        bist_enable = 0;
        #20;

        reset = 0;
        #10;

        // Normal scan shift
        scan_enable = 1;
        scan_in1 = 0;
        #10;

        // Enable BIST
        bist_enable = 1;
        #10;
        $display("BIST Activated: scan_outs = %b, OR Output = %b", scan_outs, or_output);

        #10 bist_enable = 0;

        $display("=== End of BIST Test ===");
        $finish;
    end
endmodule
