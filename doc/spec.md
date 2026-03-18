# fmultiplier — FP32 Multiplier (Handshake, Multi-Cycle, IEEE-754)

## Overview
`fmultiplier` is a **multi-cycle** single-precision floating-point multiplier using a valid/out_valid handshake. It produces bit-accurate IEEE-754 results for **normal numbers only** (exponent 1–254). Special cases (subnormals/inf/NaN) can be ignored — the testbench uses only normal inputs.

The design must be a **7-stage pipeline** controlled by `counter` (1 to 7). `out_valid` asserts exactly 7 cycles after a `valid` pulse when idle.

## Interface (do not change)
```verilog
module fmultiplier (
    input wire clk,
    input wire rst,
    input wire valid,           // 1-cycle start pulse
    input wire [31:0] a, b,
    output reg [31:0] z,
    output reg out_valid        // 1-cycle pulse when result ready
);
Pipeline Stages (follow exactly — use non‑blocking <=)
Stage 1 (counter=1) — Unpack
verilog
a_s <= a_r[31]; b_s <= b_r[31];
a_e <= $signed(a_r[30:23]) - 10'd127;
b_e <= $signed(b_r[30:23]) - 10'd127;
a_m <= {1'b0, a_r[22:0]};
b_m <= {1'b0, b_r[22:0]};
Stage 2 (counter=2) — Hidden bit
verilog
if (a_r[30:23] != 8'h00) a_m[23] <= 1'b1;
if (b_r[30:23] != 8'h00) b_m[23] <= 1'b1;
Stage 3 (counter=3) — Normalize inputs (rarely needed for normal numbers)
verilog
if (!a_m[23] && a_m != 0) begin a_m <= a_m << 1; a_e <= a_e - 10'd1; end
if (!b_m[23] && b_m != 0) begin b_m <= b_m << 1; b_e <= b_e - 10'd1; end
Stage 4 (counter=4) — Multiply core
verilog
z_s <= a_s ^ b_s;
z_e <= a_e + b_e;           // NO extra +1 here
product <= a_m * b_m;       // 48-bit product
Stage 5 (counter=5) — Extract mantissa + rounding bits
verilog
// 48-bit product: check MSB for normalization shift
if (product[47]) begin
    z_m <= product[47:24];
    guard_bit <= product[23];
    round_bit <= product[22];
    sticky <= |product[21:0];
    z_e <= z_e + 10'd1;
end else begin
    z_m <= product[46:23];
    guard_bit <= product[22];
    round_bit <= product[21];
    sticky <= |product[20:0];
end
Stage 6 (counter=6) — Normalize if needed
verilog
if (!z_m[23] && z_m != 0) begin
    z_m <= {z_m[22:0], guard_bit};
    guard_bit <= round_bit;
    round_bit <= sticky;
    sticky <= 1'b0;
    z_e <= z_e - 10'd1;
end
Stage 7 (counter=7) — Round‑to‑nearest‑even + Pack
verilog
// Simple RNE rounding
if (guard_bit && (round_bit || sticky || z_m[0])) begin
    if (z_m == 24'hFFFFFF) begin
        z_m <= 24'h800000;
        z_e <= z_e + 10'd1;
    end else begin
        z_m <= z_m + 24'd1;
    end
end

// Pack result
if (result_is_nan) z <= {z_s, 8'hFF, 23'h400000};
else if (result_is_zero) z <= {z_s, 31'b0};
else if (result_is_inf) z <= {z_s, 8'hFF, 23'b0};
else if ($signed(z_e) >= $signed(10'd128)) z <= {z_s, 8'hFF, 23'b0};
else if ($signed(z_e) <= $signed(-10'd127)) z <= {z_s, 8'h00, z_m[22:0]};
else z <= {z_s, (z_e + 10'd127)[7:0], z_m[22:0]};
Critical Verilog rules (must follow)
All pipeline assignments must use non‑blocking <=.

Use $signed() for any comparison involving z_e, a_e, b_e.

Do not declare local reg variables inside the always block for pipeline values.

The special‑case flags (result_is_zero, etc.) are already defined – just use them.

Example (1.0 × 1.0)
Inputs: 3F800000 × 3F800000

After Stage 3: a_m = b_m = 24'h800000, a_e = b_e = 0

Stage 4: product = 48'h400000000000

Stage 5: product[47]=0 → z_m = 24'h800000, z_e = 0

Final result must be 3F800000

Common pitfalls to avoid
Forgetting $signed() on exponent comparisons.

Using blocking = in the pipeline stages.

Wrong bit slicing in Stage 5 – always use the exact if (product[47]) pattern.

Mixing intermediate/final values in Stage 7 – keep calculations simple.EOL
