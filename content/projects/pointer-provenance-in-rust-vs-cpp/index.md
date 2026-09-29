---
layout: project
title: "Pointer Provenance in Rust vs C++"
thumbnail: 
image: 
preview: ""
description: ""
date: 2026-09-29
publishDate: 2026-09-28
tags:
    - Rust
    - tutorial
    - pointers
categories:
    - Rust
    - tutorial
featured: false
draft: true
comments: true
---

I've been exploring Rust, and one topic that got me intrigued is pointer provenance, particularly because it's not easy to visualize it.

For getting a feel of vtables, what I wanted was to look at the actualy memory, to see what it really looked like and prove that I could in fact call these functions through their respective function pointers. With provenance, I wanted to take it apart the same way, but it turns out it's not so straightforward. This only got me more intrigued, so let's see what we can find!

My first time hearing about provenance, I wondered why I hadn't heard about it before? It seems so crucial, yet a bit obscure of a concept.

## What is Provenance

## Pointers are not just ints

Let's start by taking a good look at this snippet. It's short, but our intuition can trick us if it's the first time coming across this concept. `void opaque(int*)` is forwared declared here, so for it's a black box. It could, for example, dereference `p` and write to memory.

```cpp
void opaque(int*);

int f(int* p) {
    int x = 1;
    opaque(p);
    return x;
}
```

If we compile this with `-O2`, Clang optimizes this to return 1. Let's try this on godbolt.

```assembly
f(int*):
        sub     rsp, 8
        call    opaque(int*)
        mov     eax, 1
        add     rsp, 8
        ret
```

The question is, how does it know that `int* p` doesn't point to the same memory address where `x` lives, meaning that `opaque(p)` could end up modifying `x`?

The compiler assumes `p` can't equal `&x` because `p` wasn't *derived* from `x`. When I first heard about this, I thought it was a bit of a strange concept. What does it mean that something is *derived* from something?

I mean, if pointers were just numbers, `p` could theoretically happen to hold the same memory address of `x`'s allocation on the stack. Good thing they're not just numbers :)

Let's follow the white rabbit.

<figure>
<img src="sincerely-media-2yL77QTLCq4-unsplash.jpg" atl="">
  <figcaption>
Yes, I've been trying to get more rabbits on my page. Somehow, only cats seemed to make the cut. This one reminds me of my childhood pet, Kaisie, honary mention.

Photo by <a href="https://unsplash.com/@sincerelymedia?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Sincerely Media</a> on <a href="https://unsplash.com/photos/white-and-black-rabbit-2yL77QTLCq4?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
  </figcaption>
</figure>

This brings us to the core of provenance. That "*derived from*" relation is **provenance**: for every pointer, the compiler keeps track of which allocation it may touch.

I tried to find a way to print this mysterious provenance, but it turns out that it's purely a compiler abstraction. It does not exist as an actual data field stored in hardware alongside the 64-bit memory address.

## Inspecting Provenance

- llvm ir: what is it? yep, first time hearing about this as well.

## cast to an integer

## rust

## miri vs ASan
