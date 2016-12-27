# Reverse Engineering Beginners Guide

A beginners guide for reverse engineering C/C++ code compiled to X86 assembly.

This is a work in progress.

### A hex editor

To modify the binary files. I use [HT](http://hte.sourceforge.net), which also comes with a builtin disassembler, allowing you to see the assembly representation of the hex code as you type.

When editing a file, you cannot change the size of a file, otherwise it will throw memory offsets used by the instructions.

### An assembler tool

To convert assembly instructions into hex. I use [rasm2](https://github.com/radare/radare2/wiki/Rasm2) from [radare2](https://radare.org).

It not only useful for converting basic instructions, but also instructions such as a near [jmp](http://x86.renejeschke.de/html/file_module_x86_id_147.html) or [call](http://x86.renejeschke.de/html/file_module_x86_id_26.html), where the offset will need to be calculated.

For example to call a function at 0x8050e3c from address 0x8051cee:

```rasm2 -o 0x8051cee -a x86 -b 32 'call 0x8050e3c'```

to generate the hex ```e849f1ffff```

### A disassembler

Is needed to convert the binary representation of the instructions back into assembly. I use objdump from [binutils](https://www.gnu.org/software/binutils).

The assembly generated cannot just be plugged into an assembler due to that the dissassembler may add additional information to the input to help in readability, data declarations may be reversed as assembly to produce nonsense instructions and amongst other problems.


```objdump -M intel -S -D -z binary_file > dump.asm```

*  ```-M intel``` tells it to output Intel assembly
* ```-S``` ...
* ```-D``` outputs everything, useful for seeing the hex values of global variables/data
* ```-z``` doesn't strip out any *uneccessary* code, otherwise they are replaced with ellipses

If you are reverse engineering an executable you can look for the function name you are interested in.

### A decompiler

Useful to see the structure of the code, identitfy externed global variables from libraries or to identify class virtual tables which can be difficult to discern from the assembly only. I've used [IDA](https://www.hex-rays.com/products/ida/).

It should be noted that the C/C++ source generated will not have completely valid syntax and will often be missing type information (except for their byte sizes).

### Learning assembly

There are two main assembly styles to choose from, Intel and AT&T. I chose Intel because it looks less cluttered, but once you know one, and are aware of the [differences](http://archive.is/f1dul), switching between them is easy enough.

A free book *PC Assembly Language by Paul A. Carter* is freely available [here](http://pacman128.github.io/pcasm).

### A debugger

For debugging problems you introduce, and also for looking at the registers, stack and heap values at runtime, I use [GDB](https://www.gnu.org/software/gdb).

GDB uses the AT&T syntax, some useful commands are:
* ```run``` - to start the program
* ```break *location```
* ```break function_name```
* ```continue``` - continue after a break
* ```info register``` - view register values
* ```x/x $esp``` - view stack values

### Understand the layout of the executable or library binary

To understand what the tools you are using are doing or how they know things, but this is not highly important. 

ELF files are broken into sections, first is the header which has information like 16/32/64 bit, endianness, machine, etc. Some information on ELF files are [here](http://archive.is/wJW5i), [here](http://archive.is/JyChY) and [here](http://archive.is/DBnia).

### Know of endianness

As you maybe confused when looking at the order of the hex code. Your binaries are most likely using little-endian as it is what Intel CPUs use.

[WIKI article](https://en.wikipedia.org/wiki/Endianness).

### Stack alignment

Depending on the compiler options when the binary was compiled, the stack might have to be aligned to a certain amount of bytes. Aligning it to 16 bytes is usually best.

A function call will push the return address onto the stack, you will need to remember to count that as well.

Also another stack related thing you might see after an a ```add esp,0x8``` is ```add esp,0xfffffff8```. This is just using the unsigned integer overflow where it wraps around, it is the same as ```sub esp,0x8```.

### Global variables

Not only global variables, but static variables and string constants are also considered part of the global variables.

The register ebx is usually used to store the address to it.

```asm
8051b00:	55                   	push   ebp
8051b01:	89 e5                	mov    ebp,esp
8051b03:	81 ec 5c 30 00 00    	sub    esp,0x305c
...
8051b0b:	53                   	push   ebx
8051b0c:	e8 00 00 00 00       	call   8051b11
8051b11:	5b                   	pop    ebx ;get current address
8051b12:	81 c3 03 36 03 00    	add    ebx,0x33603 ;add constant to ebx to the global address
...
8051b1b:	8d 83 4a bc ff ff    	lea    eax,[ebx-0x43b6]	;string constant
...
805217b:	8b 83 2c 02 00 00    	mov    eax,DWORD PTR [ebx+0x22c] ;global variable
...
80527de:	5b                   	pop    ebx
80527df:	89 ec                	mov    esp,ebp
80527e1:	5d                   	pop    ebp
80527e2:	c3                   	ret    
```

* ```[ebx-value]``` seem to refer to constants and ```[ebx+value]``` seem to refer to global variables, I don't know if this is the same everywhere all the time though.
* the line ```call 8051b11``` pushes the ```eip``` value on to the stack and calls the next line ```pop ebx``` where it is popped off into the ebx register.

### Inserting instructions
