---
title: "Let's make an Operating System: The beginning"
date: 2024-06-04T22:12:15+07:00
toc: true
tags:
  - programming
  - operating system
  - low level system
author: "Nam Nguyen"
description: "Learn about operating systems by making one from scratch"
---

We are going to build an operating system from **scratch**. Why you may ask? Well, I had finished an operating system class a few semester ago but it was a whole lot of theory and no practice on those theory what so ever. After that class, I didn't feel like I truly understand operating system and how they work. So I decided to try to make my own operating system to get more practical experience and a deeper understanding of OS development.

I'll share my journey on this blog. I'll try to explain everything in details to the best of my abilities and understanding so that anyone who wants to take on a similar endeavor can learn faster.

This blog post will just go over how to setup an environment for making this operating system.

## Picking a processor architecture

An operating system has to be coded to be able to operate on a specific processor type, just like any other programs. Modern operating systems are programmed to handle various different architectures. Just take Linux for example, you can view the [code](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch?h=v6.10-rc2) for different architectures and see how many different processor architecture it supports. 

We won't be able to program an OS that can handle many different processor architecture because that is pretty complicated and my attention span has a limit. So we'll pick **one** architecture and stick to building an OS for that specific architecture. 

I opted to use the *x86* architecture as they are very popular and there's heaps of documentations and tutorials on how to program for the *x86* architecture, which will make programming an OS for this architecture a little bit less insanity-inducing. We'll be coding specifically for the *i386* family of processors as they are pretty old, very well documented and not super complicated.

## Tools

Now let's get to installing the tools and setting up the environment. First of all, we are going to need a *Unix* operating system since it already contains all the tools or has an easy way to install those tools. I'll be using a Mac but I think Linux is a lot better for this since it will have all the standard tools already.

We'll be installing 2 tools: [*QEMU*](https://www.qemu.org/) and [*NASM*](https://nasm.us/)

- *QEMU* is an emulator. It allows us to emulate our chosen processor and run programs on it. Since *i386* is not really used nowadays, we will use this emulator to run our operating system. 
- *NASM* is an assembler. It allows us to compile/assemble our assembly program into *i386* binary executables. These binary executables will be uploaded onto *QEMU* to run.

## Setting up the environment

For Mac, [install Homebrew](https://brew.sh/) then run this command:

```sh
brew install qemu nasm
```

For Linux (Debian based), run this command if you don't already have QEMU and NASM:

```sh
sudo apt install qemu nasm
```

You may need to update the PATH variable in your `.bashrc` (or `.zshrc` if you use zsh) with the path to those tools you just installed. Here's an example for NASM:

```sh
# in .zshrc
export PATH="/usr/local/bin/nasm"
```

For Windows, I'm not sure how to set it up as I don't really have a Windows machine, but I think you can install QEMU and NASM on Windows just fine. Just make sure you can use them on your terminal. 

## Conclusion

That should be it for setting up the environment to do this project. If I do end up using more tools, I'll talk about setting up those tools in the future. For now, these tools will be all you need. 

In the [next post](./os-dev-2.md), we'll take a look at boot sectors is an how to make one.
