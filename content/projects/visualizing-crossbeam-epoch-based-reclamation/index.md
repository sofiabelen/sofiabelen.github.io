---
layout: project
title: "Visualizing crossbeam Epoch Based Reclamation"
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

On my previous post we experienced first hand the horrors that can occurr when attempting to use lock-free data structured with pointers. We left the question of how to deal with that a bit open. Languages with a Garbage Collector don't have this issue, since the runtime would delay collection until the memory address is unreachable by all threads. I recently learned that one of the ways to solve this in a deterministic manner is through somethting called Epoch-Based Reclamation (EBR), which Rust's crate `crossbeam-epoch` implements. My goal is not to explain the theoretical concepts, [as that endevour has been already accomplished by Aaron Turon, the founder of Crossbeam](https://aturon.github.io/blog/2015/08/27/epoch/#atomic). [Another pleasant read](https://codeandbitters.com/learning-rust-crossbeam-epoch/) I can recommend for a demonstration of the usage of `crossbeam-epoch`.

Instead, as I've done in previous posts, I want to dissect and visualize what exactly is going on. If that sounds fun, you're welcome to stick along for the ride!

You can access the [code](https://github.com/sofiabelen/visualizing-crossbeam-epoch) used in this post.

## Recap

We looked at the buggy implementation of a lock-free stack and actually managed to reproduce the ABA bug. Here's a diagram for a quick recap:

{{< mermaid-slider >}}
---
title: "Initial state"
---
flowchart TB
    head(("head"))

    subgraph nodea["node 'A'"]
        direction TB
        na_value["value: 1"]
        na_memory["memory: A"]
    end

    subgraph nodeb["node 'B'"]
        direction TB
        nb_value["value: 2"]
        nb_memory["memory: B"]
    end

    head --> nodea
    nodea --> nodeb
%%%
---
title: "Thread 1 reads head"
---
flowchart TB
    head(("head"))

    subgraph nodea["node 'A'"]
        direction TB
        na_value["value: 1"]
        na_memory["memory: A"]
    end

    subgraph nodeb["node 'B'"]
        direction TB
        nb_value["value: 2"]
        nb_memory["memory: B"]
    end

    t1["Thread 1<br/>head: A<br/>next: B"]

    head --> nodea
    nodea --> nodeb
    t1 --> nodea
%%%
---
title: "Thread 2 pops node 'A'"
---
flowchart TB
    head(("head"))

    subgraph nodeb["node 'B'"]
        direction TB
        nb_value["value: 2"]
        nb_memory["memory: B"]
    end

    t1["Thread 1<br/>head: A<br/>next: B"]

    subgraph nodea["node 'A' (popped)"]
        direction TB
        na_value["value: 1"]
        na_memory["memory: A"]
    end

    head --> nodeb
    nodeb ~~~ t1
    t1 --> nodea
%%%
---
title: "Thread 2 pops node 'B'"
---
flowchart TB
    head(("head (null)"))

    t1["Thread 1<br/>head: A<br/>next: B"]

    subgraph nodea["node 'A' (freed)"]
        direction TB
        na_value["value: 1"]
        na_memory["memory: A"]
    end

    head ~~~ t1
    t1 --> nodea
%%%
---
title: "Thread 2 pushes node 'C' (address A)"
---
flowchart TB
    head(("head"))

    subgraph nodec["node 'C'"]
        direction TB
        nc_value["value: 3"]
        nc_memory["memory: A"]
    end

    t1["Thread 1<br/>head: A<br/>next: B"]

    head --> nodec
    nodec ~~~ t1
    t1 --> nodec
%%%
---
title: "Thread 1 CAS succeeds: ABA bug triggered *happy noises*"
---
flowchart TB
    head(("head"))

    subgraph nodec["node 'C'"]
        direction TB
        nc_value["value: 3"]
        nc_memory["memory: A"]
    end

    t1["Thread 1 CAS<br/>head == A? true<br/>set head = B"]

    subgraph nodeb["node 'B' (freed)"]
        direction TB
        nb_value["value: 2"]
        nb_memory["memory: B"]
    end

    head --> nodec
    nodec ~~~ t1
    t1 --> nodec
    t1 ~~~ nodeb
%%%
---
title: "Final state (dangling head)"
---
flowchart TB
    head(("head (dangling)"))

    subgraph nodeb["node 'B' (freed)"]
        direction TB
        nb_value["value: 2"]
        nb_memory["memory: B"]
    end

    subgraph nodec["node 'C' (popped)"]
        direction TB
        nc_value["value: 3"]
        nc_memory["memory: A"]
    end

    head --> nodeb
    nodeb ~~~ nodec
{{< /mermaid-slider >}}

The fundamental question is:

> 1. How do we defer `drop()` until Thread A is no longer reading it, without adding locks or atomic reference counter updates on every single read?

Another important question also comes up, for example, with linked-lists, when removing a node requires two distinct steps. We need to first mark it as deleted (so other threads don't try to insert after it) and then physically unlink it via CAS. There's a limit set by the hardware that normal CPU atomic instructions can only swap one word (64 bits) at a time. We can't swap a `next` pointer and a separate `is_deleted: bool` atomically. This raises a second question, namely:

> 2. How do we atomically update both a pointer and a state flag in a single CAS instruction, without a double-word atomic write?

The goal is to dissect `crossbeam-epoch` to better understand how it solves these two questions.

## Dissecting crossbeam::epoch::Shared<T> (Answers Question 2)

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

TODO!!!!

```rust
let node = Owned::new(Node { data: 42 } );
let atomic_ptr = Atomic::from(node);

let guard = &epoch::pin();

let shared_ptr = atomic_ptr.load(Ordering::Acquire, guard);
```

Now we can play around with it, changing the tag and verifying what happens in memory:

```rust
 let node = Owned::new(Node { data: 42 } );
 let atomic_ptr = Atomic::from(node);

 let guard = &epoch::pin();

 let shared_ptr = atomic_ptr.load(Ordering::Acquire, guard);

 // 1. Initial State (Tag 0)
 println!("Initial tag: {}", shared_ptr.tag());
 println!("Initial address: {:#018x}        --> last byte: {:08b}", shared_ptr.as_raw() as usize, shared_ptr.as_raw() as usize & 0xFF);
 println!("----------------------------------");

 // 2. Set tag to 2
 // with_tag creates a new Shared pointer with updated tag bits
 let tagged_ptr_2 = shared_ptr.with_tag(2);
 let raw_bits_2 = unsafe { std::mem::transmute::<_, usize>(tagged_ptr_2) };

 println!("Tag: {}", tagged_ptr_2.tag());
 println!("Tagged address (tag 2): {:#018x} --> last byte: {:08b}", &raw_bits_2, &raw_bits_2 & 0xFF);
 println!("tagged_ptr_2.as_raw(): {:#018x}", tagged_ptr_2.as_raw() as usize);
 println!("----------------------------------");

 // 3. Update tag to 3
 let tagged_ptr_3 = tagged_ptr_2.with_tag(3);
 let raw_bits_3 = unsafe { std::mem::transmute::<_, usize>(tagged_ptr_3) };

 println!("Tag: {}", tagged_ptr_3.tag());
 println!("Tagged address (tag 3): {:#018x} --> last byte: {:08b}", &raw_bits_3, &raw_bits_3 & 0xFF);
 println!("tagged_ptr_3.as_raw(): {:#018x}", tagged_ptr_3.as_raw() as usize);
 println!("----------------------------------");

 // 4. Verify Dereference Ignores Tag Bits
 let node_ref: &Node = unsafe { tagged_ptr_3.deref() };
 let physical_addr = (node_ref as *const Node) as usize;

 println!("Clean Physical Address: {:#018x}", physical_addr);
 println!("Node inner data: {}", node_ref.data);
```

Let's run it:
<pre><code>
Initial tag: 0
Initial tag: 0
Initial address: 0x000056065369dd5<mark>0</mark>        --> last byte: 010100<mark>00</mark>
----------------------------------
Tag: 2
Tagged address (tag 2): 0x000056065369dd5<mark>2</mark> --> last byte: 010100<mark>10</mark>
tagged_ptr_2.as_raw(): 0x000056065369dd50
----------------------------------
Tag: 3
Tagged address (tag 3): 0x000056065369dd5<mark>3</mark> --> last byte: 010100<mark>11</mark>
tagged_ptr_3.as_raw(): 0x000056065369dd50
----------------------------------
Clean Physical Address: 0x000056065369dd50
Node inner data: 42
</code></pre>

This answers our second question, let's now tackle the first one:

> 1. How do we defer `drop()` until Thread A is no longer reading it, without adding locks or atomic reference counter updates on every single read?
>
> 2. ~~How do we atomically update both a pointer and a state flag in a single CAS instruction, without a double-word atomic write?~~

## The Global & Local Clockwork: How Epochs Move (Answers Question 1: Part A)

## Memory Inspection: Tracking a Node's Lifecycle (Answers Question 1: Part B)

## 
