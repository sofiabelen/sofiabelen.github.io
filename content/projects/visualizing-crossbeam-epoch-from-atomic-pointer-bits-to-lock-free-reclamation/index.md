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
featured: true
comments: true
---

As a newly-Rust convert (Rustacean?) coming from C++, it can be all too tempting to believe Rust's ownership model is the panacea, the cure of all our ailments. While it is truly groundbreaking, I'm beginning to learn about the cases where it can't really save us. But there's hope, let's not panic!

Ever since I watched (Fedor Pikus talk on atomics)[https://www.youtube.com/watch?v=ZQFzMfHIxng] (highly recommend!), I've had an interest in lock-free programming. I played around with implementing my own SPMC queue in C++, and have since started and not finished some books, trying specially hard to wrap my head around memory ordering. But I've come to realize I've been a bit stuck in tutorial hell, so this post is the beginning of me getting unstuck.

My goal is to poke at the [crossbeam crate](https://github.com/crossbeam-rs/crossbeam), with the hopes of gaining a better understanding of a real-world usecase and hopefully gain some insight which might help me contribute something back.

Let's start from the beginning, what is (one of) the problem(s) being solved.

## Why drop() Breaks in Lock-Free Code

It turns out, one of the cases where Rust's ownership model isn't enough is lock-free code.

Normally, every value has a single owner, and when that owner goes out of scope, Rust calls `drop()` to first execute its desctructor and then free its memory. If we're using a `Mutex`, we can rest assured that while one thread is mutating or destroying a node in our data structure, no other thread can hold a reference to it. 

If we want to do this the lock-free way, we'd use a CAS (Compare-And-Swap) loop instead of a Mutex. This way, multiple threads could read and modify different nodes in data structure simultaneously, without waiting for one another.

### The Use-After-Free Race Condition

```rust
pub unsafe fn pop(&self) -> Option<T> {
    loop {
        let head_ptr = self.head.load(Ordering::Acquire); // 1. Read Head
        if head_ptr.is_null() { return None; }
        
        let next_ptr = (*head_ptr).next; // 2. Dereference Head!
        
        if self.head.compare_exchange(head_ptr, next_ptr, ...).is_ok() {
            // 3. Popped successfully!
            let value = ptr::read(&(*head_ptr).value);
            
            // If we drop/free head_ptr HERE, any other thread paused at Step 2
            // will crash when it wakes up and tries to read (*head_ptr).next!
            deallocate(head_ptr); 
            return Some(value);
        }
    }
}
```

This creates a conflict though. Let's imagine the following scenario on our lock-free stack:

1. Thread 1 starts `pop()`: it reads the shared `head` pointer and sees it points to Node `A` (address `0x1000`).
2. Thread 1 get's context-switched before it can execute its CAS step (just before it can read `Node A.next`.
3. Thread 2 runs `pop()`:
    1. Reads `head` (`0x1000`, Node A).
    2. Reads `Node A.next` (`0x2000`, Node B).
    3. Succesfully executes CAS: `head` nos points to Node B.
4. Thread 2 runs `drop()` for Node A. Since thread 2 "owns" the popped Node A, it immediately calls `drops()`. Node A's destructor runs, and its memory at `0x1000` is returned to the allocator.
5. Thread 1 resumes and attempts to now read `Node A.next`, so it tries to dereference pointer `0x1000`.

**Result:** Memory at `0x1000`  has already been freed or repurposed, which leads to undefined behaviour or segmentation fault (use-after-free).


<figure>
<img src="william-dmytrow-pS6GsfrQZDk-unsplash.jpg" alt="">
<caption>
Photo by <a href="https://unsplash.com/@williamdmytrow?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">William Dmytrow</a> on <a href="https://unsplash.com/photos/winged-victory-of-samothrace-marble-statue-on-a-ship-prow-pS6GsfrQZDk?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
</caption>
</figure>

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
