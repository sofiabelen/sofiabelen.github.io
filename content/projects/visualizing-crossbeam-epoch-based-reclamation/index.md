---
layout: project
title: "Visualizing crossbeam Epoch Based Reclamation"
thumbnail: thomas-tygreat-9cyrW-OH04A-unsplash.jpg
image: thomas-tygreat-9cyrW-OH04A-unsplash.jpg
preview: ""
description: ""
date: 2026-09-21
publishDate: 2026-09-20
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

On my previous post we experienced first hand the horrors that can occurr when attempting to use lock-free data structured with pointers. We left the question of how to deal with that a bit open.

Languages with a Garbage Collector don't have this issue, since the runtime would delay collection until the memory address is unreachable by all threads. I recently learned that one of the ways to solve this in a deterministic manner is through somethting called Epoch-Based Reclamation (EBR), which Rust's crate `crossbeam-epoch` implements. My goal is not to explain the theoretical concepts, [as that endevour has been already accomplished by Aaron Turon, the founder of Crossbeam](https://aturon.github.io/blog/2015/08/27/epoch/#atomic). Also, [another pleasant read](https://codeandbitters.com/learning-rust-crossbeam-epoch/) I can recommend for a demonstration of the usage of `crossbeam-epoch`.

Instead, as I've done in previous posts, I want to dissect and visualize what exactly is going on. If that sounds fun, you're welcome to stick along for the ride!

You can access the [code](https://github.com/sofiabelen/visualizing-crossbeam-epoch) used in this post.

## Q1: When to Drop

We looked at the buggy implementation of a lock-free stack and actually managed to reproduce the ABA bug. Here's a diagram for a quick recap:

<figure class="diagram-figure">

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

<figcaption>
  Lock-free stack ABA problem during concurrent push/pop operations.
</figcaption>
</figure>

The fundamental question is:

> 1. How do we defer `drop()` until Thread A is no longer reading it, without adding locks or atomic reference counter updates on every single read?

## Q2: Updating Pointer and Flag Atomically

Another important question also comes up, with linked-lists for example, when removing a node requires two distinct steps. Check this out [Timothy Harris' Paper](https://timharris.uk/papers/2001-disc.pdf) for a more detailed explanation of this algirthm. Essentially, deleting is done in two parts:

1. Mark the node as deleted (or more specifically, it's next pointer) so other threads don't try to insert after it.
2. Physically unlink it via CAS.

There's a limit set by the hardware that normal CPU atomic instructions can only swap one word (64 bits) at a time. We can't swap a `next` pointer and a separate `is_deleted: bool` flag atomically. This raises a second question, namely:

> 2. How do we atomically update both a pointer and a state flag in a single CAS instruction, without a double-word atomic write?

<figure>
{{< mermaid-slider >}}
---
title: "Step 0: Initial state"
---
flowchart TB
    head(("head"))

    subgraph nodea["node 'A'"]
        direction TB
        na_val["value: 1"]
        na_next["next: B (flag: 0)"]
    end

    subgraph nodeb["node 'B'"]
        direction TB
        nb_val["value: 2"]
        nb_next["next: C (flag: 0)"]
    end

    subgraph nodec["node 'C'"]
        direction TB
        nc_val["value: 3"]
        nc_next["next: null"]
    end

    head --> nodea
    nodea --> nodeb
    nodeb --> nodec
%%%
---
title: "Step 1: Thread 1 reads node 'B' and its next pointer"
---
flowchart TB
    head(("head"))

    subgraph nodea["node 'A'"]
        direction TB
        na_val["value: 1"]
        na_next["next: B (flag: 0)"]
    end

    subgraph nodeb["node 'B'"]
        direction TB
        nb_val["value: 2"]
        nb_next["next: C (flag: 0)"]
    end

    subgraph nodec["node 'C'"]
        direction TB
        nc_val["value: 3"]
        nc_next["next: null"]
    end

    t1["Thread 1<br/>prepares to delete `B`, reads:<br/>curr: B<br/>next: C [0]"]

    head --> nodea
    nodea --> nodeb
    nodeb --> nodec
    t1 --> nodeb
%%%
---
title: "Step 2: Logical deletion, setting B.next's flag to 1"
---
flowchart TB
    head(("head"))

    subgraph nodea["node 'A'"]
        direction TB
        na_val["value: 1"]
        na_next["next: B (flag: 0)"]
    end

    subgraph nodeb["node 'B' (marked as deleted)"]
        direction TB
        nb_val["value: 2"]
        nb_next["next: C (flag: 1)"]
    end

    subgraph nodec["node 'C'"]
        direction TB
        nc_val["value: 3"]
        nc_next["next: null"]
    end

    t1["Thread 1 CAS on B.next:<br/>expected: C (flag: 0)<br/>new: C (flag: 1)<br/>Result: SUCCESS"]

    head --> nodea
    nodea --> nodeb
    nodeb --> nodec
    t1 --> nodeb
%%%
---
title: "Step 3: Concurrent insert fails"
---
flowchart TB
    subgraph nodeb["node 'B' (marked as deleted)"]
        direction TB
        nb_val["value: 2"]
        nb_next["next: C (flag: 1)"]
    end

    subgraph noded["node 'D' (new node)"]
        direction TB
        nd_val["value: 99"]
        nd_next["next: C"]
    end

    t2["Thread 2 tries to insert 'D' after 'B':<br/>CAS expects B.next == C (flag: 0)<br/>Actual B.next == C (flag: 1)<br/>Result: FAILS"]

    nodeb ~~~ t2
    t2 --> noded
%%%
---
title: "Step 4: Physical unlink"
---
flowchart TB
    head(("head"))

    subgraph nodea["node 'A'"]
        direction TB
        na_val["value: 1"]
        na_next["next: C (flag: 0)"]
    end

    subgraph nodeb["node 'B' (unlinked)"]
        direction TB
        nb_val["value: 2"]
        nb_next["next: C (flag: 1)"]
    end

    subgraph nodec["node 'C'"]
        direction TB
        nc_val["value: 3"]
        nc_next["next: null"]
    end

    t1["Thread 1 CAS on A.next:<br/>expected: B (flag: 0)<br/>new: C (flag: 0)<br/>Result: SUCCESS"]

    head --> nodea
    nodea --> nodec
    nodec ~~~ nodeb
    t1 --> nodea
%%%
---
title: "Step 5: Defer destroy"
---
flowchart TB
    head(("head"))

    subgraph nodea["node 'A'"]
        direction TB
        na_val["value: 1"]
        na_next["next: C (flag: 0)"]
    end

    subgraph nodec["node 'C'"]
        direction TB
        nc_val["value: 3"]
        nc_next["next: null"]
    end

    subgraph nodeb["node 'B' (retired node)"]
        direction TB
        nb_val["value: 2"]
        nb_next["next: C (flag: 1)"]
    end

    t1_bag["Thread 1 local bag<br/><code>guard.defer_destroy(B)</code>"]

    head --> nodea
    nodea --> nodec
    nodec ~~~ t1_bag
    t1_bag -->|enqueues| nodeb
{{< /mermaid-slider >}}
<figcaption>
Deleting a node from a lock-free linked list.
</figcaption>
</figure>

## Motivation and Goal

When I started writing this, I had planned for just one post visualizing the ABA bug and then analyzing how crossbeam solves it. However, I quickly realized that the ABA problem deserved its [own post](https://sofiabelen.github.io/projects/visualizing-the-aba-problem/). Then, as I was writing this "part 2" if you will, I intially just cared about **Q1**. However, I came accross something that I found really awesome (don't want to spoil the next session), and I wondered *why* do we need it... so that's how I came to **Q2**, and I couldn't just leave it out. In the mean time, I was playing around with loom, so that became its own post. So far in the series:

- **Part 1:** [Visualizing The ABA Problem: What crossbeam-epoch Solves](https://sofiabelen.github.io/projects/visualizing-the-aba-problem/)
- **Part 2:** [Using Loom to (Try To) Catch an ABA Bug in Lock-Free Rust](https://sofiabelen.github.io/projects/using-loom-to-catch-an-aba-bug/)
- **Part 3:** current ;)

Without further ado, the goal of this post is to dissect `crossbeam-epoch` to better understand how it solves **Q1** and **Q2**.

## Q2: Dissecting crossbeam::epoch::Shared<T>

To give a refresher, the building blocks of `crossbeam-epoch` are:

| crossbeam type | analogue | role |
| :--- | :--- | :--- |
| `Owned<T>` | `Box<T>` | Unique, non-atomic ownership. |
| `Atomic<T>` | `AtomicPtr<T>` | Shared, atomic location. |
| `Shared<'g, T>` | `&'g T` | Borrowed reference tied to a `Guard`.  |


The `Owned<T>` holds heap allocated data that isn't shared yet. You turn an `Owned` into a `Shared` or store it inside an `Atomic`.

`Atomic<T>` is what you'll use in your data structures. It holds a pointer that can be updated using CAS (Compare-And-Swap) or other atomic operations.

`Shared<'g, T>` is obtained by calling `.load(&guard)` on an `Atomic`. The lifetime `'g` guarantees the pointer cannot be freed by another thread as long as your `pin()` guard is alive. (We'll discuss this later).


### Using Wasted Alignment Bits for Metadata

<figure>
<img src="eduard-delputte-1BDdWC9CEQw-unsplash.jpg" alt="">
<caption>
Photo by <a href="https://unsplash.com/@edelputte?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Eduard Delputte</a> on <a href="https://unsplash.com/photos/kitten-sits-on-shelf-with-cameras-and-plants-1BDdWC9CEQw?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
</caption>
</figure>

Every type has an alignment requirement. We can imagine that memory is organized in "shelves," where each variable must sit. This way, you can't have a variable sitting in between two shelves. Each type has it's own alignemnt (shelf size) requirement. In modern 64-bit architectures, a pointer requires 8-byte alignnment.

This means that heap memory allocated for that type will start at an address that is an exact multiple of 8. If you look at memory addresses taht are multiples of 8 in binary, you can see that the lowest 3 bits are in a sense "wasted." They don't hold any other useful information, since they're guaranteed to be always `000`. 

| Decimal Address | Hex Address | 64-bit Binary Address (Lowest Byte) |
|---|---|---|
| 16 | 0x0010 | ... 0001 0000 |
| 24 | 0x0018 | ... 0001 1000 |
| 32 | 0x0020 | ... 0010 0000 |
| 40 | 0x0028 | ... 0010 1000 |

You can believe this was my face when I learned what crossbeam does with this information.

<figure style="display: flex; flex-direction: column; align-items: center; text-align: center; margin: 0 auto;">
  <img style="max-height: 70%; max-width: 70%; height: auto; width: auto;" src="hang-niu-Tn8DLxwuDMA-unsplash.jpg" alt="White kitten">
  <figcaption style="margin-top: 8px;">
    Photo by <a href="https://unsplash.com/@niuhang?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">hang niu</a> on <a href="https://unsplash.com/photos/white-kitten-Tn8DLxwuDMA?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
  </figcaption>
</figure>

<mark>What crossbeam does is reclaim those unused low-order bits to store some metadata directly inside the pointer value.</mark>

For reading or dereferencing the pointer, crossbeam masks out those bits using bitwise `AND` to reconstruct the valid address:

```
Original Address = Raw Pointer & ~ Tag Mask
```

This is purely awesome. We can store some metadata that will allow us to verify whether that memory is ready to be deallocated, without having to use a separate variable.

### Taking a Look Inside

To work with `crossbeam-epoch`, we first allocate data on the heap using `Owned<T>` (similar to `Box<T>`). We then transfer this pointer into an `Atomic<T>` location. This enables us to share it across threads.

Before reading from the `Atomic<T>`, a thread must `pin` the current global epoch. We'll discuss this in a bit, but basically we're announcing that this thread is actively working with memory, and for as long as this `Guard` lives, other threads aren't allowed to drop the object(s) we're working with. This guaranntees that any `Shared<'a, T>` references we have remain safe to dereference.

```rust
let node = Owned::new(Node { data: 42 } );
let atomic_ptr = Atomic::from(node);

let guard = &epoch::pin();

let shared_ptr = atomic_ptr.load(Ordering::Acquire, guard);
```

Now that we have our `Shared<'a, T>` we can play around with it, changing the tag and verifying what happens in memory:

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

## Q1: The Global & Local Clockwork: How Epochs Move

## Q1: Memory Inspection: Tracking a Node's Lifecycle

## Conclusion
