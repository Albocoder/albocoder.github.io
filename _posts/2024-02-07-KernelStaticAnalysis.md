---
layout: post
title: "Static analysis of Linux kernel"
categories:
  - exploitation
  - linux kernel
---

[![Analyst detective AI cover photo](/assets/blog5_static_analysis_kernel/cover.jpeg)](/assets/blog5_static_analysis_kernel/cover.jpeg)

# Introduction
I've seen a lot of papers in the academia talk about static analysis in linux kernel's source code. Some use old tools like *KINT*, some others just mention an *LLVM* use-def pass, but for a newcomer it's difficult to write all of those on their own. In this blog post, I will go step by step into **how to track a variable allocation, its data type and it respective kmalloc cache**. I will be running everything in `Ubuntu 20.04` but it can work in any OS if you try hard enough.

For this mini-project I will be using llvm/clang 13 with linux kernel `5.13.14` (same kernel as in my other <a href="/exploit/2023/03/13/KernelFileExploit.html" target="_blank">blog post</a> where I exploit `CVE-2017-27666`). 

# Setup

