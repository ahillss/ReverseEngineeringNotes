# Reverse Engineering Beginners Guide

Not really a guide, but a listing of each of the problems in no particular order that I had to go through when learning to reverse engineer C++ code compiled to 32-bit X86 assembly.

*Note: this is a work in progress.*

### Hex editor

For modifying the binary files. [HT](http://hte.sourceforge.net), comes with a builtin disassembler, allowing you to see the assembly representation of the hex code as you type.

When editing a file, you cannot change the size of a file, otherwise it will throw off memory offsets in the instructions.

### Assembler tool

To convert assembly instructions into hex. The only tool I am aware of is [rasm2](https://github.com/radare/radare2/wiki/Rasm2) from [radare2](https://radare.org).

It not only useful for converting basic instructions, but also instructions such as a near [jump](http://x86.renejeschke.de/html/file_module_x86_id_147.html) or [call](http://x86.renejeschke.de/html/file_module_x86_id_26.html), where the offset will need to be calculated.

For example to call a function at 0x8050e3c from address 0x8051cee:

```rasm2 -o 0x8051cee -a x86 -b 32 'call 0x8050e3c'```

to generate the hex ```e8 49 f1 ff ff```

Also make sure to double check the generated hex by viewing it in a disassembler to make sure it is correct, I believe rasm2 might have problems in certain circumstances.

### Disassembler

To convert the binary representation of the instructions back into assembly. An easy to use one is objdump from [binutils](https://www.gnu.org/software/binutils).

The assembly generated cannot just be plugged into an assembler due to that the dissassembler may add additional information to the input to help in readability, data declarations may be reversed as assembly to produce nonsense instructions and amongst other problems.

```objdump -M intel -S -D -z binary_file > dump.asm```

*  ```-M intel``` tells it to output Intel assembly
* ```-S``` ...
* ```-D``` outputs everything, useful for seeing the hex values of global variables/data
* ```-z``` doesn't strip out any uneccessary code, otherwise they are replaced with ellipses

If you are reverse engineering an executable you can look for the function name you are interested in.

### Decompiler

Useful to see the structure of the code, identitfy externed global variables from libraries or to identify class virtual tables which can be difficult to discern from the assembly only. A good one is [IDA](https://www.hex-rays.com/products/ida/).

It should be noted that the C/C++ source generated will not have completely valid syntax and will often be missing type information (except for their byte sizes).

### Learning assembly

There are two main assembly styles to choose from, Intel and AT&T. I prefer Intel as it is less cluttered, but once you have learned one you can look up the [differences](http://www.imada.sdu.dk/Courses/DM18/Litteratur/IntelnATT.htm) ([archived](http://archive.is/f1dul)) and be able to use the other.

The book [PC Assembly Language](http://pacman128.github.io/pcasm/) by Paul A. Carter is freely available.

### Debugger

For debugging problems you introduce, or for looking at the registers, stack and heap values at runtime. A good one is  [GDB](https://www.gnu.org/software/gdb).

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

### Layout of the executable or library binary

Useful to be able to get the pertinent information about your target binary.

ELF files are broken into sections, at the top is the header which has information like 16/32/64 bit, endianness, machine, etc.

Some information on ELF files:
* [Linux ELF Object File Format (and ELF Header Structure) Basics](http://www.thegeekstuff.com/2012/07/elf-object-file-format/) ([archived](http://archive.is/tk3eF))
* [The 101 of ELF Binaries on Linux: Understanding and Analysis](https://linux-audit.com/elf-binaries-on-linux-understanding-and-analysis/) ([archived](http://archive.is/EOkmi))
* [Computer Science from the Bottom Up - Chapter 8. Behind the process - ELF](http://www.bottomupcs.com/elf.xhtml) ([archived](http://archive.is/DBnia))

### Endianness

Important for when looking at the hex, as you maybe confused by the order it appears. Your binaries are most likely using little-endian as it is what Intel CPUs use.

### Stacks and Alignment

Depending on the compiler options used when the binary was compiled, the stack might have to be aligned to a certain amount of bytes. Aligning it to 16 bytes is usually best, but you can figured it out by looking for any stack memory that was declared (as padding) and cleaned up without being used.

Also a function call will push the return address onto the stack, you will need to remember to count that as well.

Another thing you might see after an a ```add esp,0x8``` is ```add esp,0xfffffff8```. This is just using the unsigned integer overflow where it wraps around, it is the same as ```sub esp,0x8```.

### Executable start address

When disassembling executables, the start address starts at```0x8048000```. Some disassemblers will start at ```0x0``` and others correctly (like **objdump**) at ```0x8048000```.

Hex editors all seem to start at ```0x0```, so to use the *goto address* feature you will need to convert the addresses in **objdump** by subtracting them with ```0x8048000```.

I believe shared libraries start from a different address.

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
8051b12:	81 c3 03 36 03 00    	add    ebx,0x33603 ;add constant to ebx to get the absolute address for global data ...
...
805217b:	8b 83 2c 02 00 00    	mov    eax,DWORD PTR [ebx+0x22c] ;a global variable
 -...
8051b1b:	8d 83 4a bc ff ff    	lea    eax,[ebx-0x43b6]	;a string constant
...
80527de:	5b                   	pop    ebx
80527df:	89 ec                	mov    esp,ebp
80527e1:	5d                   	pop    ebp
80527e2:	c3                   	ret    
```

* ```[ebx-value]``` seem to refer to constants and ```[ebx+value]``` seem to refer to global variables, I don't know if this is the same everywhere all the time though.
* the line ```call 8051b11``` pushes the ```eip``` value on to the stack and calls the next line ```pop ebx``` where it is popped off into the ebx register.

Also note that decompiled source code (from **ida**) will often use a ```global``` variable. Which doesn't actually refer to the pointer value stored in```ebx``` as you would assume, but rather ```ebx + offset```. I do not know how the offset is calculated or what the significance of it is.

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

;reclaim 24 bytes from sub, 8 bytes from pushes
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

;reclaim 24 bytes from sub, 8 bytes from pushes
add    esp,0x20

;4 bytes to store myfunc address
sub    esp,0x4
mov    DWORD [ebp-0x4],eax
push   DWORD [ebp+0xc] ;argv
push   DWORD [ebp+0x8] ;argc
push   ebx ;useful for accessing global variables/constants/etc

mov    eax,DWORD [ebp-0x4] ;myfunc

;myfunc(ebx,argc,argv)
call   eax

;reclaim 4 bytes from sub, 12 bytes from pushes
add    esp,0x10
```
You will need to make room for this code in the binary. One strategy is to overwrtie a section of code that is easy to replicate in your shared library. The second strategy is to overwrite a section of code that won't be missed like  *command line options* handling code, while hard coding in any options you need to use in the binary or your shared library.

### C++
Some references
* [Inside the C++ Object Model 1st Edition](https://www.amazon.com/Inside-Object-Model-Stanley-Lippman/dp/0201834545) by Stanley B. Lippman
* [Reversing C++ Virtual Functions: Part 1](https://alschwalm.com/blog/static/2016/12/17/reversing-c-virtual-functions/) ([archived](http://archive.is/ezxOe))
* [Reversing C++ Virtual Functions: Part 2](https://alschwalm.com/blog/static/2017/01/24/reversing-c-virtual-functions-part-2-2/) ([archived](http://archive.is/T9wsl))

#### Classes

#### Temporary Objects

#### Exceptions


