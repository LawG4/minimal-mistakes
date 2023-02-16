---
title: "Learning Rust with backwards compatability"
typora-root-url: ../../

header:
  teaser: /assets/Blog/RustCompatability/Teaser.png
  og_image: /assets/Blog/RustCompatability/Thumb.png
  image: /assets/Blog/RustCompatability/Thumb.png
published: false
categories:
  - Blog
tags:
  - Rust
  - C
  - C++
  - Programming Languages
---

I have a specific style of programming, the majority of my code ends up inside a library. while also ensuring the caller is responsible for memory management, this is because dynamic memory allocations can fail at any point. 

I also want my code to be able to run on retro platforms, as a result I have been programming in `C`, but I think it might be time to bring my programming into the modern age. 

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


