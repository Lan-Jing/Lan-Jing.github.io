---
title: "Programming Languages: Strict Aliasing Rules"
categories:
  - System
  - PL
---

## What Are Strict Aliasing Rules

The Strict Aliasing Rules are rules imposed by C standard, implemented by GNU C compilers to perform optimized code generation.

These rules contradict some of our naive ways to perform convenient memory access, leading to undefined behaviors of our running programs. This should be particularly taken care of in systems programming using C.

## Why Strict Aliasing Rules?

Strict Aliasing Rules are in fact several assumptions on memory layout of the program. With these assumptions compilers are safe to perform optimizations on load/store instructions.

For example, with Strict Aliasing Rules the GCC compilers can remove repeated load instructions before each assignment, which may significantly speedup the loop execution. 

Source: https://cellperformance.beyond3d.com/articles/2006/06/understanding-strict-aliasing.html

```c
typedef struct
{
  uint16_t a;
  uint16_t b;
  uint16_t c;
} Sample;

void
test( uint32_t* values,
      Sample*   uniform,
      uint64_t  count )
 {
   uint64_t i;
   for (i = 0;i < count;i++) {
     values[i] += (uint32_t)uniform->b;
   }
 }
```

The optimization can be done because GCC assumes *values[]* never overlaps with *uniform->b*, so only one load is needed for *uniform->b*. Without the assumption, GCC may expect *uniform->b* differs in each assignment, thus loading it many times. However, overlap is rarely the case.

## Other Practice