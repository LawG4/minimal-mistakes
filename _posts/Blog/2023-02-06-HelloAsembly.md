---
title: "Assembly on 64 bit Windows"
typora-root-url: ../../

header:
  teaser: /assets/Blog/RustCompatability/Teaser.png
  og_image: /assets/Blog/RustCompatability/Thumb.png
  image: /assets/Blog/RustCompatability/Thumb.png
published: false
categories:
  - Blog
tags:
  - Assembly
  - Low-level
  - Programming Languages
  - Introduction
  - Windows
  - x86
  - x64
---

On my quest to understand the entirety of the software stack, I have started looking into assembly. I found a few resources, but the majority focussed on Linux - as let's be honest who's programming assembly on windows? 

## Development Environment

As with all programming endeavours, you're going to need a way to transform the code you write into machine readable instructions. On windows this means installing Visual Studio, which will give you a set of build tools, but we're mainly interested in:

- Native Tools Command Prompt
- cl.exe: C/C++ Compiler
- link.exe: Linker

You'll also want to use the visual installer to make sure that there is at least one Windows SDK installed, we need this for the kernel. And finally you'll want `nasm` to assemble the code into an object file. There is `ml64` which is Microsoft's official assembler, however this assembler is pretty limited. 

## Object Files, Executables, Symbols

There's a couple different stages in the compilation process which most will be familiar with. Starting with human readable source code, which is then assembled into an object file. 

An object file can be thought of as the middle stage in the compilation. The functions and other resources are compiled into machine code, but there is extra information inside the object file such as the symbol table which keeps track of each of the resources names and locations within that object file. 

## Entry Point

Similarly to all programming languages, we need to define an entry point. The location where the first instructions of the program need to be placed. In `C` there is actually some stuff that executes before the user gains control in order to set up `libc`. 

However, in assembly there is nothing done before our code starts executing. Aside from the stuff that the operating system does when starting an executable (Reading the exe's header, and jumping to the start address) 

Since, we're  

## Compiled Languages Interaction

The vast majority of libraries written already are written in `C/C++` especially on retro platforms. Rather than rewriting those from scratch, we can instead make our languages interact.

Let's first consider how function calls work in general. In most cases a function can be generalised into four components; the name, input parameters, output parameters, and the  First you have to define a set of rules for how the binary interacts

## Calling C From Rust

For now let's take the simple example of a function which adds two 32 bit unsigned integers together. 

```c
// File called lib.c
// Compile on windows with : "cl.exe /c lib.c"
// Builds into lib.obj
#include <stdint.h>
uint32_t add_two(uint32_t a, uint32_t b) {return a + b;}
```

## Calling Rust From C
