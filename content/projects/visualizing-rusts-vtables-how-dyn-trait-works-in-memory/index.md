---
layout: project
title: "Visualizing Rust's Vtables: How dyn Trait Works In Memory"
thumbnail: Puccinia_graminis_149593622.jpg
image: Puccinia_graminis_149593622.jpg
preview: ""
description: "A follow-along series of coding experiments as I explore Rust's peculiarities, from a C++ developer's perspective."
date: 2026-04-20
publishDate: 2026-04-20
tags:
    - Rust
    - tutorial
categories:
    - Rust
    - tutorial
featured: true
comments: true
draft: true
---

I'm venturing into Rust and it's both satisfying and mind-boggling at the same time. So far I've been learning from [the book](https://doc.rust-lang.org/book/), but I got the itch to do some dissecting myself. The goal of these experiments is to compare Rust's approach to polymorphism with C++'s. If you're like me and want to visualize *what exactly is happening in memory,* in order to feel like you truly understand the concepts, hopefully you'll find this post useful :)

By the way, the thumbnail image is a photo of the rust fungus, to which we owe Rust's name. ([Puccinia graminis 149593622Related image](https://commons.wikimedia.org/wiki/File:Puccinia_graminis_149593622.jpg) by [bjoerns](https://www.inaturalist.org/users/188406) is licensed under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/))

You can find all the code and experiments on [GitHub](https://github.com/sofiabelen/rust-vtables-for-cpp-programmers).

## Introduction: The Crux of the Matter

What we're trying to achieve is quite simple. Let's say we have a bunch of shapes: circles, squares, triangles, and we want to call `draw()` on each one.  

### C++ Approach #1: Virtual Functions

The first way to do this that comes to mind in C++ is through virtual functions, which makes use of runtime polymorphism. The vtable pointer lives inside the object, virtual dispatch happens automatically.

```cpp
std::vector<Shape*> shapes = { new Circle(), new Square() };
for (auto* s : shapes)
    s->draw();
```

Rust's equivalent would be `dyn Trait`, which is what we ultimately want to understand. But first, let's take a look at another way we could solve this in C++.

### C++ Approach #2: CRTP

One could also go the CRTP (Curiously Recurring Template Pattern) route, which is essentially compile time polymorphism. If you're interested, this awesome [talk](https://www.youtube.com/watch?v=pmdwAf6hCWg) by Klaus Iglberger is what helped this concept finally click for me.

```cpp
template<typename Derived>
struct Shape {
    void draw() {
        static_cast<Derived*>(this)->draw();
    }
};
```

Essentially, there are no vtables and it's resolved at compile time, sacrificing readability (it really is a mouthful).

Rust offers a much more straightforward and simple equivalent to CRTP, namely monomorphization. This is the approach we'll dig into first to start constructing our mental model of what Rust has to offer.

## Static Dispatch

- static dispatch, also known as generics
- the compiler creates separate copies for each concrete time
- zero runtime cost
- types must be known at compile time
- when does a problem arise?

Comparisons to c++:
- if you used templates in C++, the template would take any type T that implements draw
- (with concepts you can narrow down)
- in rust, the types accepted are explicitely what you specified
- more intentional

```rust
trait Draw {
    fn draw(&self) -> &str;
}

struct Circle;
struct Square;

impl Draw for Circle {
    fn draw(&self) -> &str {
        "Drawing a circle"
    }
}

impl Draw for Square {
    fn draw(&self) -> &str {
        "Drawing a square"
    }
}

fn draw_shape<T: Draw>(shape: T) {
    println!("{}", shape.draw());
}

fn main() {
    let circle = Circle;
    let square = Square;
    draw_shape(circle);
    draw_shape(square);
}
```

### Side Quest: Rust's Zero-Sized Types

I tried to look at the size of `Circle` and `Square` without thinking much of it, but the result left me puzzled at first. In C++, the standard mandates that everyobject has a size of at least 1 byte, even if empty. This is so that two distinc objects always have distinct address, meaning `&obj1' must be different from `&obj2`. This is ingrained in my mind as a fact of nature, so seeing that rust returns `0` as the size of those structs stopped me in my tracks. This is the little moments that bring me so much joy as I'm exploring Rust because it deconstructs my mental model and opens my mind to alternative philosophies.

```rust
println!("{}", std::mem::size_of::<Circle>()); // 0 WHAT???
println!("{}", std::mem::size_of::<Square>()); // 0
```

This is how I discovered that Rust handles the unique address guarantee differently. Zero-sized types (ZST) are structs that don't contain any fields, therefore there's no need to allocate any memory. Rust track identity through owndership, not addresses. Every value has exactly one owner at a time, this is enforced at compile time by our friend,the borrow-checker.

In C++, we might do this to check if two pointers refeer to the same object:

```cpp
if (&a == &b) { // same object }
```

In Rust, that question is answered by the borrow-checker before the program even runs:

```rust
// the borrow checker already knows these are different bindings
// you don't need to compare addresses to tell them apart
let a = Circle;
let b = Circle;
```

The compiler tracks `a` and `b` as distinct names with distinct owners.

After learning about this, my question was, what happens then if we take the address of of a zst? Let's try it.

```rust
let a = Circle;
let b = Circle;

println!("{:p}", &a as *const Circle);
println!("{:p}", &b as *const Circle);
```

```
0x7ffdda99aece
0x7ffdda99aecf
```

Strange... so they are getting distinct stack addresses 1 byte apart (`ce` and `cf` in hex). This means the compiler allocated 1 byte each for them, just like C++ would. This is just the compiler being pragmatic though. It turns out the compiler gives ZSTs a small stack slot anyway so that the debugger can refer to them meaningfully.

If we try this in release mode, however, we get different behavior:

```
cargo run --release --bin 01_static_dispatch
```

The addresses do indeed collapse for me:

```
0x7ffdf74afa6f
0x7ffdf74afa6f
```

My take away from this is that the compiler makes no guarantees about ZST addresses, and identity is tracked by the borrow checker through ownership, not memory addresses.

## Dynamic Dispatch

To continue on our main quest, let's explore an scenario where static dispatch can't help us. If we wanted to create a `Vec<T>` containing a mix of both `Square`'s and `Circle`'s, we hit a wall with generics alone. This is the reason of being for dynamic dispatch, aka `dyn Trait`.

Let's see how it would look like with our simple example:

```rust
```

A key distinction from C++ philosophy is that in C++, the choice is made at the class level. The moment you mark a method `virtual`, that class always uses dynamic dispatch. You can't opt out.

In Rust, the choice is made at the *call site.* `Circle` is just `Circle`, it knows nothing about dispatch. You decide whether to use static or dynamic dispatch depending on how you refer to it: `&Circle` for static, `&dyn Draw` for dynamic.

## Wide Pointers


