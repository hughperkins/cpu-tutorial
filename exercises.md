1. build simulatable verilog, that will read some arbitrary 16-bit hexadecimal numbers from a text file, and output them onto the screen (using $display for output, you can use $readmemh to load the file)
2. create a module 'proc' that will output the numbers 1 to 10, changing the number output on each clock positive edge. Create a driver module, which will provide a clock and reset to the module; and will use $display to print the numbers output by the module.
3. modify the modules from 2. so that:
    - the driver module loads the hexadecimal file from 1., into a memory, and provides this memory to `proc`
    - `proc` iterates over the memory contents, giving each value to the driver module, one at a time
    - the driver module uses $display to print out each value
4. split the hexadecimal file into two sections:
    - first numbers contain instructions, which we will talk about in a second
    - second set of numbers are the numbers we want to print out, just as before
    - for now, create a single 16-bit instruction, which we'll denote as `OUTLOC`, which will contain a memory location, and will send the contents of that memory location to the driver module
    - you'll have to find a way to encode the memory location inside the instruction
        - for example, the last 8 bits of the instruction could represent the instruction type, for example you could use `1` to mean `OUTLOC`
        - and the first 8 bits of the instruction could represent the location in memory to output
    - run the driver, and check the outputs are ok
