# Reverse Engineering Beginners Guide

A beginners guide for reverse engineering C/C++ code compiled to X86 assembly. All assembly examples will be X86 32-bit.

This is a work in progress.

### A hex editor

To modify the binary files. [HT](http://hte.sourceforge.net), comes with a builtin disassembler, allowing you to see the assembly representation of the hex code as you type.

When editing a file, you cannot change the size of a file, otherwise it will throw memory offsets used by the instructions.

### An assembler tool

To convert assembly instructions into hex. The only tool I am aware of is [rasm2](https://github.com/radare/radare2/wiki/Rasm2) from [radare2](https://radare.org).

It not only useful for converting basic instructions, but also instructions such as a near [jump](http://x86.renejeschke.de/html/file_module_x86_id_147.html) or [call](http://x86.renejeschke.de/html/file_module_x86_id_26.html), where the offset will need to be calculated.

For example to call a function at 0x8050e3c from address 0x8051cee:

```rasm2 -o 0x8051cee -a x86 -b 32 'call 0x8050e3c'```

to generate the hex ```e8 49 f1 ff ff```

Also make sure to double check the generated hex by viewing it in a disassembler to make sure it is correct, I believe rasm2 might have problems in certain circumstances.

### A disassembler

Is needed to convert the binary representation of the instructions back into assembly. An easy to use one is objdump from [binutils](https://www.gnu.org/software/binutils).

The assembly generated cannot just be plugged into an assembler due to that the dissassembler may add additional information to the input to help in readability, data declarations may be reversed as assembly to produce nonsense instructions and amongst other problems.

```objdump -M intel -S -D -z binary_file > dump.asm```

*  ```-M intel``` tells it to output Intel assembly
* ```-S``` ...
* ```-D``` outputs everything, useful for seeing the hex values of global variables/data
* ```-z``` doesn't strip out any *uneccessary* code, otherwise they are replaced with ellipses

If you are reverse engineering an executable you can look for the function name you are interested in.

### A decompiler

Useful to see the structure of the code, identitfy externed global variables from libraries or to identify class virtual tables which can be difficult to discern from the assembly only. A good one is [IDA](https://www.hex-rays.com/products/ida/).

It should be noted that the C/C++ source generated will not have completely valid syntax and will often be missing type information (except for their byte sizes).

### Learning assembly

There are two main assembly styles to choose from, Intel and AT&T. Intel is less cluttered, but you learn one and are aware of the [differences](http://archive.is/f1dul), switching between them is easy enough.

Thw book *PC Assembly Language by Paul A. Carter* is freely available [here](http://pacman128.github.io/pcasm).

### A debugger

For debugging problems you introduce, and also for looking at the registers, stack and heap values at runtime, a good one is  [GDB](https://www.gnu.org/software/gdb).

GDB uses the AT&T syntax, some useful commands are:
* ```run``` - to start the program
* ```break *location``` - break at the location a hex value
* ```break function_name``` - break when the function with passed name is called
* ```continue``` - continue after a break
* ```info register``` - view register values
* ```x/x $esp``` - view stack values
* ```info frame``` - get frame info
* ```frame 0``` - change frame to the integer provided
* ```bt``` - a stack trace

### Understand the layout of the executable or library binary

To be able to get the pertinent information about your target binary.

ELF files are broken into sections, at the top is the header which has information like 16/32/64 bit, endianness, machine, etc. Some information on ELF files are [here](http://archive.is/wJW5i), [here](http://archive.is/JyChY) and [here](http://archive.is/DBnia).

### Know of endianness

As you maybe confused when looking at the order of the hex code. Your binaries are most likely using little-endian as it is what Intel CPUs use.

### Stack alignment

Depending on the compiler options used when the binary was compiled, the stack might have to be aligned to a certain amount of bytes. Aligning it to 16 bytes is usually best, but it can be figured out by looking at any padding (declared but unused) used.

Also a function call will push the return address onto the stack, you will need to remember to count that as well.

Another stack related thing you might see after an a ```add esp,0x8``` is ```add esp,0xfffffff8```. This is just using the unsigned integer overflow where it wraps around, it is the same as ```sub esp,0x8```.

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

The easiest way to reverse engineer a binary is to replicate the code bit by bit (usually starting with the main function) in your own shared library. Then load the shared library and call your code from the binary file at runtime.

The [OpenRCT](https://openrct2.org/) project [used](http://archive.is/SDuL0) a program called [CFF Explorer](http://www.ntcore.com/exsuite.php) to load their own DLL. But I am unaware of a similar project in Linux of which I am using, so I will show you how to modify the binary the binary to load your own shared library and call a function from it. I will be using the ```dlopen``` and ```dlsym``` functions, which your binary will need to have available (accessible from the executable). There is probably a way to load them if they are not there, but for now I am yet to figure out how.

The equivalent code in C will look like this:
```C
int main(int argc, char *argv[]) {
    //...

    void *lib=dlopen("./libmy.so",RTLD_LAZY);
    void (*func)(char*,int,char**)= dlsym(lib,"myfunc");
    func(global,argc,argv);

    //...
}
```
The same code in assembly, but with a few minor tweaks:

```asm
; 12 bytes for string, 12 bytes for padding
sub    esp,0x18 

; push null terminated string "./libmy.so" onto the stack
mov    DWORD [ebp-0xc],0x696c2f2e ;;il/.
mov    DWORD [ebp-0x8],0x2e796d62 ;;.ymb
mov    DWORD [ebp-0x4],0x6f73 ;;\0\0os

push   0x1 ;;RTLD_LAZY
lea    eax,[ebp-0xc]
push   eax
call   dlopen
add    esp,0x20

; 4 bytes for storing lib address returned from dlopen,
; 8 bytes for string and 12 bytes for padding
sub    esp,0x18 

;store lib
mov    DWORD [ebp-0xc],eax

;push null terminated string "myfunc" onto the stack
mov    DWORD [ebp-0x8],0x7566796d ;;ufym
mov    DWORD [ebp-0x4],0x636e ;;\0\0cn
lea    eax,[ebp-0x8]
push   eax

;use lib as param
push   DWORD [ebp-0xc]

;dlsym(lib,"myfunc");
call   dlsym
add    esp,0x20

sub    esp,0x4
mov    DWORD [ebp-0x4],eax
mov    eax,DWORD [ebp+0xc] ;argv
push   eax
mov    eax,DWORD [ebp+0x8] ;argc
push   eax
mov    eax,DWORD [ebx+0x20a] ;global
push   eax

mov    eax,DWORD [ebp-0x4] ;myfunc

;myfunc(global,argc,argv)
call   eax

add    esp,0x10
```
