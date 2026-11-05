---
layout: project
title: "Pointer Provenance in Rust vs C++"
thumbnail: yves-alarie-MmZet_7mzMY-unsplash-cropped.jpg
image: yves-alarie-MmZet_7mzMY-unsplash.jpg
preview: "Exploring pointer provenance, inspecting LLVM IR, and comparing C++ and Rust's approach."
description: "Exploring pointer provenance, inspecting LLVM IR, and comparing C++ and Rust's approach."
date: 2026-11-02
publishDate: 2026-11-01
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
featured: true
comments: true
---

I've been exploring Rust, and one topic that got me intrigued is pointer
provenance, particularly because it's not easy to visualize.

To get a feel for
[vtables](https://sofiabelen.github.io/projects/visualizing-rusts-vtables-how-dyn-trait-works-in-memory/),
what I wanted was to look at the actual memory, to see what it really looked
like and prove that I could in fact call these functions through their
respective function pointers. With provenance, I wanted to take it apart the
same way, but it turns out it's not so straightforward. This only got me more
intrigued, so let's see what we can find!

My first time hearing about provenance, I wondered, why hadn't I heard about it
before? It seems so crucial, yet a bit of an obscure concept.

## What is Provenance: Pointers Are Not Just Glorified Ints

<figure>
<img src="chuttersnap-Fy7guhZfmAM-unsplash.jpg" alt="">
  <figcaption>
Victoria crowned pigeon: some will say it's just a pigeon.

Photo by <a href="https://unsplash.com/@chuttersnap?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">CHUTTERSNAP</a> on <a href="https://unsplash.com/photos/red-and-blue-bird-with-white-wings-Fy7guhZfmAM?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
  </figcaption>
</figure>

Let's start by taking a good look at this snippet. It's short, but our intuition
can trick us if it's our first time coming across this concept. `void
opaque(int*)` is forward declared here, so for us it's a black box. Neither we
nor the compiler can rule out that this function dereferences `p` and writes to
memory.

```cpp
void opaque(int*);

int f(int* p) {
    int x = 1;
    opaque(p);
    return x;
}
```

If we compile this with `-O2`, Clang optimizes this to return 1. Let's try this
on Godbolt:

<iframe width="800px" height="200px" src="https://godbolt.org/e?hideEditorToolbars=true#g:!((g:!((g:!((h:codeEditor,i:(filename:'1',fontScale:14,fontUsePx:'0',j:1,lang:c%2B%2B,selection:(endColumn:1,endLineNumber:8,positionColumn:1,positionLineNumber:8,selectionStartColumn:1,selectionStartLineNumber:8,startColumn:1,startLineNumber:8),source:'void+opaque(int*)%3B%0A%0Aint+f(int*+p)+%7B%0A++++int+x+%3D+1%3B%0A++++opaque(p)%3B%0A++++return+x%3B%0A%7D%0A'),l:'5',n:'0',o:'C%2B%2B+source+%231',t:'0')),k:44.33887569279493,l:'4',n:'0',o:'',s:0,t:'0'),(g:!((h:compiler,i:(compiler:clang2310,filters:(b:'0',binary:'1',binaryObject:'1',commentOnly:'0',debugCalls:'1',demangle:'0',directives:'0',execute:'1',intel:'0',libraryCode:'0',trim:'1',verboseDemangling:'0'),flagsViewOpen:'1',fontScale:14,fontUsePx:'0',j:1,lang:c%2B%2B,libs:!(),options:'-O2',overrides:!(),selection:(endColumn:1,endLineNumber:1,positionColumn:1,positionLineNumber:1,selectionStartColumn:1,selectionStartLineNumber:1,startColumn:1,startLineNumber:1),source:1),l:'5',n:'0',o:'+x86-64+clang+23.1.0+(Editor+%231)',t:'0')),k:55.66112430720507,l:'4',n:'0',o:'',s:0,t:'0')),l:'2',n:'0',o:'',t:'0')),version:4"></iframe>

<!--
```assembly
f(int*):
        sub     rsp, 8
        call    opaque(int*)
        mov     eax, 1
        add     rsp, 8
        ret
```
-->

The question is, how does it know that `int* p` doesn't point to the same memory
address where `x` lives (meaning that `opaque(p)` could end up modifying `x`)?

The compiler assumes `p` can't equal `&x` because `p` wasn't *derived* from `x`.

When I first heard about this, I thought it was a bit of a strange concept. What
does it mean that something is *derived* from something?

I mean, if pointers were just numbers (memory addresses), `p` could
theoretically happen to hold the same memory address as `x`'s allocation on the
stack. Good thing they're not just numbers!

Let us follow the white rabbit.

<figure>
<img src="sincerely-media-2yL77QTLCq4-unsplash.jpg" alt="">
  <figcaption>
Yes, I've been trying to get more rabbits on my page. Somehow, only cats seemed to make the cut. This one reminds me of my childhood pet, Kaisie (honorary mention).

Photo by <a href="https://unsplash.com/@sincerelymedia?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Sincerely Media</a> on <a href="https://unsplash.com/photos/white-and-black-rabbit-2yL77QTLCq4?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
  </figcaption>
</figure>

This brings us to the core of provenance. <mark>That "*derived from*" relation
is **provenance**: for every pointer, the compiler keeps track of which
allocation it belongs to and may, therefore, touch.</mark>

I tried to find a way to print this mysterious provenance, but it turns out that
it's purely a compiler abstraction. It does not exist as an actual data field
stored alongside the memory address. (There are exceptions though: feel free to
look up the CHERI architecture to go down another rabbit hole).

## Inspecting Provenance: LLVM IR

So what can we do to inspect it? If the compiler knows about it then it must
exist somewhere. If so, there must be a way to *see* it.

This is how I learned about LLVM IR (intermediate representation), which is
essentially a low-level programming language that acts as a bridge between
high-level languages (like Rust, C++, etc) and machine code.

Let's try it for this code instead, where we also pass `&x` to `opaque` to see
what happens when the compiler doesn't optimize that part away:

```c++
void opaque(int*);

int f(int* p) {
    int x = 1;
    opaque(p);
    opaque(&x);
    return x;
}
```

We can use the `-emit-llvm` flag on Godbolt to [see the LLVM
IR](https://godbolt.org/#g:!((g:!((g:!((h:codeEditor,i:(filename:'1',fontScale:14,fontUsePx:'0',j:1,lang:c%2B%2B,selection:(endColumn:2,endLineNumber:8,positionColumn:2,positionLineNumber:8,selectionStartColumn:2,selectionStartLineNumber:8,startColumn:2,startLineNumber:8),source:'void+opaque(int*)%3B%0A%0Aint+f(int*+p)+%7B%0A++++int+x+%3D+1%3B%0A++++opaque(p)%3B%0A++++opaque(%26x)%3B%0A++++return+x%3B%0A%7D'),l:'5',n:'0',o:'C%2B%2B+source+%231',t:'0')),k:44.33887569279493,l:'4',n:'0',o:'',s:0,t:'0'),(g:!((h:compiler,i:(compiler:clang2310,filters:(b:'0',binary:'1',binaryObject:'1',commentOnly:'0',debugCalls:'1',demangle:'0',directives:'0',execute:'1',intel:'0',libraryCode:'0',trim:'1',verboseDemangling:'0'),flagsViewOpen:'1',fontScale:14,fontUsePx:'0',j:1,lang:c%2B%2B,libs:!(),options:'-O2++-emit-llvm',overrides:!(),selection:(endColumn:29,endLineNumber:24,positionColumn:1,positionLineNumber:1,selectionStartColumn:29,selectionStartLineNumber:24,startColumn:1,startLineNumber:1),source:1),l:'5',n:'0',o:'+x86-64+clang+23.1.0+(Editor+%231)',t:'0')),k:55.66112430720507,l:'4',n:'0',o:'',s:0,t:'0')),l:'2',n:'0',o:'',t:'0')),version:4):

<pre><code>
define dso_local noundef i32 @f(int*)(ptr noundef <mark>%0</mark>) local_unnamed_addr #0 !dbg !14 {
  <mark>%2 = alloca i32, align 4</mark> // --> <mark>root of %2 allocation (x)</mark>
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

What the compiler does is to conceptually construct a DAG (directed acyclic
graph), to answer the question of where a pointer comes from and whether two
pointers can touch the same memory.

If the root origins don't match, LLVM can guarantee that writing through `%0`
will never alter the value stored at `%2`.

{{< mermaid-slider >}}
---
title: "Allocation Tree for x"
---
graph TD
    R2["Root: stack allocation (alloca)"]
    P2["%2 = alloca i32"]
    S["store i32 1, ptr %2"]
    C["call @opaque(ptr %2)"]
    L["load i32, ptr %2"]

    R2 --> P2
    P2 --> S
    P2 --> C
    P2 --> L
%%%
---
title: "Allocation Tree for p"
---
graph TD
    R1["Root: external allocation (belongs to the caller)"]
    P0["%0 (function parameter)"]
    C0["call @opaque(ptr %0)"]

    R1 --> P0
    P0 --> C0
{{< /mermaid-slider >}}

There is so much more to explore here that it's hard not to get carried away
because it's truly fascinating. However, my original goal was to compare how C++
and Rust handle this, so let's continue.

## C++: What Happens If We Cast to an Integer?

If we were to cast `&x` to an integer, then LLVM must assume that anyone could
rebuild a pointer to `x` from that integer.

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

The result is that the compiler gives up the optimization:

<iframe width="800px" height="200px" src="https://godbolt.org/e?hideEditorToolbars=true#g:!((g:!((g:!((h:codeEditor,i:(filename:'1',fontScale:14,fontUsePx:'0',j:1,lang:c%2B%2B,selection:(endColumn:2,endLineNumber:11,positionColumn:2,positionLineNumber:11,selectionStartColumn:2,selectionStartLineNumber:11,startColumn:2,startLineNumber:11),source:'%23include+%3Ccstdint%3E%0A%0Avoid+g(std::uintptr_t)%3B%0Avoid+opaque(int*)%3B%0A%0Aint+f(int*+p)+%7B%0A++++int+x+%3D+1%3B%0A++++g(reinterpret_cast%3Cstd::uintptr_t%3E(%26x))%3B+//+address+of+x+is+now+!'exposed!'%0A++++opaque(p)%3B%0A++++return+x%3B%0A%7D'),l:'5',n:'0',o:'C%2B%2B+source+%231',t:'0')),k:44.33887569279493,l:'4',n:'0',o:'',s:0,t:'0'),(g:!((h:compiler,i:(compiler:clang2310,filters:(b:'0',binary:'1',binaryObject:'1',commentOnly:'0',debugCalls:'1',demangle:'0',directives:'0',execute:'1',intel:'0',libraryCode:'0',trim:'1',verboseDemangling:'0'),flagsViewOpen:'1',fontScale:14,fontUsePx:'0',j:1,lang:c%2B%2B,libs:!(),options:'-O2+',overrides:!(),selection:(endColumn:1,endLineNumber:1,positionColumn:1,positionLineNumber:1,selectionStartColumn:1,selectionStartLineNumber:1,startColumn:1,startLineNumber:1),source:1),l:'5',n:'0',o:'+x86-64+clang+23.1.0+(Editor+%231)',t:'0')),k:55.66112430720507,l:'4',n:'0',o:'',s:0,t:'0')),l:'2',n:'0',o:'',t:'0')),version:4"></iframe>

That's a pity if we only wanted to read the address and never intended to cast
it back to a pointer. Wouldn't it be nice if we could communicate this intent to
the compiler?

## Rust: Making Your Intent Explicit

This is exactly what Rust provides with its powerful provenance APIs.

### Inspecting Without Exposing

```rust
let p: *const u32 = &x;
let a: usize = p.addr();
```

`p.addr()` extracts the raw memory address integer while telling Rust's abstract
machine:

> I am reading this numeric address for inspection, but I guarantee I WILL NOT
> cast it back to a valid pointer.

Provenance is therefore not exposed.

To mirror our previous experiments, we can try to see if the optimization was
avoided or not:

<iframe width="800px" height="200px" src="https://rust.godbolt.org/e?readOnly=true&hideEditorToolbars=true#g:!((g:!((g:!((h:codeEditor,i:(filename:'1',fontScale:14,fontUsePx:'0',j:1,lang:rust,selection:(endColumn:2,endLineNumber:14,positionColumn:2,positionLineNumber:14,selectionStartColumn:2,selectionStartLineNumber:14,startColumn:2,startLineNumber:14),source:'unsafe+extern+%22C%22+%7B%0A++++fn+g(addr:+usize)%3B%0A++++fn+opaque(p:+*mut+i32)%3B%0A%7D%0A%0A%23%5Bunsafe(no_mangle)%5D%0Apub+unsafe+fn+f_addr(p:+*mut+i32)+-%3E+i32+%7B%0A++++let+mut+x+%3D+1%3B%0A++++unsafe+%7B%0A++++++++g((%26raw+mut+x).addr())%3B%0A++++++++opaque(p)%3B%0A++++%7D%0A++++x%0A%7D'),l:'5',n:'0',o:'Rust+source+%231',t:'0')),k:50,l:'4',n:'0',o:'',s:0,t:'0'),(g:!((h:compiler,i:(compiler:nightly,filters:(b:'0',binary:'1',binaryObject:'1',commentOnly:'0',debugCalls:'1',demangle:'0',directives:'0',execute:'1',intel:'0',libraryCode:'0',trim:'1',verboseDemangling:'0'),flagsViewOpen:'1',fontScale:14,fontUsePx:'0',j:1,lang:rust,libs:!(),options:'-C+opt-level%3D3++-C+debuginfo%3D0',overrides:!((name:edition,value:'2024')),selection:(endColumn:1,endLineNumber:1,positionColumn:1,positionLineNumber:1,selectionStartColumn:1,selectionStartLineNumber:1,startColumn:1,startLineNumber:1),source:1),l:'5',n:'0',o:'+rustc+nightly+(Editor+%231)',t:'0')),k:50,l:'4',n:'0',o:'',s:0,t:'0')),l:'2',n:'0',o:'',t:'0')),version:4"></iframe>

What we see is that `mov eax, dword ptr [rsp + 12]` reloads `x` after
`opaque(p)`, meaning that, at least for now, the optimization is given up. Same
as in the C++ case.

Still, we can benefit from using the more expressive API, and not relying on
exposure semantics. It gives the compiler room to optimize in the future, and,
by telling Miri explicitly what we mean, it can better help us catch bugs.

### Preserving Provenance

<figure>
<img src="michael-myers-FvuisLAN-rA-unsplash.jpg" alt="">
  <figcaption>
Pointers reproduce like rabbits and someone has to keep track of the family
tree.

Photo by <a href="https://unsplash.com/@myers2021?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Michael Myers</a> on <a href="https://unsplash.com/photos/a-group-of-rabbits-sitting-on-top-of-a-pile-of-hay-FvuisLAN-rA?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
  </figcaption>
</figure>

In lock-free data structures, we sometimes need to store status flags in the
lower unused bits of a pointer. It's a very cool trick, which left me with my
eyes wide open for a minute when I first read about it. I talk more about it in
[this
post](https://sofiabelen.github.io/projects/visualizing-crossbeam-epoch-based-reclamation/#q2-dissecting-crossbeamepochshared).
Essentially, due to alignment, the lower bits of pointers are "wasted" (always
0), so we can use them to store flags.

What we can do is work with an intermediate pointer value, which will have the
flag bit set BUT that also inherits provenance from the original pointer. This
is exactly the functionality that `with_addr` provides us. After we're done, we
can recreate our valid initial pointer.

The tagged value is stays a pointer rather than 'usize' because an integer is
just a number and can't carry provenance.

```rust
let tagged: *const u32 = p.with_addr(p.addr() | 1);
// ...
let revived_ptr: *const u32 = tagged.with_addr(tagged.addr() & !1);
```

Keep in mind that `tagged` isn't dereferenceable, given that `p.addr() | 1` is
misaligned for `u32`. Dereferencing it would be UB.

In short:

- `addr()` reads the address without exposing provenance.

- `with_addr()` builds a new pointer with a new address but the *same*
  provenance as the original.

### Legacy Rust, Similar To C++

In legacy code, we'd have to do something like this:

```rust
let addr = p as usize;
let tagged_addr = addr | 1;
let revived_ptr = (tagged_addr & !1) as *const u32;
println!("*revived_ptr: {}", unsafe { *revived_ptr});
```

This code will run, but if we run it with Miri, it emits a warning:

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

Actually, the reason that it runs is also interesting. What happens is similar
to the C++ example. `p as usize` exposes the provenance of `p`, and since the
reconstructed pointer points to the same allocation, provenance is recovered.

Provenance would be lost if we did some pointer arithmetic and landed in a
different allocation.

### Casting Without Exposing Provenance

<figure>
<img src="dimitry-b-GNu4g4Z5jN4-unsplash.jpg" alt="">
  <figcaption>
Photo by <a href="https://unsplash.com/@dimitry_b?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Dimitry B</a> on <a href="https://unsplash.com/photos/brown-monkey-photograph-GNu4g4Z5jN4?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
  </figcaption>
</figure>

To see what happens when provenance is **not** exposed, we can use
`ptr::without_provenance`, which creates a bare address with no allocation
behind it. (It's useful for dangling or sentinel pointers, but it can't be
dereferenced).

```rust
let bare_ptr = std::ptr::without_provenance::<u32>(p.addr());
println!("*tagged_ptr: {}", unsafe { *bare_ptr});
```

As expected, Miri complains about a <mark>*dangling pointer*</mark> and tells us explicitly that <mark>it has no provenance.</mark>

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

## Recap

Provenance, in short, is a hidden piece of information tracking "which
allocation" a pointer originates from, which LLVM uses for optimizations.

In C++, `uintptr_t` casts force the optimizer to prepare for the worst case by
disabling some optimizations.

In Rust, `addr()`, `with_addr()` and `without_provenance()` let you state your
intent explicitly: you only want the number, or you are deriving a new pointer
from an existing one. In principle, that gives the compiler room to optimize
more aggressively. In our experiment, however, the Rust version is unoptimized
too: `x` was still reloaded from the stack. Using more expressive
APIs is still valuable though, as Miri can enforce the rules and catch bugs that
stem from unintended usage.

I had planned to compare what Miri and ASan can catch, but I'll save that for
another post. It was enough new info as it is and it takes some time to fully
wrap my head around it.

I'm currently between jobs and looking for a Rust role, so writing these posts
keeps me motivated and has helped me find a sense of community. Thank you for
reading, and I hope you enjoyed it!
