# Reverse Engineering Guide

A guide for reverse engineering C/C++ code compiled to X86 assembly.

This is a work in progress.

## Modifying a binary

### A hex editor

Will be necessary to modify the binary files. I use [HT](http://hte.sourceforge.net), which also comes with a builtin disassembler, allowing you to see the assembly representation of the hex code as you type.

When editing a file, you cannot change the size of a file, otherwise it will throw memory offsets used by the instructions.

### An assembler tool

Will be needed to convert assembly instructions into hex. I use [rasm2](https://github.com/radare/radare2/wiki/Rasm2) from [radare2](https://radare.org).

It not only useful for converting basic instructions, but also instructions such as a near [jmp](http://x86.renejeschke.de/html/file_module_x86_id_147.html) or [call](http://x86.renejeschke.de/html/file_module_x86_id_26.html), where the offset will need to be calculated.

For example to call a function at 0x8050e3c from address 0x8051cee:

```rasm2 -o 0x8051cee -a x86 -b 32 'call 0x8050e3c'```

to generate the hex ```e849f1ffff```

#### A disassembler

Is needed to convert the binary representation of the instructions back into assembly. I use objdump from [binutils](https://www.gnu.org/software/binutils).

The assembly generated cannot just be plugged into an assembler due to that the dissassembler may add additional information to the input to help in readability, data declarations may be reversed as assembly to produce nonsense instructions and amongst other problems.


```objdump -M intel -S -D -z binary_file > dump.asm```

*  ```-M intel``` tells it to output Intel assembly
* ```-S``` ...
* ```-D``` outputs everything, useful for seeing the hex values of global variables/data
* ```-z``` doesn't strip out any *uneccessary* code, otherwise they are replaced with ellipses

If you are reverse engineering an executable you can look for the function name you are interested in.

Depending on the compiler, version and options used, the code may look something like this:
```asm
8051600:	55                   	push   ebp
8051601:	89 e5                	mov    ebp,esp
8051603:	83 ec 10             	sub    esp,0x10
8051606:	56                   	push   esi
8051607:	53                   	push   ebx
8051608:	e8 00 00 00 00       	call   805160d <_start@@Base+0xdd>
805160d:	5b                   	pop    ebx
805160e:	81 c3 07 3b 03 00    	add    ebx,0x33b07
 
...

8051af8:	5b                   	pop    ebx
8051af9:	5e                   	pop    esi
8051afa:	5f                   	pop    edi
8051afb:	89 ec                	mov    esp,ebp
8051afd:	5d                   	pop    ebp
8051afe:	c3                   	ret    
```

### A decompiler

Can be useful to see the structure of the code, identitfy externed global variables from libraries or to identify class virtual tables which can be difficult to discern from the assembly only. I've used [IDA](https://www.hex-rays.com/products/ida/).

It should be noted that the C/C++ source generated will not have completely valid syntax and will often be missing type information (except for their byte sizes).

### Learning assembly

Will be necessary, there are two main assembly styles to choose from, Intel and AT&T. I chose Intel because it looks less cluttered, but once you know one, and are aware of the [differences](http://archive.is/f1dul), switching between them is easy enough.

A free book *PC Assembly Language by Paul A. Carter* is freely available [here](http://pacman128.github.io/pcasm).

### A debugger

Is useful for debugging problems you introduce, and also for looking at the registers, stack and heap values at runtime, I use [gdb](https://www.gnu.org/software/gdb).

### Understand the layout of the executable or library binary

So you can understand what the tools you are using are doing or how they know things, but this is not highly important. Some information on ELF files are here:

* [[1]](http://archive.is/wJW5i)
* [[2]](http://archive.is/JyChY)
* [[3]](http://archive.is/DBnia)

### Know of endianness

As you maybe confused when looking at the order of the hex code. Your binaries are most likely using little-endian as it is what Intel CPUs use.

[WIKI article](https://en.wikipedia.org/wiki/Endianness).

