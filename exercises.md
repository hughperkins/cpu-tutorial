1. build simulatable verilog, that will read some arbitrary 16-bit hexadecimal numbers from a text file, and output them onto the screen (using $display for output, you can use $readmemh to load the file)
2. create a module 'proc' that will output the numbers 1 to 10, changing the number output on each clock positive edge. Create a driver module, which will provide a clock and reset to the module; and will use $display to print the numbers output by the module.
3. modify the modules from 2. so that:
    - the driver module loads the hexadecimal file from 1., into a memory, and provides this memory to `proc`
    - `proc` iterates over the memory contents, giving each value to the driver module, one at a time
    - the driver module uses $display to print out each value
    - it's possible to create the memory in driver module, and then send the entire memory into the proc, using iverilog (this won't synthesize with yosys, but we can think about that later)
4. split the hexadecimal file into two sections:
    - first numbers contain instructions, which we will talk about in a second
    - second set of numbers are the numbers we want to print out, just as before
    - for now, create a single 16-bit instruction, which we'll denote as `OUTLOC`, which will contain a memory location, and will send the contents of that memory location to the driver module
    - you'll have to find a way to encode the memory location inside the instruction
        - for example, the last 8 bits of the instruction could represent the instruction type, for example you could use `1` to mean `OUTLOC`
        - and the first 8 bits of the instruction could represent the location in memory to output
    - run the driver, and check the outputs are ok
5. since creating the instructions is kind of tedious, create a python script (or C++, or whatever language you like), that will take a text file with assembly code, and convert it into the hexadecimal instructions. The assembly code can look something like the following:
```
outloc 64
outloc 68
outloc 72
outloc 76

location 64:
    abcd
    1234
    dead
    beef
```
    - check that you can run your assembler to produce machine code, and then run your verilog simulation, to run the machine code, and the outputs look correct
6. __registers__
    - Modify your proc module so that it has a memory to store 32 registers, which we will denote x0 to x31.
    - add a new instruction `LI`, which will load a number into a register
        - for example `li x1, 123` will load the number 123 into register x1
    - add an instruction `outr` which will print out the value of a register, eg `outr x1` will print out the value of x1
    - create some assembly code to test these two instruction, assemble this assembly, and run it
    - check the output looks ok
7. __memory__
   - move the memory out of proc/driver modules, into a new file, e.g. `mem.sv`
      - you will need to design an appropriate protocol for `mem.sv`, to allow reads and writes by `proc.sv`
      - you will need to design an appropriate protocl so that the driver module can write the initial hexadecimal instructions and data into the memory
      - easiest way might be to create a second 'write port' into `mem.sv`, that only the driver module uses
      - assembler and run your assembler programs, and check they continue to run ok
      - you will need to somehow handle that reading in instructions from memory will now take more than a single clock cycle
          - at this point, you will probalby want to start making proc.sv be a finite state machine
          - (if you've no idea how to start with this, you could try some of the FSM problems in hdlbits, if you haven't already; and make sure you completed the 'game of life' problem, if you didn't already)
8. add `load` and `store` instructions, that will load the value of a register from a memory location, or store the value of a register into a memory location, respectively. e.g. `load x1 64`, will load the value from memory location 64 into register x1; and `store x2 68` will store the value from register `x2` into memory location `68`.
    - you can add a new instruction `outloc`, if you wish, that outputs the value at a particular memory location, e.g. `outloc 64` will send the value at location 64 to the driver for output
    - both load and store will likely need multiple clock cycles, so you will need to continue working with the finite state machine you created in step 7
9. __RISCV__
    - migrate your instructions to be RISC-V compliant
    - don't modify `out` or `outloc` or `outr` for now, since these are not RISC-V instructions
    - you'll need to change your instructions to be 32-bit
    - you'll need to implement `li` as a pseudoinstruction, that does something like `addi x5, x0, 123`, which takes the value of register `x0` (always 0), adds 123, and places the result into register x5
    - you'll need to look up the binary representations used in RISC-V in the RISC-V volume 1, unprivileged spec, https://github.com/riscv/riscv-isa-manual/releases/download/Ratified-IMAFDQC/riscv-spec-20191213.pdf
10. add arithmetic, like `add`, `sub`, `mul`, using the simple verilog operators `+` and `-`, `*`
    - write assembly to test this
    - run, and check output is ok
11. build an assembly program to add the numbers 1 to 5
    - you'll need to add at least one branch instruction to do this
    - this means you'll need to add the ability to create labels to your assembler
        - eg something like `somelabel:` is a label called `somelabel`
    - for now, you can simply allow backwards jumps only, which simplifies your assembler, since you only need a single pass
12. __delay propagation__ at some point you need to start synthesizing your design, and doing things like:
    - gate-level simulation
    - measuring propagation delay
    - measuring die area
    - let's start with measuring propagation delay
    - there are various repos around which say they can do this (e.g. [OpenTimer](https://github.com/OpenTimer/OpenTimer)), however I didn't have much/any success in using them, so I built my own script, [https://github.com/hughperkins/VeriGPU/blob/main/verigpu/timing.py](https://github.com/hughperkins/VeriGPU/blob/main/verigpu/timing.py)
    - you are free to measure propagation delay however you want, but I feel you do need to start measuring it :)
    - obviously, my own opinion is that using my own script is the easiest way, but your mileage may vary :)
    - the way my script works is:
         - first use yosys to synthesize your verilog down to gate-level cells, using the OSU018 tech, [https://github.com/hughperkins/VeriGPU/tree/main/tech/osu018](https://github.com/hughperkins/VeriGPU/tree/main/tech/osu018)
         - assign a relative propagation delay, relative to that of a single NAND gate, to each node in the tree, based loosely on the propagation delays in [https://web.engr.oregonstate.edu/~traylor/ece474/reading/SAED_Cell_Lib_Rev1_4_20_1.pdf](https://web.engr.oregonstate.edu/~traylor/ece474/reading/SAED_Cell_Lib_Rev1_4_20_1.pdf)
         - walk the graph, finding the longest path between flip-flops, outputs and inputs, as a sum of the propagation delays of the walked nodes
    - in any case, measure somehow the propagation delay of your `proc.sv` module
13. __div__ create a division module, that will divide two integers, returning the result and the remainder
   - note that using the verilog `/` operator will result in the division running in a single cycle
   - this is *very* slow: high propagation delay. you can measure using whatever approach you settled on in 12
   - so you will likely want to make the division run over multiple cycles somehow (e.g. 32 cycles, one for each bit, for example; whatever it takes to keep the propagation delay short)
14. __divu__, __remu__ : use the module from the previous step to implement `divu` and `remu` instructions in `proc.sv`
15. create an assembler program that outputs the first few prime numbers
    - you can do this two ways: sieve of aristothenes (I can never spell this...), or iterating over each integer, and checking for factors
    - the first way doesn't need either `divu` or `remu`, so try the second way, even if you do the sieve of aristothenes too
16. add float support. I used `zfinx`, i.e. using same registers for both floats and integers, but you could use the more standard `F` extension of RISC-V
    - ensure that at least the following work for now:
         - `li x1, 1.23`
         - `outr.s x1`  output x1 as a float
          - load and store for floats (either using `lw` and `sw` if using zfinx, or using `flw.s` and `fsw.s` if using `f`)
17. add add for floats
18. add subtract for floats
19. add multiplication for floats
20. add division for floats
21. write a matrix multiplication assembler program
    - this gets pretty fiddly
22. since writing assembler for the matrix multiplication was getting fiddly, let's migrate to use a compiler to be able to convert e.g. C/C++ programs into assembler, which we can then assemble and run
    - you can use the `clang+` compiler provided with llvm
    - if you use the options `-S -target riscv32`, then `clang+` will convert your C++ into RISC-V assembler
    - a few complications:
        - what you'll need to write in the C or C++ is a function
        - you'll need to add additional 'header' assembler in front of the resulting assembler to jump into this function
        - then halt afterwards
        - you'll need to therefore implement jump instructions (possiblye a `halt` instruction, if you don't already have one)
        - you'll need to be able to handle forward jumps, so you'll need to make your assembler handle this somehow (e.g. using two passes)
        - you'll need to load any parameters into the registers `a0`, `a1`, etc
        - you'll need to make `a0`, `a1` etc be aliases to the appropriate risc-v `x` registers
        - you'll also need to add in other aliases, such as `sp`
        - you'll need to point `sp` towards the top of some unused memory before jumping into the function
        - if you want to output anything, you'll need to make your code call a function `void out(int value);`
            - and you'll need to implement this in assembler, as part of the 'header' assembler
23. at this point, you could also consider migrating from using your own assembler, to using the clang/llvm assembler, e.g. `llc`
24. once you've got to this point, you can already run fairly complex programs
25. your next steps will be things like:
    - adding instruction caching, and memory caching in general
    - adding parallel instruction execution

(Note: you can see my own processor at https://github.com/hughperkins/VeriGPU , which I basically wrote by doing approximately what I've outlined above :) (I did a few GPU-specific things too; but I'm making this a CPU tutorial, not a GPU tutorial; happy to extend this for GPU if interest))
