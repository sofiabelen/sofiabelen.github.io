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
    - C++
    - Rust
    - tutorial
    - pointers
categories:
    - C++
    - Rust
    - tutorial
    - pointers
featured: false
comments: true
---

I've been exploring Rust, and one topic that got me intrigued is pointer provenance, particularly because it's not easy to visualize it.

For getting a feel of vtables, what I wanted was to look at the actualy memory, to see what it really looked like and prove that I could in fact call these functions through their respective function pointers. With provenance, I wanted to take it apart the same way, but it turns out it's not so straightforward. This only got me more intrigued, so let's see what we can find!

My first time hearing about provenance, I wondered why I hadn't heard about it before? It seems so crucial, yet a bit of an obscure concept.

## What is Provenance: Pointers Are Not Just Glorified Ints

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

Let us follow the white rabbit.

<figure>
<img src="sincerely-media-2yL77QTLCq4-unsplash.jpg" atl="">
  <figcaption>
Yes, I've been trying to get more rabbits on my page. Somehow, only cats seemed to make the cut. This one reminds me of my childhood pet, Kaisie, honary mention.

Photo by <a href="https://unsplash.com/@sincerelymedia?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Sincerely Media</a> on <a href="https://unsplash.com/photos/white-and-black-rabbit-2yL77QTLCq4?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
  </figcaption>
</figure>

This brings us to the core of provenance. That "*derived from*" relation is **provenance**: for every pointer, the compiler keeps track of which allocation it may touch.

I tried to find a way to print this mysterious provenance, but it turns out that it's purely a compiler abstraction. It does not exist as an actual data field stored alongside the memory address.

## Inspecting Provenance: LLVM IR

So what can we do to inspect it? If the compiler knows about then it must exist somewhere. If so, there must be a way to *see* it.

This is how I learned about LLVM IR (intermediate representation), which is essentially a low-level programming language that acts as a bridge between high-level languages (like Rust, C++, etc) and machine code.

We can use the `-emit-llvm` flag on godbolt to see this.

Let's try it for this code instead, where we pass `&x` to `opaque` to see what happens when the compiler doesn't optimize that part away:

```c++
void opaque(int*);

int f() {
    int x = 1;
    opaque(&x);
    return x;
}
```

<pre><code>
define dso_local noundef i32 @f(int*)(ptr noundef <mark>%0</mark>) local_unnamed_addr #0 !dbg !14 {
  <mark>%2 = alloca i32, align 4</mark> // --> root of %1 allocation (x)
    #dbg_assign(i1 poison, !22, !DIExpression(), !23, ptr %2, !DIExpression(), !24)
    #dbg_value(ptr %0, !21, !DIExpression(), !24)
  call void @llvm.lifetime.start.p0(ptr nonnull %2) #3, !dbg !25
  store i32 1, ptr %2, align 4, !dbg !26
    #dbg_assign(i32 1, !22, !DIExpression(), !28, ptr %2, !DIExpression(), !24)
  tail call void @opaque(int*)(<mark>ptr noundef %0</mark>), !dbg !29
  call void @opaque(int*)(<mark>ptr noundef nonnull %2</mark>), !dbg !30
  %3 = load i32, ptr %2, align 4, !dbg !31
  call void @llvm.lifetime.end.p0(ptr nonnull %2) #3, !dbg !32
  ret i32 %3, !dbg !33
}

declare void @llvm.lifetime.start.p0(ptr captures(none)) #1

declare !dbg !34 void @opaque(int*)(ptr noundef) local_unnamed_addr #2

declare void @llvm.lifetime.end.p0(ptr captures(none)) #1

attributes #0 = { mustprogress uwtable "min-legal-vector-width"="0" "no-trapping-math"="true" "stack-protector-buffer-size"="8" "target-cpu"="x86-64" "target-features"="+cmov,+cx8,+fxsr,+mmx,+sse,+sse2,+x87" "tune-cpu"="generic" }
attributes #1 = { mustprogress nocallback nofree nosync nounwind willreturn memory(argmem: readwrite) }
attributes #2 = { "no-trapping-math"="true" "stack-protector-buffer-size"="8" "target-cpu"="x86-64" "target-features"="+cmov,+cx8,+fxsr,+mmx,+sse,+sse2,+x87" "tune-cpu"="generic" }
attributes #3 = { nounwind }
</code></pre>

What the compiler does is construct a DAG (direct acyclic graph), to answer the question of where a pointer comes from and whether two pointers can touch the same memory.

If the root origins don't match, LLVM can guarantee that writing through `%0` will never alter the value stored at `%3` or `%2`.

```
[ Root: External Allocation ]              [ Root: Stack Slot #1 ]
             │                                        │
            %0                                       %2
                                                      │
                                                      ▼
                                                     %3
```

There is much more to explore here that it's hard not to get carried away because it's truly fascinating. However, my original goal was to compare how C++ and Rust handle this, so let's continue.

## C++: What Happens If We Cast to an Integer?

If we were to cast `&x` to an integer, then LLVM must assume that anyone could rebuild a pointer to `x` from that integer.

```cpp
#include <cstdint>

void g(std::uintptr_t);
void opaque(int*);

int f(int* p) {
    int x = 1;
    g(reinterpret_cast<std::uintptr_t>(&x)); // address of x is now 'exposed'
    opaque(p);
    return x;
}
```

The result is that it gives up the optimization:

```
f(int*):
        push    rbx
        sub     rsp, 16
        mov     rbx, rdi
        mov     dword ptr [rsp + 12], 1
        lea     rdi, [rsp + 12]
        call    g(unsigned long)@PLT
        mov     rdi, rbx
        call    opaque(int*)@PLT
        mov     eax, dword ptr [rsp + 12]
        add     rsp, 16
        pop     rbx
        ret
```

Which is a pity here because maybe we just wanted to read the address but we never intended to cast it back to a pointer. Wouldn't it be nice if we could communicate this intent with the compiler?

## Rust: Making Your Intent Explicit

This is exactly what Rust provides with its powerful provenance APIs.

### Inspecting Without Exposing

```rust
let p: *const u32 = &x;
let a: usize = p.addr();
```

`p.addr()` extracts the raw memory addres integer while telling LLVM:

> I am reading this numeric address for inspection, but I guarantee I WILL NOT cast it back to a valid pointer.

Therefore, provenance is not exposed, and `x` remains isolated for optimizations (loop vectorization, register caching, etc).

### Preserving Provenance

<figure>
<img src="michael-myers-FvuisLAN-rA-unsplash.jpg" atl="">
  <figcaption>
Pointers reproduce like rabbits and someone has to keep track of the family tree.

Photo by <a href="https://unsplash.com/@myers2021?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Michael Myers</a> on <a href="https://unsplash.com/photos/a-group-of-rabbits-sitting-on-top-of-a-pile-of-hay-FvuisLAN-rA?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
  </figcaption>
</figure>

On lock-free data structures, we sometimes need to store status flags in the lower unused bits of a pointer. It's a very cool trick, which left me eyes wide open for a minute when I first read about it. I talk more about this in [this post](). Essentially, due to alignemnt, the lower bits of pointers are "wasted" (always 0), so we can use them to store flags.

So, what we need in that case is to cast the pointer to an integer, modify it to hold extra info (flag), and then cast it back to a valid pointer (without the flags). `with_addr` allows us to do just that, while maintaining provenance.

```rust
let tagged = p.with_addr(p.addr() | 1); // create the integer holding the pointer value + flag while conserving provenance
// ...
let revived_ptr = tagged.with_addr(tagged.addr() & !1); // recreate a valid pointer from the integer
```

`addr()` reads teh address without exposing provenance.

`with_addr()` re-attaches the pointer's original provenance.

### Legacy Rust, Similar To C++

In legacy code, we'd have to do something like this:

```rust
let addr = p as usize;
let tagged_addr = addr | 1;
let revived_ptr = (tagged_addr & !1) as *const u32;
println!("*revived_ptr: {}", unsafe { *revived_ptr});
```

This code will run, but if we run it with Miri

```
cargo run miri
```

it emits a warning:

```
warning: integer-to-pointer cast
 --> src/main.rs:6:22
  |
6 |     let revived_ptr = (tagged_addr & !1) as *const u32;
  |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ integer-to-pointer cast
  |
  = help: this program is using integer-to-pointer casts or (equivalently) `ptr::with_exposed_provenance`, which means that Miri might miss pointer bugs in this program
  = help: see https://doc.rust-lang.org/nightly/std/ptr/fn.with_exposed_provenance.html for more details on that operation
  = help: to ensure that Miri does not miss bugs in your program, use Strict Provenance APIs (https://doc.rust-lang.org/nightly/std/ptr/index.html#strict-provenance, https://crates.io/crates/sptr) instead
  = help: you can then set `MIRIFLAGS=-Zmiri-strict-provenance` to ensure you are not relying on `with_exposed_provenance` semantics
  = help: alternatively, `MIRIFLAGS=-Zmiri-permissive-provenance` disables this warning

*revived_ptr: 1
warning: 1 warning emitted
```

Actually, the reason that it runs is also interesting. What happens is similar to the C++ example. `p as usize` exposes the provenance of `p`, and since the reconstructed pointer points to the same allocation, provenance is recovered. Provenance would be lost if we did some pointer arithmetics and landed in a different allocation.

### Casting Without Exposing Provenance

To see what happens when provenance is **not** exposed, we can use `ptr::without_provenance`, which creates a bare address with no allocation behind it. (It's useful for dangling or sentinel pointers, but it an't be dereferenced).

```rust
let bare_ptr = std::ptr::without_provenance::<u32>(p.addr());
println!("*tagged_ptr: {}", unsafe { *bare_ptr});
```

As expected, Miri complains about a <mark>*dangling pointer*</mark> and tells us explicitily that <mark>it has no provenance.</mark>

<pre><code>
error: Undefined Behavior: memory access failed: attempting to access 4 bytes, but got 0x20b28[noalloc] which is a <mark>dangling pointer (it has no provenance)</mark>
  --> src/main.rs:11:42
   |
11 |     println!("*tagged_ptr: {}", unsafe { *bare_ptr});
   |                                          ^^^^^^^^^ Undefined Behavior occurred here
   |
   = help: this indicates a bug in the program: it performed an invalid operation, and caused Undefined Behavior
   = help: see https://doc.rust-lang.org/nightly/reference/behavior-considered-undefined.html for further information

note: some details are omitted, run with `MIRIFLAGS=-Zmiri-backtrace=full` for a verbose backtrace

error: aborting due to 1 previous error
</code></pre>

## Miri vs ASan

## Recap

Provenance, in short, is a hidden piece of information tracking "which allocation" a pointer originates from, which LLVM uses for optimizations. In C++, `uintptr_t` casts force the optimizer to prepare for the worst case by disabling some optimizations.

In Rust, `addr()`, `with_addr()` and `without_provenance()` allow you to communicate your intent with the compiler, allowing it to optimize when possible, and catching bugs that stem from unintended usage.

## Resources
