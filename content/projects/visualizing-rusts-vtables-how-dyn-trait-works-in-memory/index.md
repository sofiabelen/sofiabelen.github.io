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

## Introduction


