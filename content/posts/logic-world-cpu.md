+++ 
draft = false
date = 2026-11-14T12:36:55-05:00
title = "A 16 bit CPU in Logic World"
description = ""
slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++
I created a 16 bit harvard architecture CPU in the "game" [Logic World](https://logic.world/)! I call it the zcpu_v0 architecture! 

![zcpu in all it's messy glory](/pictures/logic-world-cpu/big-picture.png)

I've been interested for a while in learning the lower level side of software and hardware, and you can't get much lower than logic gates. I played another game, [Turing Complete](https://turingcomplete.game/), up through around the midpoint of the campaign to learn some of the cpu hardware basics in a guided way. At that point Turing Complete had gotten up to a working 8bit cpu. I was ready to jump into 16 bits on my own, and popped over to Logic World.

Moving into the 3D space of logic world, and building out the components there is just way more fun than drawing lines on a 2D screen. You can fly around the structures of the processor, and it gives an awesome sense of scale to the complexity of building out the CPU components.

## The CPU Design

The zcpu is a simple 16 bit, Harvard architecture machine. Harvard architecture means that unlike a modern cpu, this one has separate ROM instruction memory and runtime RAM. Having separate program and runtime memory keeps the architecture and instruction decoder simpler, but I will likely try to adapt this into a von-neumann architecture later.

The CPU is more like something a modern microcontroller would use rather than a full modern computer. Real computers with OSes have been built on similar hardware (see the Apple II)! But modern operating systems expect a few more hardware features: privileged instruction sets, interrupt timers, memory management units for mapping process memory, etc. All hardware features in modern CPUs that are used by operating systems to run multiple processes in a way that's efficient without letting them interfere with each other. Other than that though, this is a fully working CPU.

To keep the memory design and interface simple, RAM is word-addressed, and each address points to a 16bit word value.

The cpu has 16 registers, labeled reg0-15. There is a single ALU with just a few operations: Add, sub, adc, and, or, neg, SHR (shift right).

Reg1 and reg2 are hardwired into the ALU, so all ALU ops are on those registers. For ops that only use a single operand (neg, shr) only reg1 is used.

The CPU is made of of a few core components:

- [Register File](#the-register-file)
- [Program Counter](#program-counter-and-register)
- [Arithmetic Logic Unit](#alu)
- [Instruction Decoder](#instruction-decoder)
- [ROM - Read Only Memory](#the-rom-block)
- [Random Access Memory](#ram)

Now, honestly the RAM and ROM really aren't part of the core CPU. But I'm including them here anyway.

And there is a section covering the Instruction Set Architecture and commands:
- [ISA](#instructions-and-words)



## The Register File

Registers form the inner-most storage for data in the CPU. They are used for ALU reads/writes, and address and program control.

The 16 registers are all 16 bits wide - each is general purpose, though reg1 and reg2 are hardwired to the ALU inputs. I made that choice mostly to keep the wiring simple, since dragging around the 16 bit bus wires into demuxes is a real pain to manage. The registers are each a clock enabled fip-flop, allowing outputting the current value while receiving a new input value in the same instruction.

I originally created them as plain latches, unable to receive both an input and output! But I quickly realized what a pain it would be to, for example, disallow writing an alu output to reg1 because reg1 was being used as alu input.

There is one main bus input/output to the register file. Both an output and input register can be selected to connect to the bus - selecting one of each will move the value between the registers.

There is also a separate output port and selector for the RAM address line, independent of the main bus i/o.


![register-file](/pictures/logic-world-cpu/register-file1.png)

The green panels on the bottom of the register file are decoders for:
- output register
- input register
- address out register


Having learned a bunch through this process, if I were to redo the register file today I'd probably take the RAM address output and turn it into a general purpose output port, and select RAM address out based on the command. Then I could also route this to one of the ALU inputs. But I hadn't learned about the idea of organizing with input and output ports, so I wired up what I needed to make it work.


## Program Counter and Register

The Program Counter is fairly straightforward. It's a register that directly outputs to the ROM instruction address. The program counter keeps track of what instruction should be executed next.

Each clock cycle, the current PC value runs through a +1 adder, and increments by one, reading out the next instruction.

There is also control logic to allow the following

- Output the current PC value to the bus
- Store the current value on the bus to the PC

These together help implement control flow - program jumps, and function calls.

![PC block pic](/pictures/logic-world-cpu/PC-1.png)

That's the main bus on the right side. The PC register is the two green rows of latch blocks, forming a flip-flop. 


## ALU

The ALU only has a couple of basic functions - add, subtract, add with carry bit, Bitwise AND, OR, NOT, and bit shift right. It performs operations on inputs, and stores those back into a register.

The NOT and bit shift use only the first ALU input, currently reg1. ALU outputs are dumped to the main bus.

A few status bits based on the result of the last ALU operation are stored in a small register too, those are:

- carry bit
- negative
- zero

The alu status bits can be used for conditional jumps, like a jump if result was zero.

![alu pic](/pictures/logic-world-cpu/alu-1.png)

On the left side is the 16bit adder, split over 2 levels. The components on the right side are the bitwise operations.

As you can see, there is no hardware multiplication, or barrel shifter for bit shifts. I'll probably add a shift left. But otherwise multiplication and division must be done in software.


## Instruction Decoder

As a logic component, the instruction decoder was the most complex part for me - and also why it was important to keep the Instruction Set Architecture simple and consistent. That kept the decoding manageable.

The difficulty with the decoder was keeping track of all of the various control lines for the CPU, and making sure each required control line was selected for the relevant command, without causing any interference or accidental triggers for unrelated control lines.

But this is also the component that brings it all together and turns a bunch of logic gates into a programmable machine!

![decoder pic](/pictures/logic-world-cpu/decoder-1.png)

The instruction word comes in at the top. On the right side are some of the control lines and register selectors. The bottom is the jmp decode.


I didn't want to go the microcode ROM route - since we already know ROM programming is a pain. It's also very heavy transistor/logic gate wise in Logic World. However, if I were to build build on the complexity of the CPU, and especially if it ended up as a von-nuemann machine, I'd likely go with a microcode ROM. That would make adding, or adjusting instruction much simpler.


## The ROM Block

The ROM (read only memory) holds the computer program memory. Each CPU instruction is input into the ROM before the computer starts. Then each cycle, an instruction is ready out into the instruction bus and into the instruction decoder to be executed.

![ROM Close Front](/pictures/logic-world-cpu/rom-2.png)

The ROM is built up of panels of 16, 16bit instruction words. The ROM blocks are just individual bit switches.. for every single bit. Programming the ROM means flipping each bit switch to input the required 16 bit CPU instruction, for each instruction. It is a painfully slow and error prone process. I imagine it's not too far off of the punch-card programming days. I'm sure glad we don't have to do that anymore!

The total ROM is 32 panels deep, providing 512 total words of program memory - so 512 instructions.

![ROM zoomed out](/pictures/logic-world-cpu/rom-3.png)

While this isn't much program memory, and I could technically fill the whole 16 bit address space, it's still way, way more than I'm going to program by hand!

Plus, two other things prevent me from adding more ROM:
- Placing these giant ROM blocks does a number on the framerate when in build mode.
- Making large bit decoders like 16 bits, is a ton of output lines! Those get hard to route and manage quickly.

The current address selector works with a 4 bit decoder for the individual word on a panel, a 4 bit decoder for one of the panels in a 16 bit block, and another bit to select the front or back block of 16 panels. But those large decoders mean routing a lot of individual decode lines.

One thing that took some time to get right is the timing of the instruction register - it's timed to load the instruction on the low clock pulse, so it is fully decoded and ready for execution on the next high pulse. Timing issues are tricky.


## RAM

The RAM (random access memory) is similar to the ROM, being 16 bit words, and word addressed instead of byte addressed. Keeping everything 16 bit word addressed simplifies memory access, at the cost of either bit-packing bytes into 16bit words, or less efficient storage for single byte values.

Ram stores or loads data from the main data bus. A load/store instruction selects which register to use for the memory address, and which register to use for either to store in RAM, or as the destination for loading from the given RAM address.

I went with 32 word ram planes with a 5 bit decoder to select the word per plane - and made them green because it really starts to look like a circuit board. Total is 16 planes for 512 words of ram (or 1024 bytes).

![ram](/pictures/logic-world-cpu/ram-3.png)




## Instructions and words.

Each instruction is a 16 bit instruction word. The ISA is designed to be simple, to keep the cpu architecture and instruction decoding simple. By simple, I mean that I try to keep bit positions consistent semantically, so the instruction decoder can decode the same bits for similar instructions.

In general, the top 4 bits determines the high-level instruction. Details of the opcode are determined by the rest of the bits in the instruction. There are 16 registers, so those are addressed in 4 bits.

Currently Supported Commands:

#### Noop

No operation - does nothing.

0 0 0 0 | any

#### IMM dest, #number

The immediate instruction loads a given 8 bit number into the given register.
The bit layout is:

0 0 0 1 | destination register | data bits


#### IM12 #number

The im12 loads a 12 bit number into reg0 immediately. NOTE: may change dest register in the future.

0 0 1 0 | data bits

#### mov rega(to), regb(from)

Copy a value from register b to register a.

0 0 1 1 | destination | source | unused


#### ALU Ops

All ALU ops start with the same top bits. The specific operation is defined by the bottom nibble. ALU ops can be saved to any register.

0 1 0 0 | destination | not used | alu op

This leaves 16 possible alu ops under this instruction. Only a few are used now, so there is room to expand the number of ALU ops.

Also, as mentioned the ALU inputs are hardcoded to reg1 for input1, and reg2 for input 2. For subtraction, it is always reg1 - reg2. Not and shift commands operation on reg1.

With the current ISA, it would be fairly simple to support a single variable input register to the ALU operations. Supporting 2 would likely mean some ISA redesign though.

#### ADD dest

Adds reg1 and reg2, writes to the given register.

ALU op: 0 0 0 0

#### SUB dest

Subtracts reg2 from reg1, writes to given register

ALU op: 0 0 0 1

#### ADC dest

Adds reg1 and 2 with the current value of the carry flag, output in dest.

ALU op: 0 0 1 0

#### other ALU OPS..

- AND: 0 0 1 1
- OR:  0 1 0 0
- NEG: 0 1 0 1
- SHR: 0 1 1 0


#### JMP/CMP Ops

The jmp/cmp ops are the branching operations. They take a register to use as the operation address, and jump if the condition is satisfied.

Similar to the ALU ops, there is currently a single overall jmp high bits, with the low bits specifying the operation.

0 1 0 1 | unused | address register | jump operation

A jump works by writing the contents of the given jump register to the bus, and storing them in the Program Counter, if the given condition is met. This jumps the program counter to the given instruction, starting there. Pretty neat! Jumps are also how function calls can be implemented, along with the ability to load the current PC value so you where to return to.

jmp operations:

- JMP:  0 0 0 0
- JMPZ: 0 0 0 1
- JNZ:  0 0 1 0


#### JMPI #addr

Jump immediate to a 12 bit address.

0 1 1 0 | address bits


#### Ldr dest, [addr]

This command loads data from ram at the address at the given register, and puts it into the dest register.

1 0 0 0 | dest | addr register | unused

#### str reg, [addr]

Stores the value in reg, to the ram address in the register in [addr].

1 0 0 1 | data register | address register | unused

#### LPC dest

Load the current PC+1, and store it in the given register. Note that this is the pointer to the next instruction, not the current instruction.

1 0 1 0 | dest | unused | unused


## Registers and Interface

As mentioned there are 16 registers in the CPU. Each register can be used as a general purpose register, and each is 16 bits. Here I'll describe their general use.

Most importantly, reg1 and reg2 are hardwired as ALU inputs. On a subtraction, it is reg1 - reg2.

- reg0: immediate 12 load destination
- reg0-reg6: ALU, function inputs
- reg7-reg11: callee preserved registers
- reg12-reg13: addr scratch registers
- reg14: stack pointer
- reg15: link register

The rest of the use descriptions really just form the basics of the binary interface. As I've learned more about how to program this thing in machine-code/assembly, I've learned the importance of having a stable binary interface and contract between functions. Without that, it is very easy to stomp on a callers registers and corrupt your program.


# Potential improvements

These are potential CPU/hardware improvements (rather than instruction set)

- One variable ALU input
- Add jneg/jpos for jumping on a negative/positive alu flag
- Immediate load and store commands.


# What's Next?

I've learned a ton about the lowest level of computers and programming through this project. I also wrote a simple assembler in rust, so I can write assembly and get machine code, instead of having to write out each instruction word by hand. The assembler includes jump labels so I don't need to track or calculate jump addresses by hand. I'm planning to add some macro-type functionality to the assembler as well, most usefully for function calls, which require some setup.

I've written some small test programs for the logic-world cpu, but I am running into one major slowdown - actually getting the program in the computer! Having to flip every bit by hand in the program ROM is just too tedious for any program past 20 or so instructions.

I've started translating the cpu design to a digital logic simulator called [Digital](https://github.com/hneemann/Digital). From there I'll be able to adapt the assembler to assemble to files that can be loaded into the ROM directly. Programming will be much easier.

There are still some minor improvements I'd like, in the section above. Variable ALU inputs is probably the big one to remove more shuffling around registers. Maybe immediate load/stores to simplify programs a bit.

And of course this is all hardware level - I'm looking forward to building up some complex programs in my assembly language - getting into stack management, an improved binary interface, heap management and allocators, etc. Maybe adding some memory-mapped IO devices.

Moving to a real von-neumann architecture is still something I'm planning in the longer term, but that seems like it comes with a bunch of complexity. At that point I'd probably switch the hardware decoder out for microcode ROM.

Once the CPU is in Digital, that also opens the door for writing this out to a real FPGA and running my custom CPU on real hardware, cool!

