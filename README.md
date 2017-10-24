# Reverse Engineering Guide

Not really a guide, but a listing of each of the problems in no particular order that I had to go through when learning to reverse engineer C++ code compiled to 32-bit X86 assembly.

*Note: this is a work in progress.*

## Hex editor

To modify the binary files you will need a hex editor.

A good one is [HT](https://github.com/sebastianbiallas/ht) which comes with a builtin disassembler where you can see the assembly representation of the hex code as you type.

Note that HT's disassembler runs on the whole file, disassembling not only the code, but also incorrectly on the header, data etc as well. I'm not sure why the the header being disassembled doesn't throw off the disassembler's alignment with the code, but it doesn't seem to be a problem.

There is also [beye](http://beye.sourceforge.net) which also has a disassembler but frequently crashes.

When modifying binary files, you cannot change the file size or move around blocks of instructions, as it will throw off memory offsets used in the instructions.

## Assembler tool

For converting assembly instructions into hex. The only one I am aware of is [rasm2](https://github.com/radare/radare2/wiki/Rasm2) from [radare2](https://radare.org).

It not only useful for converting simple instructions, but also ```jmp``` and ```call``` instructions, where the memory offsets will need to be calculated depending on the location of the instruction.

For example to call a function at ```0x8050e3c``` from address ```0x8051cee``` use:

```rasm2 -o 0x8051cee -a x86 -b 32 'call 0x8050e3c'``` to generate the hex ```e8 49 f1 ff ff```

A note of warning that rasm2 might have issues with some instructions where it will still output something but not what you gave it (I believe it occurs when dereferencing an operand for certain instructions).

## Disassembler

To convert the binary representation of the instructions back into assembly. 

Two good free ones are:
* objdump from [binutils](https://www.gnu.org/software/binutils) (e.g. ```objdump -M intel -S -D -z binary_file > dump.asm```)
* [borg](http://www.caesum.com) for win32 binaries

Note that the disassembled assembly cannot be used with an assembler due to the data sections being interpreted as instructions and also disassemblers sometimes insert extra information or instructions that aren't always valid assembly but are useful for the user.

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

## Layout of executables and libraries

Binary executable and library formats tend to contain a series of headers detailing information like versions, 16/32/64 bit, endianness, machine, linked libraries, code/data section locations, etc. After the headers or (maybe) in between are code/data section and other information like resources etc.

Typically on Linux the **Executable and Linkable file Format (ELF)** is used and on Windows the **Portable Executable (PE) file format**.

An excellent resource for **PE file format** is [The Portable Executable File Format from Top to Bottom by Randy Kath](http://www.csn.ul.ie/~caolan/pub/winresdump/winresdump/doc/pefile2.html) [(archived)](https://archive.is/uwMRp).

Some information on the **ELF file format** can be [found here](http://www.bottomupcs.com/elf.xhtml)

## Endianness

Important for when looking at the hex, as you maybe confused by the order it appears. Your binaries are most likely using little-endian (what Intel CPUs use).

For example in little-endian the integer ```54233456``` (or as hex ```0x33b8970```) will be stored as 4 bytes in this order ```0x70 0x89 0x3b 0x03```.

## Stacks and Alignment

Depending on the compiler options used when the binary was compiled, the stack may have to be aligned to a certain amount of bytes. Aligning it to 16 bytes is usually best, but you can figured it out by looking for padding (any stack memory that was declared and cleaned up without being used).

Be aware that function calls will push the return address onto the stack, you will need to remember to count them as well.

Another thing you might see is the stack being modfied like ```add esp,0xfffffff8```, this is just using the unsigned integer overflow where it wraps around, it is the same as ```sub esp,0x8```.

## Executable start address

Certain disassemblers output the memory address next to the disassembled instructions. Some incorrectly start at ```0x0``` and others like **objdump** give the correct address used at runtime.

For example X86 32-bit executables (I believe) should start at ```0x8048000```.

Also hex editors usually start at ```0x0```, so you may need to subtract for example ```0x8048000``` from an instruction's address to find it in the hex editor.

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

Some resources:
* [Position Independent Code on i386](http://archive.is/GZAvx)
* [What is the Symbol Table and What is the Global Offset Table?](http://grantcurell.com/2015/09/21/what-is-the-symbol-table-and-what-is-the-global-offset-table/) ([archived](http://archive.is/Rrmm3))

## Inserting instructions

The easiest way to reverse engineer a binary is to replicate the code bit by bit (usually starting with the main function) in your own shared library. You then load the shared library from the binary at runtime.

The [OpenRCT](https://openrct2.org/) project [used](http://archive.is/SDuL0) a program called [CFF Explorer](http://www.ntcore.com/exsuite.php) to load their own DLL. But I am unaware of a similar project for Linux, so I will show you how to modify the binary to load your own shared library and call a function from it. I will be using the ```dlopen``` and ```dlsym``` functions, which your binary will need to have available (accessible from the executable). There is probably a way to load them if they are not there, but I do not know how.

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

You will need to make room for your code in the binary. One strategy is to overwrite a section of code that is easy to replicate in your shared library. The second strategy is to overwrite a section of code that won't be missed like  *command line options* handling code, while hard coding in any options you need to use in the binary or your shared library.

## C++ References

* [Inside the C++ Object Model 1st Edition](https://www.amazon.com/Inside-Object-Model-Stanley-Lippman/dp/0201834545) by Stanley B. Lippman
* [Reversing C++ Virtual Functions: Part 1](https://alschwalm.com/blog/static/2016/12/17/reversing-c-virtual-functions/) ([archived](http://archive.is/ezxOe))
* [Reversing C++ Virtual Functions: Part 2](https://alschwalm.com/blog/static/2017/01/24/reversing-c-virtual-functions-part-2-2/) ([archived](http://archive.is/T9wsl))

## Library based disassembler

For writing your own disassembly tools and automating disassembly tasks.

[Distorm](https://github.com/gdabah/distorm) can be used with python and other languages.
