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

As a newly-Rust convert coming from C++, it can be all too tempting to believe Rust's ownership model is the panacea, the cure of all our ailments. While it is truly groundbreaking, I'm beginning to learn about the cases where it can't really save us. But there's hope, let's not panic!

## The Concurrency Dilemma: Why drop() Breaks in Lock-Free Code

One of those cases is lock-free code.

Normally, every value has a single owner, and when that owner goes out of scope, Rust calls `drop()` to first execute its desctructor and then free its memory. If we're using a `Mutex`, we can rest assured that while one thread is mutating or destroying a node in our data structure, no other thread can hold a reference to it. 

If we want to do this the lock-free way, we'd use a CAS (Compare-And-Swap) loop instead of a Mutex. This way, multiple threads could read and modify different nodes in data structure simultanously, without waiting for one another.

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

The fundamental question is:

> How do we defer `drop()` until Thread A is no longer reading it, without adding locks or atomic reference counter updates on every single read?

### The ABA Problem


### The fix 

Languages with a Garbage Collector don't have this issue, since the runtime would delay collection until `0x1000` is unreachable by all threads.

I recently discovered that one of the ways to solve this in a deterministic manner is through somethting called Epoch-Based Reclamation (EBR), which Rust's crate Crossbeam implements.

## Dissecting crossbeam::epoch::Shared<T>

- `Box<T>`: there must be a single owner and it cannot be safely shared across 
- `&T` requires compile-time checks
- `Arc<T>`: reference counting

## The Global & Local Clockwork: How Epochs Move

## Memory Inspection: Tracking a Node's Lifecycle

## 
