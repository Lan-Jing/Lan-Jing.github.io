---
title: "Programming Languages: Strict Aliasing Rules"
categories:
  - System
  - PL
---

## What Are Strict Aliasing Rules

The Strict Aliasing Rules are rules imposed by the C standard, implemented by GNU C compilers to perform optimized code generation.

These rules contradict some of our naive ways to perform convenient memory access, leading to undefined behaviors of our running programs. This should be particularly taken care of in systems programming using C.

## Why Strict Aliasing Rules?

Strict Aliasing Rules are several assumptions on the memory layout of the program. With these assumptions, compilers are safe to perform optimizations on load/store instructions.

For example, with Strict Aliasing Rules, the GCC compilers can remove repeated load instructions before each assignment, which may significantly speedup the loop execution. 

[See this web for more details](https://cellperformance.beyond3d.com/articles/2006/06/understanding-strict-aliasing.html)

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

The optimization can be done because GCC assumes *values[]* never overlaps with *uniform->b*, so only one load is needed for *uniform->b*. Without the assumption, GCC may expect *uniform->b* differs in each assignment, thus loading it many times. However, overlapping is rarely the case.

## How to Do That Properly

The so-called standard way of doing dynamic casting in C is by using union or pointers to union. For example, if one wants to read a 32-bit space as one **int** or two **shorts**, it's best to have your code like:

```c
typedef union
{
  uint32_t u32;
  uint16_t u16[2];
} U32;

uint32_t swap_words(uint32_t arg)
{
  U32 in;
  in.u32 = arg;
  swap(in.u16[0], in.u16[1]);

  return in.u32;
}
```

Built-in types that are compatible(such as signed/unsigned int) are fine to be referenced by pointers of different types. In any circumstances, memory is allowed to be interpreted as char, using char*.