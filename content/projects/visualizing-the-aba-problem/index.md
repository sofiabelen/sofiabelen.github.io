---
layout: project
title: "Visualizing The ABA Problem: What crossbeam-epoch Solves"
thumbnail: Janus-statue-and-his-two-faces-past-and-future.webp
image: Janus-statue-and-his-two-faces-past-and-future.webp
preview: "Visualizing lock-free concurrency in rust: reproducing the aba problem to begin understanding crossbeam. I promise lots of diagrams!"
description: "Visualizing lock-free concurrency in rust: reproducing the aba problem to begin understanding crossbeam. I promise lots of diagrams!"
date: 2026-09-18
publishDate: 2026-09-17
tags:
    - Rust
    - tutorial
    - lock-free-programming
    - concurrency
    - crossbeam
categories:
    - Rust
    - tutorial
    - lock-free-programming
    - concurrency
    - crossbeam
featured: true
comments: true
---

As a newly-Rust convert (Rustacean?) coming from C++, it can be all too tempting to believe Rust's ownership model is the panacea, the cure of all our problems. While it is truly groundbreaking, I'm beginning to learn about the cases where it can't really save us. But there's hope, let's not panic!

Ever since I watched [Fedor Pikus talk on atomics](https://www.youtube.com/watch?v=ZQFzMfHIxng) (highly recommend!), I've had an interest in lock-free programming. I played around with implementing my own SPMC queue in C++, and have since started (and not finished) some books, trying specially hard to wrap my head around memory ordering. But I've come to realize I've been a bit stuck in tutorial hell, so this post is the beginning of me getting unstuck.

My goal is to poke at the [crossbeam crate](https://github.com/crossbeam-rs/crossbeam), with the hopes of gaining a better understanding of a real-world usecase and hopefully gain some insight which might help me contribute something back.

Let's start from the beginning, exploring what is one of the problems that crossbeam-epoch tackles.

See all the code and experiments over at my [GitHub](https://github.com/sofiabelen/visualizing-crossbeam-epoch).

> Side note: the two-faced statue in the thumbnail is the Roman god Janus, who looks toward the past and future simultaneously. In a moment you'll see what this has to do with our pointers in the ABA problem.

## The Question of When to drop() in Lock-Free Code

It turns out, one of the cases where Rust's ownership model isn't enough is lock-free code.

Normally, every value has a single owner, and when that owner goes out of scope, Rust calls `drop()` to first execute its desctructor and then free its memory. If we're using a `Mutex`, we can rest assured that while one thread is mutating or destroying a node in our data structure, no other thread can hold a reference to it. 

If we want to do this the lock-free way, we'd use a CAS (Compare-And-Swap) loop instead of a Mutex. This way, multiple threads could read and modify different nodes in data structure simultaneously, without waiting for one another.

## What CAS Guarantees

So what does CAS actually promise?

```rust
fn compare_exchange(
    &self,
    current: T,
    new: T,
    success: Ordering,
    failure: Ordering,
) -> Result<T, T>;
```

In simple terms, what is says is: "if the current value equals `current`, swap it for the new one, atomically." The intuition is that we usually want to modify a variable based on what it currently holds. If, in between reading it and attempting to change it, another thread has modified it from under our feet, we'd need to update our notion of *what's the current value* before we can change it. This is the usual usecase for the CAS loop.

Here's the simplest example, of incremeting a counter using a CAS loop:

```rust
use std::sync::atomic::{AtomicUsize, Ordering};

fn increment(counter: &AtomicUsize) {
    loop {
        let current = counter.load(Ordering::Acquire);
        let new = current + 1;

        if counter
            .compare_exchange_weak(current, new, Ordering::Release, Ordering::Relaxed)
            .is_ok()
        {
            break;
        }
        // else: someone else changed it first, retry
    }
}
```

What if I told there's an scenario when this promise isn't enough and our intuition can betray us into a false sense of correcteness? When the A we saw first is not the same as current A, even though the value is exactly the same? When A ≠ A?

My first time hearing about this, I was in utter disbelief. I had a hard time imagining, in pure mathematical/abstract terms, why in the world would it matter that the value has changed, if it's ultimately been "restored" to the same value? Surely our previous assumptions still hold and we're free to continue with our operation...

<!--
## The Use-After-Free Race Condition

Maybe you've guessed that the case where our normal intuition starts falling apart is when we working not with values directly but with pointers. How does the old saying go? All problems in computer science are caused by adding a level of indirection? The important thing to understand is that a pointer can point to the same memory but that memory may not be the same.

To build a bit of suspense, let's first look at a similar bug, which helped me to build the intuition for understanding the ABA problem.

Imagine we want to use a lock-free stack. Consider how we'd implement the `pop()` operation. The naive way would look something like this:

```rust
fn pop(&self) -> Option<T> {
    let mut current_head = self.head.load(Ordering::Acquire);

    loop {
        if current_head.is_null() { return None; }

        let new_head = unsafe { (*current_head).next };

        match self.head.compare_exchange_weak(
            current_head,
            new_head, 
            Ordering::AcqRel, 
            Ordering::Acquire) {
            
            Ok(_) => {
                let node = unsafe { Box::from_raw(current_head) };

                // If we drop here, any other thread paused at step 2
                // will crash when it wakes up and tries to read (*current_head).next

                return Some(node.value); // our ptr gets dropped as the Box goes out of scope
            },
            Err(actual_head) => {
                current_head = actual_head;
            }
        }
    }
}
```

This creates a conflict though. Let's imagine the following scenario:

1. Thread 1 starts `pop()`: it reads the shared `head` pointer and sees it points to Node `A` (address `0x1000`).
2. Thread 1 get's context-switched before it can execute its CAS step (just before it can read `Node A.next`.
3. Thread 2 runs `pop()`:
    1. Reads `head` (`0x1000`, Node A).
    2. Reads `Node A.next` (`0x2000`, Node B).
    3. Succesfully executes CAS: `head` nos points to Node B.
4. Thread 2 runs `drop()` for Node A. Since thread 2 "owns" the popped Node A, it immediately calls `drops()`. Node A's destructor runs, and its memory at `0x1000` is returned to the allocator.
5. Thread 1 resumes and attempts to now read `Node A.next`, so it tries to dereference pointer `0x1000`.

{{< mermaid-slider >}}
---
title: "Initial state"
---
flowchart TB
    head(("head"))

    subgraph nodea["node A"]
        direction TB
        na_value["value: 1"]
        na_memory["memory: 0x1000"]
    end

    subgraph nodeb["node B"]
        direction TB
        nb_value["value: 2"]
        nb_memory["memory: 0x2000"]
    end

    head --> nodea
    nodea --> nodeb
%%%
---
title: "Step 1: Thread 1 starts pop() and reads head"
---
flowchart TB
    head(("head"))

    subgraph nodea["node A"]
        direction TB
        na_value["value: 1"]
        na_memory["memory: 0x1000"]
    end

    subgraph nodeb["node B"]
        direction TB
        nb_value["value: 2"]
        nb_memory["memory: 0x2000"]
    end

    t1["Thread 1<br/>head: 0x1000<br/>next: (not read yet)"]

    head --> nodea
    nodea --> nodeb
    t1 -.->|reads| nodea
%%%
---
title: "Step 2: Thread 1 context-switched before reading A.next"
---
flowchart TB
    head(("head"))

    subgraph nodea["node A"]
        direction TB
        na_value["value: 1"]
        na_memory["memory: 0x1000"]
    end

    subgraph nodeb["node B"]
        direction TB
        nb_value["value: 2"]
        nb_memory["memory: 0x2000"]
    end

    t1["Thread 1 (PAUSED)<br/>head: 0x1000<br/>next: ???"]

    head --> nodea
    nodea --> nodeb
    t1 -.-> nodea
%%%
---
title: "Step 3: Thread 2 executes pop() successfully"
---
flowchart TB
    head(("head"))

    subgraph nodeb["node B"]
        direction TB
        nb_value["value: 2"]
        nb_memory["memory: 0x2000"]
    end

    t1["Thread 1 (PAUSED)<br/>head: 0x1000<br/>next: ???"]

    subgraph nodea["node A (popped by T2)"]
        direction TB
        na_value["value: 1"]
        na_memory["memory: 0x1000"]
    end

    head --> nodeb
    nodeb ~~~ t1
    t1 -.-> nodea
%%%
---
title: "Step 4: Thread 2 drops Node A and frees memory at 0x1000"
---
flowchart TB
    head(("head"))

    subgraph nodeb["node B"]
        direction TB
        nb_value["value: 2"]
        nb_memory["memory: 0x2000"]
    end

    t1["Thread 1 (PAUSED)<br/>head: 0x1000<br/>next: ?"]

    subgraph nodea["freed memory (0x1000)"]
        direction TB
        na_status["[deallocated]"]
    end

    head --> nodeb
    nodeb ~~~ t1
    t1 -.->|dangling reference| nodea
%%%
---
title: "Step 5: Thread 1 resumes and attempts use-after-free read"
---
flowchart TB
    head(("head"))

    subgraph nodeb["node B"]
        direction TB
        nb_value["value: 2"]
        nb_memory["memory: 0x2000"]
    end

    t1["Thread 1 resumes<br/>tries to read (0x1000).next<br/>use-after-free"]

    subgraph nodea["freed memory (0x1000)"]
        direction TB
        na_status["[deallocated]"]
    end

    head --> nodeb
    nodeb ~~~ t1
    t1 ==> |fails to dereference!| nodea
{{< /mermaid-slider >}}

**Result:** Memory at `0x1000`  has already been freed or repurposed, which leads to undefined behaviour or segmentation fault (use-after-free).

<figure>
<img src="william-dmytrow-pS6GsfrQZDk-unsplash.jpg" alt="">
<caption>
Photo by <a href="https://unsplash.com/@williamdmytrow?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">William Dmytrow</a> on <a href="https://unsplash.com/photos/winged-victory-of-samothrace-marble-statue-on-a-ship-prow-pS6GsfrQZDk?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
</caption>
</figure>

The fundamental question is:

> How do we defer `drop()` until Thread A is no longer reading it, without adding extra overhead in the form of locks or atomic reference counter updates on every single read?
-->

## What is The ABA Problem

Maybe you’ve guessed that the case where our normal intuition starts falling apart is when we working not with values directly but with pointers. How does the old saying go? All problems in computer science are caused by adding a level of indirection? The important thing to understand is that a pointer can point to the same memory but that memory may not be the same.

<!--Now that we've built an intuition about the strange type of bugs that come up when working with pointers in lock-free code, what is the ABA bug?-->

Let's picture this scenario:

0. Current shape of our stack `[A, B]`.
1. Thread 1 reads `head = A`, reads that `A.next = B`, then gets descheduled *right before* the CAS loop.
2. Thread 2 pops `A` and deallocates the memory.
3. Thread 2 pops `B`.
4. Thread 2 allocates a new node object at the newly freed memory with address `A`, and pushes `A` to the stack. `A.next` is therefore set to null (the stack is now just `[A]`).
5. Thread 1 resumes. Its `compare_exchange(expected = A, new = B)` succeeds, because `head` is in fact `A` again. However, it should've failed because it's not the *same* A. Thread 1 had no way of knowing this though. The CAS sets `head = B`, but `B` was already popped and deallocated in step 3. Now the stack's head is a dangling pointer to freed memory.

The worst thing about this bug is that it's very easy to miss. It needs precise conditions to be met: the interleaving of the two threads as well as the allocator to actually reuse the recently freed address. (stick with me, **promised diagram** coming up).

## Reproducing The ABA Problem

<figure>
<img src="william-dmytrow-pS6GsfrQZDk-unsplash.jpg" alt="">
<caption>
Photo by <a href="https://unsplash.com/@williamdmytrow?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">William Dmytrow</a> on <a href="https://unsplash.com/photos/winged-victory-of-samothrace-marble-statue-on-a-ship-prow-pS6GsfrQZDk?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
</caption>
</figure>
<!--<figure style="text-align: center;">
  <img src="Shipwreck_of_the_Minotaur_William_Turner.jpg" alt="J. M. W. Turner: The Wreck of a Transport Ship" style="display: block; margin: 0 auto;">
  <caption>
    J. M. W. Turner: The Wreck of a Transport Ship
  </caption>
</figure>
-->

I had opened a can of worms. I still didn't understand how come it was the first time in my career hearing about this strange kind of bug. Needless to say, I was intrigued, I felt the urge to try to reproduce it myself, hoping that maybe that'd help build my intuition for detecting this kind of bugs that weren't under my radar before. It turned out to be harder than I expected.

These next few sections are me hitting my head against a wall until I got the basics down. Feel free to skip if you're familiar with these concepts already :)

### Rust's \*mut T

The first roadblock I came across when trying to build my lock-free stack, was understanding raw pointers (`*mut`), since using something like `Arc` would defeat the lock-free effort. This is another rabbit hole, but what's important to understand for us, is that it requires an `unsafe` block for dereferencing. You can pass around raw pointers, that's fine, but when dereferencing it to read or write to the memory behind `*mut T`, the compiler can't guarantee memory safety, hence the need for `unsafe`.

So, our nodes would look like:

```rust
struct Node<T> {
    value: T,
    next: *mut Node<T>,
}
```

We'd dereference it like:

```rust
unsafe { (*node).next = another_node }
```

Also important to remember, dropping a `*mut T` does NOT run `T`'s `Drop` or free its heap memory. For that, we can manually convert it back to an owned type and let it handle the cleanup automatically when it goes out of scope.

??? do we need to assign it?
```rust
let node = unsafe { Box::from_raw(current_head) };
```

Similarly, we use `Box::new` for allocating memory, and then `Box::into_raw` for taking ownership of the underlying `*mut T`.

```rust
let node = Box::new(Node::new(value));
let new_head = Box::into_raw(node);
```

### Why Passing &self is Enough

<figure style="text-align: center;">
  <img src="Echo_and_Narcissus_-_John_William_Waterhouse.jpg" alt="John William Waterhouse: Echo and Narcissus" style="display: block; margin: 0 auto;">
  <caption>
    John William Waterhouse: Echo and Narcissus
  </caption>
</figure>

My first instinct when typing out the `pop` and `push` methods was to use `&mut self`. This made sense at first, since in both cases we'd be modifying the stack. However, I soon realized that if I wanted to pass a reference of the same stack to multiple threads, it'd have to be a shared reference. It sounds obvious in retrospect, I mean, that's the whole point of what we're trying to achieve!

The reason why `&self` is enough is because atomics (`AtomicPtr`) handle thread synchronization internally at the hardware level, essentially bypassing the compiler's borrow rules. They can be mutated behind a shared (`&self`) reference using atomic instructions, like CAS.

### Sharing Across Threads

{{< mermaid-diagram >}}
---
title: "Arc shared across threads"
---
flowchart TB
    subgraph heap["heap"]
        stack["Stack＜T＞"]
    end

    subgraph t1["thread 1"]
        arc1["Arc＜Stack＜T＞＞"]
    end

    subgraph t2["thread 2"]
        arc2["Arc＜Stack＜T＞＞"]
    end

    arc1 -->|points to| stack
    arc2 -->|points to| stack

    t1 -. "pop" .-> stack
    t2 -. "push" .-> stack
{{< /mermaid-diagram >}}

By wrapping our stack in an `Arc`, each thread gets a cloned shared handle pointing to the same stack memory address, allowing concurrent calls to `pop(&self)` and `push(&self, value: T)`.

However, one thing is still missing. It turns out that Rust automatically **disables** `Send` and `Sync` for any type containing raw pointers.

- `Send`: Safe to transfer ownership to another thread.
- `Sync`: Safe to share references (`&Stack<T>`) across multiple threads simultaneously.

So we need to explictily implement those traits (though technically we only need `Sync` for our example):

```rust
unsafe impl<T: Send> Send for Stack<T> {}
unsafe impl<T: Send> Sync for Stack<T> {}
```

We can conclude our final picture of what it looks like it in memory:

{{< mermaid-diagram height="1000px" >}}
---
title: "Memory Layout"
---
flowchart TB
    subgraph threads["What each thread views"]
        subgraph t1["thread 1 call stack"]
            arc1["Arc＜Stack＜T＞＞"]
        end

        subgraph t2["thread 2 call stack"]
            arc2["Arc＜Stack＜T＞＞"]
        end
    end

    subgraph heap["heap"]
        subgraph stack_obj["Stack＜T＞"]
            head["head: AtomicPtr＜Node＜T＞＞"]
        end

        subgraph nodes["linked nodes"]
            n1["Node A (head)<br/>value: T<br/>next: *mut Node＜T＞"]
            n2["Node B<br/>value: T<br/>next: *mut Node＜T＞;"]
            n3["Node C<br/>value: T<br/>next: null"]
        end
    end

    arc1 -->|"&self shared ref"| stack_obj
    arc2 -->|"&self shared ref"| stack_obj

    head -->|"raw ptr (*mut)"| n1

    n1 -->|next| n2
    n2 -->|next| n3

    t1 -. "pop" .-> head
    t2 -. "push" .-> head
{{< /mermaid-diagram >}}

### Buggy Lock-Free Stack

So, for my naive implementation of a lock-free stack, this is what `push` turned out like:

```rust
fn push(&self, value: T) {
    let mut current_head = self.head.load(Ordering::Acquire);
    let node = Box::new(Node::new(value));
    let new_head = Box::into_raw(node);

    loop {
        unsafe { (*new_head).next  = current_head };

        match self.head.compare_exchange_weak(
            current_head,
            new_head, 
            Ordering::AcqRel, 
            Ordering::Acquire
        ) {
            Ok(_) => break,
            Err(actual_head) => {
                current_head = actual_head;
            },
        }
    }
}
```

And `pop`:

```rust
fn pop(&self) -> Option<T> {
    let mut current_head = self.head.load(Ordering::Relaxed);

    loop {
        if current_head.is_null() { return None; }

        // Safety: how do we know no other read is modifying this?
        let new_head = unsafe { (*current_head).next };

        match self.head.compare_exchange_weak(
            current_head,
            new_head, 
            Ordering::AcqRel, 
            Ordering::Relaxed) {
            
            Ok(_) => {
                // Safety: as I'm writing this, rusts forces me to think about the safety of the unsafe operations,
                // and the fact that I can't write a safety statement should be a red flag
                let node = unsafe { Box::from_raw(current_head) };

                return Some(node.value); // our ptr gets dropped as the Box goes out of scope
            },
            Err(actual_head) => {
                current_head = actual_head;
            }
        }
    }
}
```

### Reproducing It (Why No SegFault?)

<figure style="text-align: center;">
  <img src="Boxer_at_rest.jpg" alt="Boxer at Rest" style="display: block; margin: 0 auto;">
</figure>

Now for the moment we've all been waiting for. I've made use of some `thread::sleep`s to trigger the (un)desired order of operations.

<details>
<summary>Expand to view the aba test</summary>

{{< highlight rust >}}
#[test]
fn aba() {
    let stack = Arc::new(Stack::<i32>::new());
    let stack_clone = stack.clone();
    stack.push(2);
    stack.push(1);
    // stack at this point: head -> [1] -> [2] -> nullptr

    thread::scope(|s| {

        // This thread pops, but with a delay between reading head and the CAS loop
        // By the time it enters the CAS loop, the second thread has essentially
        // replaced the head, [1], with [3] that shares the same memory address as [1]
        // original: head -> [1] -> [2]
        // now     : head -> [3] -> nullptr
        s.spawn(|| {
            println!("thread 1 starts pop operation");
            let node = stack.pop_with_delay();

            // If this pop shows up as [3] this means the CAS succeeded,
            // and we were able to reproduce the bug, yay!
            println!("thread 1 pop: {:?}", node);
        });

        // During the first thread's delay window:
        // 1. This thread pops [1]
        // 2. Then pops [2]
        // 3. Pushes a new node [3], with the same address as [1]
        s.spawn(|| {
            // We wait a little to make sure the first thread gets a head (no pun intended) start
            thread::sleep(Duration::from_millis(20));

            println!("thread 2 pops: [{}]", stack_clone.pop().unwrap());
            println!("thread 2 pops: [{}]", stack_clone.pop().unwrap());

            // Let's hope the system heap allocator reuses the memory that was just freed
            // Stack now: head -> [3] -> nullptr
            stack_clone.push(3);
            println!("thread 2 pushes [3]");
        });

        thread::sleep(Duration::from_millis(500));
        // Current stack: head -> [2 (freed)]
        println!("Final pop: [{:?}]", stack.pop());
    });
}
{{< /highlight >}}

</details>

The output, surprinsingly:

```
thread 1 starts pop operation
thread 2 pops: [1]
thread 2 pops: [2]
thread 2 pushes [3]
thread 1 pop: Some(3)
Final pop: [None]
```
Essentially, we've shown that this is what happens:

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

But... 

The output is a bit anticlimatic, isn't it? I was expecting explosions or a segfault at least.

If we didn't know what we were looking for, we might see the output and believe it's completely normal and all is well. What actually happens is undefined behaviour (UB). Why not segfault necessarily? Well, because in this scenario the memory behind the freed object is most likely still mapped, meaning it still belongs to the process, so the OS doesn't complain.

However, if you want to see chaos, there's still hope!

### Miri

What the crossbeam devs use for testing is [Miri](https://github.com/rust-lang/miri/). Quoting Miri's official repo README:

> Miri is an Undefined Behavior detection tool for Rust. It can run binaries and test suites of cargo projects and detect unsafe code that fails to uphold its safety requirements. For instance:
> 
> - Out-of-bounds memory accesses and use-after-free
> - Invalid use of uninitialized data
> - Violation of intrinsic preconditions (an unreachable_unchecked being reached, calling copy_nonoverlapping with overlapping ranges, ...)
> - Not sufficiently aligned memory accesses and references
> - Violation of basic type invariants (a bool that is not 0 or 1, for example, or an invalid enum discriminant)
> - Data races and emulation of some weak memory effects, i.e., atomic reads can return outdated values

Wonderful, exactly what we need!!

Let's try it:

```bash
cargo +nightly miri test aba_problem::tests::aba  
```

It caught our bug! Notice how it explicitly mentions "Undefined Behaviour" due to a data race.

<style>
  .miri-output {
    --bg-color: #181825;
    --border-color: #313244;
    --text-color: #cdd6f4;
    
    --err-color: #f38ba8;
    --err-bg: rgba(243, 139, 168, 0.18);
    --help-color: #89b4fa;
    --location-color: #e5c890;
    --code-color: #a6e3a1;
    --dim-color: #6c7086;
    
    --badge-bg: #313244;
    --t1-color: #89b4fa;
    --t2-color: #f38ba8;
    
    --details-border: #45475a;
    --details-hover: #ffffff;

    background-color: var(--bg-color);
    color: var(--text-color);
    font-family: 'JetBrains Mono', 'Fira Code', Consolas, monospace;
    line-height: 1.5;
    padding: 16px;
    border-radius: 8px;
    border: 1px solid var(--border-color);
    overflow-x: auto;
  }

  @media (prefers-color-scheme: light) {
    .miri-output {
      --bg-color: #f8f9fa;
      --border-color: #dcdfe6;
      --text-color: #24292e;
      
      --err-color: #d73a49;
      --err-bg: rgba(215, 58, 73, 0.12);
      --help-color: #0366d6;
      --location-color: #b05a00;
      --code-color: #1b7c2b;
      --dim-color: #6a737d;
      
      --badge-bg: #e1e4e8;
      --t1-color: #0366d6;
      --t2-color: #d73a49;
      
      --details-border: #d1d5da;
      --details-hover: #000000;
    }
  }

  .miri-err-title { color: var(--err-color); font-weight: bold; }
  .miri-highlight { background-color: var(--err-bg); padding: 1px 4px; border-radius: 3px; }
  .miri-help { color: var(--help-color); }
  .miri-location { color: var(--location-color); font-weight: 500; }
  .miri-code { color: var(--code-color); }

  .miri-noise {
    opacity: 0.45;
    transition: opacity 0.2s ease;
  }
  .miri-noise:hover {
    opacity: 0.9;
  }

  .badge {
    background-color: var(--badge-bg);
    padding: 1px 6px;
    border-radius: 4px;
    font-weight: bold;
    text-transform: uppercase;
  }
  .badge-thread1 { color: var(--t1-color); }
  .badge-thread2 { color: var(--t2-color); }

  .miri-backtrace {
    margin: 8px 0;
    color: var(--dim-color);
  }
  .miri-backtrace summary {
    cursor: pointer;
    color: var(--dim-color);
    user-select: none;
    font-weight: 500;
  }
  .miri-backtrace summary:hover {
    color: var(--details-hover);
  }
  .miri-backtrace-content {
    opacity: 0.55;
    padding-left: 12px;
    border-left: 2px solid var(--details-border);
    margin-top: 4px;
  }
</style>

<pre class="miri-output"><code><span class="miri-noise">test aba_problem::tests::aba ... </span><span class="miri-err-title">error: Undefined Behavior: Data race detected</span> between <span class="badge badge-thread1">(1) thread `unnamed-2`</span> and <span class="badge badge-thread2">(2) thread `unnamed-3`</span> at alloc44257
    --> /home/sofia/.../alloc/src/boxed.rs:1578:9
     |
1578 |         <span class="miri-highlight">Box(unsafe { Unique::new_unchecked(raw) }, alloc)</span>
     |         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ <span class="badge badge-thread2">(2) retag write happened here</span>
     |
<span class="miri-help">help: and (1) occurred earlier here</span>
    --> <span class="miri-location">src/aba_problem.rs:67:37</span>
     |
  67 |             <span class="miri-code">let new_head = unsafe { (*current_head).next };</span>
     |                                     ^^^^^^^^^^^^^^^^^^^^ <span class="badge badge-thread1">(1) non-atomic read</span>
     |
<span class="miri-noise">     = help: retags occur on all (re)borrows and as well as when references are copied or moved</span>
<span class="miri-noise">     = help: retags permit optimizations that insert speculative reads or writes</span>
<span class="miri-noise">     = help: therefore from the perspective of data races, a retag has the same implications as a read or write</span>
     = help: this indicates a bug in the program: it performed an invalid operation, and caused Undefined Behavior
     = note: this is on thread `unnamed-3`

<details class="miri-backtrace">
  <summary>[ Click to expand full backtrace ]</summary>
  <div class="miri-backtrace-content">
     = note: stack backtrace:
             0: std::boxed::Box::&lt;aba_problem::Node&lt;i32&gt;&gt;::from_raw_in
                 at /home/sofia/.../alloc/src/boxed.rs:1578:9
             1: std::boxed::Box::&lt;aba_problem::Node&lt;i32&gt;&gt;::from_raw
                 at /home/sofia/.../alloc/src/boxed.rs:1345:18
             2: aba_problem::Stack::&lt;i32&gt;::pop_internal
                 at src/aba_problem.rs:83:41
             3: aba_problem::Stack::&lt;i32&gt;::pop
                 at src/aba_problem.rs:53:9
             4: aba_problem::tests::aba::{closure#0}::{closure#1}
                 at src/aba_problem.rs:138:49
  </div>
</details>
note: the last function in that backtrace got called indirectly due to this code
    --> <span class="miri-location">src/aba_problem.rs:134:13</span>
     |
 134 | /             <span class="miri-code">s.spawn(|| {</span>
 135 | |                 // We wait a little to make sure the first thread gets a head start
 136 | |                 thread::sleep(Duration::from_millis(20));
...  |
 144 | |                 println!("thread 2 pushes [3]");
 145 | |             <span class="miri-code">});</span>
     | |______________^
</code></pre>

There is still much to learn about Miri, like what exactly is a **"retag"** operation, but for now, we can be happy that that it helped us detect our bug.

## The fix 

The big question now is, okay, what do we do about it? I won't go into detail in this post, but some of the techniques include:

- Tagged pointers: adding a tag to the pointer, that is incremented every time the pointer changes.
- Hazard pointers: threads use hazard pointers to mark the objects they are working on so they don't get dropped.
- Deferred reclamation
    - garbage collection
    - epoch-based reclamation (EBR) --> what **crossbeam-epoch** provides :) stay tuned for the next post exploring this!

## Conclusion

It was a fun experiment trying to reproduce this and seeing first hand how non-trivial it actually is to catch.

My takeaway: If it was so tricky to reproduce knowing from the start what we're looking for, imagine how hard it'd be to detect in production code, if we're not vigilant?
