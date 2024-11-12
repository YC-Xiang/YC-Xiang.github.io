---
date: 2024-11-12T22:35:27+08:00
title: "GNU Linker Script"
tags:
  - Linker Script
categories:
  - MISC
---

# Reference

官方文档：

https://sourceware.org/binutils/docs/ld.pdf

# 2. Invocation

讲了 command line ld 的各种选项。

# 3. Linker Script

## 3.1 Basic Linker Script Concepts

**loadable section and allocatable section**

A section may be marked as **loadable**, which means that the contents should be loaded into memory when the output file is run.

A section with no contents may be **allocatable**, which means that an area in memory should
be set aside, but nothing in particular should be loaded there (in some cases this memory
must be zeroed out).

A section which is neither loadable nor allocatable typically contains
some sort of debugging information.

</br>

**VMA and LMA**

loadable or allocatable output section has two address **VMA** and **LMA**.

VMA(virtual memory address) is the address the section will have when the output file is run.

LMA(load memory address) is the address at which the section will be loaded.

In most cases **VMA=LMA**.

An example of when they might be different is when a data section is loaded into ROM, and then copied
into RAM when the program starts up (this technique is often used to initialize global
variables in a ROM based system). In this case the ROM address would be the LMA, and
the RAM address would be the VMA.

## 3.2 Linker Script Format

## 3.3 Simple Linker Script Example

利用`SECTIONS`来描述 output file 的 memory layout。

最简单的 linker script:

```txt
SECTIONS
{
  . = 0x10000;
  .text : { *(.text) }
  . = 0x8000000;
  .data : { *(.data) }
  .bss : { *(.bss) }
}
```

所有 input file 的\*(.text)段放到 output file .text 段，地址为 0x10000;

.data 和.bss 段同理。

## 3.4 Simple Linker Script Commands

### 3.4.1 Setting the Entry Point

在 linker script 中通过 `ENTRY(symbol)` 来设置 program 执行的第一条指令。

program 的 entry point 根据以下顺序来寻找：

- the ‘-e’ entry command-line option;
- the ENTRY(symbol) command in a linker script;
- the value of a target-specific symbol, if it is defined; For many targets this is start.
- the address of the first byte of the code section, if present and an executable is being created - the code section is usually ‘.text’, but can be something else;
- The address 0.

### 3.4.2 Commands Dealing with Files

`INCLUDE filename`

include 其他 linker script。会查找当前目录和 command line -L 指定的目录。

`INPUT(file, file, ...)`  
`INPUT(file file ...)`

用来传入 input object files，这样在 command line ld 命令不用主动加.o files 了。

INPUT(-lfile)，相当于加进了 libfile.a

`GROUP(file, file, ...)`  
`GROUP(file file ...)`

和 INPUT 类似，不过这里的 file 都是.a 文件。

`AS_NEEDED(file, file, ...)`  
`AS_NEEDED(file file ...)`

用于放在 INPUT 和 GROUP 命令中，// TODO: 用法没看懂

`OUPUT(filename)`

指定输出文件名称，和 command line `-o filename`作用一样。

`SEARCH_DIR(path)`

指定查找 library 路径，和 command line `-L path`作用一样。

`STARTUP(filename)`

和 INPUT 命令一样，传入 object file，不过这边传入的是第一个 input file。

This may be useful when using a system in which the entry
point is always the start of the first file

### 3.4.3 Commands Dealing with Object File Formats

`OUTPUT_FORMAT(bfdname)`  
`OUTPUT_FORMAT(default, big, little)`

> BFD 库（二进制文件描述库）是 GNU 项目用于解决不同格式的目标文件的可移植性的主要机制。到 2003 年为止，它支持 25 种不同 CPU 体系结构上的大约 50 种文件格式。

设置 output file 的 BFD 格式。

第一种传入一个参数，直接指定 BFD 格式。

第二种传入三个参数，比如 mips 架构 OUTPUT_FORMAT(elf32-bigmips, elf32-bigmips, elf32-littlemips)

那么默认的为第一个，如果 command line 传入了 `-EL` 那就是第二个，传入了 `-EB` 那就是第三个。

`TARGET(bfdname)`

设置从 input files 读取的 BFD format。和 command line `-b bfdname`作用一样。

如果使用了 TARGET 而没有使用 OUTPUT_FORMAT，那么最后一个 TARGET 命令设置的格式将作为 output file 的格式。

### 3.4.4 Assign alias names to memory regions

`REGION_ALIAS(alias, region)`

region 为 MEMORY 命令中定义的内存区域。

```txt
MEMORY
  {
    ROM : ORIGIN = 0, LENGTH = 2M
    ROM2 : ORIGIN = 0x10000000, LENGTH = 1M
    RAM : ORIGIN = 0x20000000, LENGTH = 1M
  }
REGION_ALIAS("REGION_TEXT", ROM);
REGION_ALIAS("REGION_RODATA", ROM2);
REGION_ALIAS("REGION_DATA", RAM);
REGION_ALIAS("REGION_BSS", RAM);
```

```txt
INCLUDE linkcmds.memory
SECTIONS
  {
    .text :
      {
	*(.text)
      } > REGION_TEXT
    .rodata :
      {
        *(.rodata)
        rodata_end = .;
      } > REGION_RODATA
    .data : AT (rodata_end)
      {
        data_start = .;
        *(.data)
      } > REGION_DATA
    data_size = SIZEOF(.data);
    data_load_start = LOADADDR(.data);
    .bss :
      {
	*(.bss)
      } > REGION_BSS
  }
```

通过`>REGION_XXX`, 将某些段放进指定的别名，即内存区域中。

### 3.4.5 Other Linker Script Commands

`ASSERT(exp, message)`
