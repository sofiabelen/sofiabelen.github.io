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
