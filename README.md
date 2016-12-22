# Reverse Engineering Tutorial

A tutorial for reverse engineering C/C++ code compiled to X86 assembly.

## Getting Started

You will need:

#### a disassembler

I use [objdump from binutils](https://www.gnu.org/software/binutils). 

#### a hex editor

I use [HT](http://hte.sourceforge.net), which also comes with a builtin disassembler, allowing you to see the assembly representation of the hex code as you type.

#### to know assembly

Have a basic knowledge of assembly used to create the binary.

In this tutorial I will use X86 assembly. There are two main assembly styles to choose from, Intel and AT&T. I prefer Intel because it looks less cluttered. The [differences can be found here](http://archive.is/f1dul). The book I used was *PC Assembly Language by Paul A. Carter* [available free online here](http://pacman128.github.io/pcasm).

#### assembly to hex converter

To insert code into the binary you will need program to convert your assembly into hex code. I use rasm2 from [radare2](https://radare.org).

#### a debugger

I use [gdb](https://www.gnu.org/software/gdb/). Useful for debugging errors you (or the original developers) made or for looking at the runtime values of the registers/stack/heap.

#### a decompiler (optional)

I've used [IDA](https://www.hex-rays.com/products/ida/). Useful for understanding the structure of the code, and also the decompilers can automatically trace the code and find the name of things like externed global variables (from libraries) or identify the virtual tables of classes, which can be difficult when only looking at the assembly.

#### an assembler, or a C/C++ compiler or both (optional)

An assembler for either generating your own shared library, executable or just object files. A C/C++ compiler for the same reason, depending on if you prefer to write in either assembly, C/C++ or both.

## Minor details
I will mention a few things here that you should know of, but don't need to go indepth.

* know how the executable/dll/shared library files are layed out. For Linux, ELF files are commonly used, some websites I read about them are [[1]](http://archive.is/wJW5i), [[2]](http://archive.is/JyChY) and [[3]](http://archive.is/DBnia).
* Know the difference between **real mode** and **protected mode** for programs.
* know about endianness 

## Beginner Misconceptions

When I first stated learning about reverse engineering, I had a couple of misconceptions.

* The assembly disassembled from a binary can run through an assembler. Some dissemblers have slight syntax changes, memory addresses and offsets cause problems, global variables/data are treated like instructions and not data.

* Decompiled code can be recompiled. Of the various decompilers I tried, they all outputted invalid C/C++ syntax in certain areas, also they often cannot resolve the data types properly, only getting the data type sizes correct.

## Disassembling the binary

I will be using only the 32-bit X86 assembly.

To disassemble a binary use **objdump**.

```	objdump -M intel -S -D -z binary_file > dump.asm```

*  ```-M intel``` tells it to output Intel assembly
* ```-S``` not sure ...
* ```-D``` outputs everything, useful for seeing the hex values of global variables/data
* ```-z``` doesn't strip out any *uneccessary* code, otherwise they are replaced with ellipses

If you are reverse engineering an executable you can look for the **main** function, or if a shared library you can look for the function you are interested in.

Depending on the compiler, version and options used, the code may look something like this:
```
8051600:	55                   	push   ebp
8051601:	89 e5                	mov    ebp,esp
8051603:	83 ec 10             	sub    esp,0x10
8051606:	56                   	push   esi
8051607:	53                   	push   ebx
8051608:	e8 00 00 00 00       	call   805160d <_start@@Base+0xdd>
805160d:	5b                   	pop    ebx
805160e:	81 c3 07 3b 03 00    	add    ebx,0x33b07
 
 ...
8051af2:	8d a5 58 ef ff ff    	lea    esp,[ebp-0x10a8]
8051af8:	5b                   	pop    ebx
8051af9:	5e                   	pop    esi
8051afa:	5f                   	pop    edi
8051afb:	89 ec                	mov    esp,ebp
8051afd:	5d                   	pop    ebp
8051afe:	c3                   	ret    
```

## Inserting code

The easiest way to reverse engineer a binary is to replicate the code bit by bit (starting with the main/root function) in your own shared library. This can be done in assembly or C/C++ or both. Then have the binary file load it at runtime.

The [OpenRCT](https://openrct2.org/) project [used](http://archive.is/SDuL0) a program called [CFF Explorer](http://www.ntcore.com/exsuite.php) to load their DLL. Though in this tutorial I will inserting my own code into the binary to load the shared library.

The first problem is that you cannot change the size of the binary file, otherwise it will throw off all the memory addresses used. So we will need to make some room by overwriting the existing code which is the second problem. You can either identify some unecessary code or you can find some simple code that can be easily replicated in your shared library.

I will be using the **dlfcn.h** library. I've only used this on Linux and I am not sure if it is available on Windows/WIN32.

The inserted code will look something like:

```C
int main(int argc, char *argv[]) {
    //...

    void *lib=dlopen("./libmy.so",RTLD_LAZY);
    void (*func)(char*,int,char**)= dlsym(lib,"myfunc");
    func(global,argc,argv);

    //...
    return 0;
}
```

The same code in assembly, but with a few minor tweaks:

```asm
	sub    esp,0x18
	
	;push null terminated string "./libmy.so" onto the stack
	mov    DWORD [ebp-0xc],0x696c2f2e ;;./li
	mov    DWORD [ebp-0x8],0x2e796d62 ;;bmy.
	mov    DWORD [ebp-0x4],0x6f73 ;;so\0
	
	push   0x1 ;;RTLD_LAZY
	lea    eax,[ebp-0xc]
	push   eax
	call   dlopen
	add    esp,0x20

	sub    esp,0x18
	mov    DWORD [ebp-0xc],eax
	
	;push null terminated string "myfunc" onto the stack
	mov    DWORD [ebp-0x8],0x7566796d ;;myfu
	mov    DWORD [ebp-0x4],0x636e ;;nc\0
	lea    eax,[ebp-0x8]
	push   eax
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

Things to note:

* the file and function names are stored on the stack
* uses dlopen, dlsym
* stack is always 16byte aligned
* use of eax register only
* ebx+0x20a points to the start of the global variables
