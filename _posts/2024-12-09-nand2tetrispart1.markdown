---
layout: post
title: "Nand2Tetris, Unit 1"
description: "Constructing logic gates"
date: 2024-12-09
categories: [nand2Tetris, computer architecture]
---

One of my favorite topics in my computer science degree was computer architecture. From logic gates to instruction sets to assembly language, few aspects of the field are more fascinating to me than the logic and components that allow a computer to *compute*. While I understand most of these concepts in theory, I've always wanted to revisit them in a more in-depth, hands-on fashion because they're so foundational.

Enter [nand2Tetris](nand2tetris.org), a course that starts from the most fundamental building blocks of architecture and takes you through the construction of a functioning CPU, along with RAM and an assembler (a subsequent course focuses on building a high-level programming language, a corresponding compiler, and an operating system for your new makeshift virtual computer). It's just the sort of hands-on experience I wanted, and so I thought I'd share my thought process as I work through the projects in the course.

## Project 1
The first project is dedicated to building the logic gates that will eventually be used to construct an ALU. To do this, we're given the very basic Nand (not-and) gate and must use it along with Boolean logic to construct other logic gates of increasing complexity.

## Key Concepts
**Boolean logic**: a form of logic that deals with operations on binary values (0/1, off/on, True/False) and forms the basis for digital circuit design

**Boolean operators**: terms like AND, OR, and NOT, which are used to combine or invert Boolean values according to certain logical rules

**Logic gates**: circuits designed to perform Boolean operations on one or more inputs in order to produce a certain output

## Gates
Small disclaimer: not all of these gate implementations are fully optimized. In some cases, I opted for a more intuitive but slightly less optimized implementation. In other cases, I simply haven't worked out the absolute most optimal implementations myself, but I don't want to just look up the answers. Optimization may merit its own separate post sometime in the future.

### Not
```Boolean
PARTS:
Nand(a=in, b=in, out=out);
```
The Not gate is the first gate we're asked to implement, and luckily it's also one of the simplest. It can be confusing to think about implementing a single-input gate using a two-input gate, but the key is remembering that inputs can have unlimited fan-out. That is, the input *a* can fan out to be fed into *both* the first and second input pins of a Nand gate. So (*a* AND *a*) simplifies to *a* and is then inverted by the Nand gate to produce NOT(*a*), exactly what we're looking for.

### And
```Boolean
PARTS:
Nand(a=a, b=b, out=nandOut);
Not(in=nandOut, out=out);
```
This one shouldn't need much explanation. NOT(*a* AND *b*) is already the inversion of (*a* AND *b*), so when we pass this into the Not gate we just implemented, the two NOTs cancel one another out, and we get (*a* AND *b*).

### Or
```Boolean
PARTS:
Not(in=a, out=notA);
Not(in=b, out=notB);
Nand(a=notA, b=notB, out=out);
```
According to De Morgan's laws, NOT(*a* AND *b*) = NOT(*a*) OR NOT(*b*), so if we invert *a* and *b* and pass those values through a Nand gate, we get (*a* OR *b*).

### Xor
```Boolean
PARTS:
Not(in=a, out=notA);
Not(in=b, out=notB);
And(a=a, b=notB, out=aAndNotB);
And(a=notA, b=b, out=notAAndB);
Or(a=aAndNotB, b=notAAndB, out=out);
```
Outputs 1 if *a* is 1 OR *b* is 1 but not both. Also a good example of the cumulative nature of gate construction, as we can now use the Not, And, and Or gates we just designed.

### Mux
```Boolean
PARTS:
Not(in=sel, out=notSel);
And(a=a, b=notSel, out=aAndNotSel);
And(a=b, b=sel, out=bAndSel);
Or(a=aAndNotSel, b=bAndSel, out=out);
```
The multiplexer gate throws in a new twist with a third input: the selection bit. We want this gate to output 1 either when the input is *a* AND the selection bit is 0 OR when the input is *b* AND the selection bit is 1, so all we need to do is set up the corresponding gates.

### DMux
```Boolean
PARTS:
Not(in=sel, out=notSel);
And(a=in, b=sel, out=b);
And(a=in, b=notSel, out=a);
```
The demultiplexer gate performs the opposite of the multiplexer operation: that is, it takes an input and assigns it to one of two different output channels. To implement it, we need to use two And gates: one takes the values of the input and the inversion of the selection bit and outputs *a*, while the other takes the values of the input and the selection bit and outputs *b*.

### 16-bit gates
The next part of the assignment involves implementing 16-bit versions of the Not, And, Or, and Mux gates. This is very simple, as it just requires performing sixteen bitwise operations on the 16-bit buses being input using the corresponding 1-bit gates we just designed. For example, the parts for the And gate look like this:
```
PARTS:
And(a=a[0], b=b[0], out=out[0]);
And(a=a[1], b=b[1], out=out[1]);
[...]
And(a=a[15], b=b[15], out=out[15]);
```
And so on, for each 16-bit gate. No twists or surprises here.

### Or8Way
Much like the 16-bit gates, the 8-way Or gate might look superficially complicated but turns out to be another extremely simple implementation. For this one, we start by passing the first two bits of the 8-bit input bus through an Or gate, the output of which is a placeholder value. We then pass this placeholder value through another Or gate along with the third bit of the bus, and so on until we finally perform the OR operation on the cumulative placeholder value and the last bit of the bus, like so:
```Boolean
PARTS:
Or(a=in[0], b=in[1], out=temp1);
Or(a=temp1, b=in[2], out=temp2);
Or(a=temp2, b=in[3], out=temp3);
[...]
Or(a=temp6, b=in[7], out=out);
```

### Mux4Way16
```Boolean
PARTS:
Mux16(a=a, b=b, sel=sel[0], out=temp1);
Mux16(a=c, b=d, sel=sel[0], out=temp2);
Mux16(a=temp1, b=temp2, sel=sel[1], out=out);
```
The Mux16 gate we designed earlier is 2-way, using a 1-bit selector value to output one of two values, iterating through two 16-bit input buses. The 4-way gate is similar, but it uses a 2-bit selector and operates on *four* 16-bit input buses rather than two.

We can implement this by first setting up two Mux16 gates: one using the least significant bit of the 2-bit selector (sel[0]) to select either input *a* or input *b*, the other performing the same operation to select either input *c* or input *d*. We then pass the two selected buses to a third Mux16 gate, which uses the most significant bit of the 2-bit selector (sel[1]) to select between either *a* and *c* or *b* and *d* depending on the results of the previous operations.

### Mux8Way16
```Boolean
PARTS:
Mux4Way16(a=a, b=b, c=c, d=d, sel=sel[0..1], out=temp1);
Mux4Way16(a=e, b=f, c=g, d=h, sel=sel[0..1], out=temp2);
Mux16(a=temp1, b=temp2, sel=sel[2], out=out);
```
The 8-way multiplexer gate extends the logic of the 4-way mux gate, with two 4-way gates taking the place of the first two 2-way gates in the Mux4Way16 implementation above, each using the two least significant bits (sel[0..1]) of the selector to determine its output. A Mux16 gate then uses the most significant bit (sel[2]) of the 3-bit selector to select the correct value.

### DMux4Way
```Boolean
PARTS:
DMux(in=in, sel=sel[0], a=aOrC, b=bOrD);
DMux(in=bOrD, sel=sel[1], a=b, b=d);
DMux(in=aOrC, sel=sel[1], a=a, b=c);
```
The 4-way demultiplexer gate operates much like the basic 2-way demultiplexer gate we've already designed. For the first demux gate, we'll input the data bit and the least significant bit of the selector. If the least significant selector bit is 0, the output of the first gate will be channeled to another demux gate that will use the *most* significant selector bit to determine whether the final output should be *a* or *c*; if the least significant selector bit is 1, the output will be channeled to a third gate, which will determine whether the output is *b* or *d*.

Put more simply, we're breaking the demultiplexing process into two phases by splitting the 2-bit selector into two individual selector bits.

### DMux8Way
```Boolean
PARTS:
DMux(in=in, sel=sel[2], a=former, b=latter);
DMux4Way(in=former, sel=sel[0..1], a=a, b=b, c=c, d=d);
DMux4Way(in=latter, sel=sel[0..1], a=e, b=f, c=g, d=h);
```
If you've noticed a pattern here, then you can probably guess that the 8-way demultiplexer gate extends the logic of the 4-way demux gate. We first set up a basic 2-way demux gate with the most significant bit of the 3-bit selector to split the gates into two groups of four. We then use two DMux4Way gates to handle these two groups, using the two least significant bits of the selector to finally channel the output to the correct gate.

## Concluding thoughts
Anyone who's spent any time at all programming probably won't find the most elementary gates here all that revelatory, but personally, the real value of the project was in being forced to become familiar with the multiplexer/demultiplexer gates and multi-bit/multi-way gates. You can read about these things all day, but that will never provide the same intuitive understanding as actually using (or, in this case, building) them. It's also a good exercise in Boolean logic if you're unfamiliar or out of practice.

I'm looking forward to easing into a bit more complexity later in the course. Very curious about how these gates can be combined to form an ALU, which we'll learn next unit.

Here's hoping the next project is a bit less repetitive to write (and read) about!