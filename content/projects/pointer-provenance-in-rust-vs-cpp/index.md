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

```cpp
void opaque(int*);

int f(int* p) {
    int x = 1;
    opaque(p);   // p is some unrelated pointer
    return x;
}
```

If we compile this with `-O2`, Clang optimizes this to return 1. The question is, how does it know that `int* p` doesn't point to the same memory address where `x` lives, meaning that `opaque(p)` could end up modifying `x`?

godbolt:

```assembly
f(int*):
        sub     rsp, 8
        call    opaque(int*)
        mov     eax, 1
        add     rsp, 8
        ret
```

The compiler assumes `p` can't equal `&x` because `p` wasn't *derived* from `x`. When I first heard about this, I thought it was a bit of a strange concept. What does it mean that something is *derived* from something?

Let's follow the white rabbit.
