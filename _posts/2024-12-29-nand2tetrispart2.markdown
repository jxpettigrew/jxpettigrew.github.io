---
layout: post
title: "Nand2Tetris, Unit 2"
description: "Constructing an ALU"
date: 2024-12-29
categories: [nand2Tetris, computer architecture]
---
## Project 2
In this project, we use the logic gates constructed in Project 1 to build adder chips of increasing complexity. These chips are then used to build a functional arithmetic logic unit, the first major component of our eventual CPU.

## Key Concepts
**Two's complement**: the most common way to represent signed integers in binary, can be derived by inverting every bit and adding 1

**Arithmetic logic unit**: the part of a CPU that handles arithmetic and logical operations

## Chips
### Half Adder
```
PARTS:
Xor(a=a, b=b, out=sum);
And(a=a, b=b, out=carry);
```
The most basic of the adding chips we're asked to build for this project is the half adder, which adds two bits. Because it can only add two bits at a time, it cannot handle potential carry bits, hence the name "half" adder.

| Input A | Input B | Sum | Carry|
|:--:|:--:|:--:|:--:|
| 0 | 0 | 0 | 0 |
| 0 | 1 | 1 | 0 |
| 1 | 0 | 1 | 0 |
| 1 | 1 | 0 | 1 |

This chip must output two bits: the least significant bit, called the sum, and the most significant bit, called the carry. As shown in the half adder truth table above, the sum output needs to be 1 when either *a* or *b* (but not both) is 1, and the carry output needs to be 1 only when both *a* *and* *b* are 1. If this sounds familiar, it's because the outputs are identical to those of the XOR and AND gates we constructed in Project 1. And indeed the implementation for this chip is simply running the *a* and *b* inputs first through an XOR gate (for the sum) and then through an AND gate (for the carry).

### Full Adder
```
PARTS:
HalfAdder(a=a, b=b, sum=sum1, carry=carry1);
HalfAdder(a=sum1, b=c, sum=sum, carry=carry2);
Xor(a=carry1, b=carry2, out=carry);
```
The full adder can add three bits at a time, which means it can handle carry bits, allowing us to complete the full addition process for any two (binary) numbers.

To implement this, we add the first two inputs with a half adder, producing a sum and a carry (as discussed in the half adder implementation above). We then use another half adder to add that sum bit to the third input, which produces the final sum output and a second carry bit. Lastly, we run the two carry bits through an XOR gate to produce the final carry output.

### Add16
```
PARTS:
HalfAdder(a=a[0], b=b[0], sum=out[0], carry=carry1);
FullAdder(a=a[1], b=b[1], c=carry1, sum=out[1], carry=carry2);
FullAdder(a=a[2], b=b[2], c=carry2, sum=out[2], carry=carry3);
[...]
FullAdder(a=a[15], b=b[15], c=carry15, sum=out[15], carry=carry16);
```
The 16-bit adder simply takes in two 16-bit numbers (input buses *a* and *b*) and does bitwise addition on each corresponding pair of bits from right (least significant) to left (most significant). 

We execute the first addition operation using a half adder and then feed the carry output from that into a full adder, which will add that carry bit and the next pair of bits (*a*[1] and *b*[1]). The carry output from *that* full adder will be piped into yet another full adder, and so on until all 16 pairs of bits (along with any carry bits) have been added.

### Inc16
```
PARTS:
HalfAdder(a=in[0], b=true, sum=out[0], carry=carry1);
FullAdder(a=in[1], b=false, c=carry1, sum=out[1], carry=carry2);
FullAdder(a=in[2], b=false, c=carry2, sum=out[2], carry=carry3);
[...]
FullAdder(a=in[15], b=false, c=carry15, sum=out[15], carry=carry16);
```
The 16-bit incrementor is effectively identical to the 16-bit adder — but instead of adding two 16-bit buses, it simply adds the constant 1 to a 16-bit bus (*a*).

### ALU
```
PARTS:
Mux16(a=x[0..15], b=false, sel=zx, out=outzx);
Not16(in=outzx, out=notOutzx);
Mux16(a=outzx, b=notOutzx, sel=nx, out=outnx);
Mux16(a=y[0..15], b=false, sel=zy, out=outzy);
Not16(in=outzy, out=notOutzy);
Mux16(a=outzy, b=notOutzy, sel=ny, out=outny);
And16(a=outnx, b=outny, out=andxy);
Add16(a=outnx, b=outny, out=addxy);
Mux16(a=andxy, b=addxy, sel=f, out=fout);
Not16(in=fout, out=notFout);
Mux16(a=fout, b=notFout, sel=no, out=out, out[15]=ng, out[0..7]=half1, out[8..15]=half2);
Or8Way(in=half1, out=zeroCheck1);
Or8Way(in=half2, out=zeroCheck2);
Or(a=zeroCheck1, b=zeroCheck2, out=notZr);
Not(in=notZr, out=zr);
```
The arithmetic logic unit is what we've been building toward so far. The ALU is perhaps the most crucial component of a CPU, as it performs the actual computation implied by the word "computer." Our ALU will take in two 16-bit buses (*x* and *y*) as input and must be able to perform bitwise zeroing and negation on them (determined by input bits *zx*, *zy*, *nx*, and *ny*), then add or AND them together depending on an input bit *f*, and finally negate the output (or not) based on an input bit *no*. It must also flag negative inputs and inputs equal to zero via output bits *ng* and *zr*.

The zero and negation operations are trivially easy to implement: we just use a Mux16 gate to output zero if the corresponding input bit equals 1, pass that output through a Not16 gate to get its negative, and then use another Mux16 gate to output the negative if the corresponding input bit (*nx* or *ny*) equals 1.  Of course, we must do this for both the *x* and *y* input buses.

The output of those last Mux16 gates can then either be added together (*x* + *y*) using an Add16 gate or ANDed together (*x* & *y*) using an And16 gate. We determine which operation to execute by setting up another Mux16 gate and using the *f* input as the selection bit.

To implement the final operation, we run the output of the previous Mux16 gate through a Not16 gate, then use yet another Mux16 gate to determine whether to select the negated output or non-negated output using the *no* bit. While this *is* the final operation required for the gate, we aren't done quite yet, as we still have to check whether the output is zero or negative and return the results as *ng* and *zr*.

Checking for negation is simple: because we're using a two's complement system, the most significant bit will be 0 if the number is positive and 1 if the number is negative. Therefore, in the previous Mux16 gate, in addition to getting the final output, we can also set the most significant bit of the output (*out[15]*) equal to *ng*.

Almost done! Now we just have to check whether the output is equal to zero, which we could do by running each bit through an Or16 gate. While that would be the easiest way to implement this last check bit, we don't have an Or16 gate in this course. But we *do* have an Or8 gate, and we can use two of those and then run their outputs through a final Or gate to replicate the function of an Or16 gate. All that's left is running the Or gate output through a Not gate to get the correct *zr* value.

## Concluding thoughts

I'm surprised at the relative ease of building the ALU. Considering it performs the main function of the CPU, I assumed its construction would be complex, but my implementation uses just 15 gates, most of which are the same basic gates built in Project 1. Nor was the logic of the implementation challenging once I realized the primary action required for the chip is simply using the input bits to repeatedly select one of two functions, which is exactly what the multiplexer gates are for.

The only issue I had was actually an issue with the application itself — when trying to implement the output bits, I kept getting "Internal pins can't be subscripted or indexed" and "Output gates cannot be used as input" errors. The way around these errors was simply to index the outputs of the final Mux16 gate directly, as you can see in my implementation above, rather than try to index them as inputs to new gates.