# Reverse Engineering Guide

Not really a guide, but a listing of each of the problems in no particular order that I had to go through when learning to reverse engineer C++ code compiled to 32-bit X86 assembly.

*Note: this is a work in progress.*

## (Hex) Editors

To modify the binary files you will need a hex editor.

Note when modifying binary files, you cannot move around blocks of instructions, as it will throw off memory offsets used in the instructions.

### [HT](https://github.com/sebastianbiallas/ht)

Comes with a builtin disassembler where you can see the assembly representation of the hex code as you type. It also allows you to view the executable/library file headers (though the virtual address you get from the header part seems to be off compared to what is outputted from other tools).

Note like many other disassemblers, HT's one also runs on the whole file, disassembling not only the code, but also incorrectly on the header, data etc as well so you will need to jump to the code section you intend to modify/view.

### [Cutter](https://github.com/radareorg/cutter)

Provides a GUI interface to [Radare2](https://radare.org). Shows the disassembly but also allows you to edit those instructions and will write the hex changes for you. 

One draw back is that it doesn't seem to want to let you see the hex for non code parts of the file. Also the hex codes aren't side by side the dissasembly which makes it slightly harder to see what you are doing when making changes, though you can bring up a second window to show the hex.

### [HxD](https://mh-nexus.de/en/hxd/)

For windows only, it has a neat side bar showing various decoding of any selected hex as int16/32/64, float, disassembly16/32/64 etc.

## Disassembler

To convert the binary representation of the instructions back into assembly. 

### objdump from [binutils](https://www.gnu.org/software/binutils) 

```objdump -M intel -S -D -z binary_file > dump.asm```

Note for the PE format it fails to retrieve any symbols (e.g. function names etc)

### radare2

```radare2 -q -e scr.color=false -e asm.cmt.right=true -c 'b $SS ; pD $SS@$S' binary_file > dump.asm```

Note not sure this is the correct usage, seems to dump the ```.text``` section.

## Decompiler

Reading vast amounts of code from the disassembly can be difficult, a decompiler can make that job much easier, especially for following control flow. It can also be useful for identifying global variables and the virtual tables of classes. A good one is [IDA](https://www.hex-rays.com/products/ida/).

The C/C++ source generated will not have completely valid syntax (as extra information is inserted) and will often be missing type information (except for their byte sizes).

## Learning assembly

There are two main assembly styles to choose from, Intel and AT&T. I prefer Intel as it is less cluttered, but once you have learned one you can look up the [differences](http://www.imada.sdu.dk/Courses/DM18/Litteratur/IntelnATT.htm) ([archived](http://archive.is/f1dul)) and have no trouble using the other.

For Intel the book [PC Assembly Language](http://pacman128.github.io/pcasm/) by Paul A. Carter is freely available.

## Debugger

For debugging problems you may introduce, or for looking at the registers, stack and heap values at runtime. A good one is  [GDB](https://www.gnu.org/software/gdb).

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

## Generating hex for instructions

To modify a binary using a hex editor, you will need to know the hex codes for each instruction. This can be done using [rasm2](https://github.com/radare/radare2/wiki/Rasm2) from the [Radare2](https://radare.org) collection of disassembly tools.

For example to call a function at ```0x8050e3c``` from address ```0x8051cee``` you would use ```rasm2 -o 0x8051cee -a x86 -b 32 'call 0x8050e3c'``` to generate the hex ```e8 49 f1 ff ff```.

## Layout of executables and libraries

Binary executable and library formats tend to contain a series of headers detailing information like versions, 16/32/64 bit, endianness, machine, linked libraries, code/data section locations, virtual memory offsets, etc. After the headers or (maybe) in between are code/data section and other data like resources etc.

Information about the headers can be found using [rabin2](https://radare.gitbooks.io/radare2book/content/rabin2/intro.html) (e.g. ```rabin2 -H binary```).

Typically used on Linux is the **Executable** and **Linkable** file **Format (ELF)** and on Windows the **Portable Executable (PE)** file format.

### PE

* [The Portable Executable File Format from Top to Bottom](http://www.csn.ul.ie/~caolan/pub/winresdump/winresdump/doc/pefile2.html) [(archived)](https://archive.is/uwMRp)
* [x86 Disassembly/Windows Executable Files](https://en.wikibooks.org/wiki/X86_Disassembly/Windows_Executable_Files) [(archived)](https://archive.is/Awq16)
* [Peering Inside the PE: A Tour of the Win32 Portable Executable File Format](https://msdn.microsoft.com/en-us/library/ms809762.aspx) [(archived)](https://archive.is/Q2mx4)

### ELF

* [ELF Hello World Tutorial](http://www.cirosantilli.com/elf-hello-world) [(archived)](https://archive.is/Dfzk6)

## Endianness

Important for when looking at the hex, as you maybe confused by the order it appears. Your binaries are most likely using little-endian (what Intel CPUs use).

For example in little-endian the integer ```54233456``` show as hex here ```0x33b8970``` will be stored as 4 bytes in this order ```0x70 0x89 0x3b 0x03```.

## Stacks and Alignment

Depending on the compiler options used when the binary was compiled, the stack may have to be aligned to a certain amount of bytes. Aligning it to 16 bytes is usually best, but you can figured it out by looking for padding (any stack memory that was declared and cleaned up without being used).

Be aware that function calls will push the return address onto the stack, you will need to remember to count them as well.

Another thing you might see is the stack being modfied like ```add esp,0xfffffff8```, this is just using the unsigned integer overflow where it wraps around, it is the same as ```sub esp,0x8```.

## Virtual addresses

If your disassembler outputs the instruction memory addresses (like objdump), then they will either use the physical address (starting at ```0x0```), or the virtual address (the locations of instructions at runtime).

As hex editors usually use the physically locations, then you will have have to convert from virtual addresses to physical addresses to make modifications:

```virtual_address - section_virtual_address + section_physical_address```

To get the ```section virtual address``` and ```section physical address``` you can use ```rabin2 -S binaryfile```.

## Global variables

Not only global variables, but static variables and string constants are also part of the global variables.

The register ebx is usually used as an offset to them.

```asm
8051b00:	55                   	push   ebp
8051b01:	89 e5                	mov    ebp,esp
8051b03:	81 ec 5c 30 00 00    	sub    esp,0x305c
...
8051b0b:	53                   	push   ebx
8051b0c:	e8 00 00 00 00       	call   8051b11
8051b11:	5b                   	pop    ebx ;get current address
8051b12:	81 c3 03 36 03 00    	add    ebx,0x33603 ;add constant to get address to global data
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

* ```[ebx-value]``` seem to refer to constants and ```[ebx+value]``` to global variables, I don't know if this is the same everywhere all the time though.
* the line ```call 8051b11``` pushes the ```eip``` value on to the stack and calls the next line ```pop ebx``` where it is popped off into the ebx register.

Also note that decompiled source code (from **ida**) will often use a ```global``` variable. Which doesn't actually refer to the pointer value stored in```ebx``` as you would assume, but rather ```ebx + offset```. I do not know how the offset is calculated or what the significance of it is.

Some resources:
* [Position Independent Code on i386](http://archive.is/GZAvx) ([archived](https://web.archive.org/web/20160315011254/http://www.greyhat.ch/lab/downloads/pic.html))
* [What is the Symbol Table and What is the Global Offset Table?](http://grantcurell.com/2015/09/21/what-is-the-symbol-table-and-what-is-the-global-offset-table/) ([archived](http://archive.is/Rrmm3))

## Inserting instructions

The easiest way to reverse engineer a binary is to replicate the code bit by bit (usually starting with the main function) in your own shared library. You then load the shared library from the binary.

### Finding the main function

You can either search for "main" within the disassembled executable, or use ```rabin2 -M exefile``` (to get the physical and virtual addresses). Any tool that displays the header information of an executable should be able help you find the executing starting point of the code.

### Adding another code section

I've read that you can modify the ELF/PE header to add another code section to the binary file and then add a jump instruction from the main function to it. 

[CFF Explorer](http://www.ntcore.com/exsuite.php) supposed to be able do the PE format (I still haven't got it working).

[Radare2](https://radare.org/) should be able to do it for both the PE and ELF formats.

### Shared Library example

What I did was make some room for my code by NOP-ing out unimportant code that was easy to replicated in my shared library and then inserting my own code to load that shared library and call a function from it.

I used the ```dlopen``` and ```dlsym``` functions, which your binary will need to have available (accessible from the executable). There is probably a way to load them if they are not there, but I do not know how.

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

For ```call dlopen``` and ```call dlsym``` you will have to use their addresses as specified in the binary. You can search for their names in the disassembled binary. e.g. from objdump:

```asm
Disassembly of section .plt:

08050e3c <dlsym@plt>:
8050e3c:	ff 25 80 51 08 08    	jmp    DWORD PTR ds:0x8085180
8050e42:	68 c0 00 00 00       	push   0xc0
8050e47:	e9 60 fe ff ff       	jmp    8050cac <_init@@Base+0x30>
 
0805150c <dlopen@plt>:
805150c:	ff 25 34 53 08 08    	jmp    DWORD PTR ds:0x8085334
8051512:	68 28 04 00 00       	push   0x428
8051517:	e9 90 f7 ff ff       	jmp    8050cac <_init@@Base+0x30>

```

## C++ References

* [Inside the C++ Object Model 1st Edition](https://www.amazon.com/Inside-Object-Model-Stanley-Lippman/dp/0201834545) by Stanley B. Lippman
* [Reversing C++ Virtual Functions: Part 1](https://alschwalm.com/blog/static/2016/12/17/reversing-c-virtual-functions/) ([archived](http://archive.is/ezxOe))
* [Reversing C++ Virtual Functions: Part 2](https://alschwalm.com/blog/static/2017/01/24/reversing-c-virtual-functions-part-2-2/) ([archived](http://archive.is/T9wsl))

## Library based disassembler

For writing your own disassembly tools and automating disassembly tasks.

[Distorm](https://github.com/gdabah/distorm) can be used with python and other languages.
