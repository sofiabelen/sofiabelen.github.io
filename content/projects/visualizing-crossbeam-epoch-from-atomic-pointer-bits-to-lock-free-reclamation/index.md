---
layout: project
title: "Visualizing crossbeam-epoch: From Atomic Pointer Bits to Lock-Free Reclamation"
thumbnail: abhishek-ravi-ujPDUtsCdl0-unsplash.jpg
image: abhishek-ravi-ujPDUtsCdl0-unsplash.jpg
preview: ""
description: ""
date: 2026-09-09
publishDate: 
tags:
    - Rust
    - tutorial
    - lock-free-programming
    - concurrency
categories:
    - Rust
    - tutorial
    - lock-free-programming
    - concurrency
featured: false
draft: true
comments: true
---

The fundamental question is:

> How do we defer `drop()` until Thread A is no longer reading it, without adding locks or atomic reference counter updates on every single read?

### The fix 

Languages with a Garbage Collector don't have this issue, since the runtime would delay collection until `0x1000` is unreachable by all threads.

I recently discovered that one of the ways to solve this in a deterministic manner is through somethting called Epoch-Based Reclamation (EBR), which Rust's crate Crossbeam implements. My goal is not to explain the theoretical concepts, [as that endevour has been already accomplished by Aaron Turon, the founder of Crossbeam](https://aturon.github.io/blog/2015/08/27/epoch/#atomic). Instead, as I've done in previous posts, I want to dissect and visualize what exactly is going on. If that sounds fun, you're welcome to stick along for the ride!

## Dissecting crossbeam::epoch::Shared<T>

- `Atomic<T>`
- `Owned<T>`
- `Shared<'g, T>`

### Using Wasted Alignment Bits for Metadata

<figure>
<img src="eduard-delputte-1BDdWC9CEQw-unsplash.jpg" alt="">
<caption>
Photo by <a href="https://unsplash.com/@edelputte?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Eduard Delputte</a> on <a href="https://unsplash.com/photos/kitten-sits-on-shelf-with-cameras-and-plants-1BDdWC9CEQw?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
</caption>
</figure>

Every type has an alignment requirement. We can imagine that memory is organized in "shelves," where each variable must sit. This way, you can't have a variable sitting in between two shelves. Each type has it's own alignemnt (shelf size) requirements. In modern 64-bit architectures, a pointer requires 8-byte alignnment.

This means that heap memory allocated for that type will start at an address that is an exact multiple of 8. If you look at memory addresses taht are multiples of 8 in binary, you can see that the lowest 3 bits are in a sense "wasted." They don't hold any other useful information, since they're guaranteed to be always `000`. 

| Decimal Address | Hex Address | 64-bit Binary Address (Lowest Byte) |
|---|---|---|
| 16 | 0x0010 | ... 0001 0000 |
| 24 | 0x0018 | ... 0001 1000 |
| 32 | 0x0020 | ... 0010 0000 |
| 40 | 0x0028 | ... 0010 1000 |

What crossbeam does is reclaim those unused low-order bits to store some metadata directly inside the pointer value.

For reading or dereferencing the pointer, crossbeam masks out those bits using bitwise `AND` to reconstruct the valid address:

```
Original Address = Raw Pointer & ~ Tag Mask
```

This is awesome because we can store some metadata that will allow us to verify whether that memory is ready to be deallocated, without having to use a separate variable.

### Taking a Look Inside

## The Global & Local Clockwork: How Epochs Move

## Memory Inspection: Tracking a Node's Lifecycle

## 
