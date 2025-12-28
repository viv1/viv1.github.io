---
layout: post
title: Simplifying the complicated world of Libraries - Part 1
date: 2024-04-14 09:22
category: Operating Systems
tags: ["library", "static library", "dynamic library", "shared library", "static linking", "dynamic linking", "system library"]
description: We talk about library binaries, and its types, advantages, disadvantages, and focus on compatibility across platforms with respect to shared libraries.
pin: true
---

<!-- TOC start (generated with https://github.com/derlin/bitdowntoc) -->

- [Introduction](#introduction)
- [Layman version overview](#layman-version-overview)
   * [How They Work ?](#how-they-work)
   * [Where Things Get Tricky with Borrowing Tool kit](#where-things-get-tricky-with-borrowing-tool-kit)
   * [The Beauty of Borrowing the Tool kit](#the-beauty-of-borrowing-the-tool-kit)
- [Types of Libraries: Static vs Dynamic](#types-of-libraries-static-vs-dynamic)
   * [Static Library](#static-library)
   * [Dynamic Library](#dynamic-library)
   * [Differences Table](#differences-table)
- [Compatibility across different platforms](#compatibility-across-different-platforms)
   * [Can a library compiled for an `x86` Linux OS system run on an `ARM` macOS system ?](#can-a-library-compiled-for-an-x86-linux-os-system-run-on-an-arm-macos-system-)
   * [Can a library compiled for an `x86` Linux OS system run on an `x86` macOS system ?](#can-a-library-compiled-for-an-x86-linux-os-system-run-on-an-x86-macos-system-)
   * [Strategies for Limited Compatibility](#strategies-for-limited-compatibility)
- [Conclusion](#conclusion)

<!-- TOC end -->

> I have split this into 2 articles, to keep it in consumeable sizes. `Part 1` introduces libraries, and talks about its types, advantages, disadvantages, limitations and compatibility across platforms. In `Part 2` (WIP), we dive a little bit more while looking at a couple of real world examples of the use of `dynamic/shared` libraries.
{:.prompt-info}

<!-- TOC --><a href="#" name="introduction"></a>
## Introduction

Remember that time back in college, scratching your head, trying to figure out why a game or a side project that built & ran flawlessly on a friend's system wouldn’t work on yours? Or in a professional setting, puzzling over why an application you packaged on an Mac M1 suddenly refused to play nice when you deployed it to a x86 linux box ?

More often than not, the culprit behind these compatibility issues is the way libraries are handled. Libraries are essentially collections of pre-written code that provide functionality to programs, saving developers from reinventing the wheel every time they need to perform common tasks like file compression, networking, or graphics rendering.

More recently, I encountered such compatibility issues while I was working on making one of our services architecture-agnostic (i.e. ensuring it runs seamlessly across multiple architectures)

Navigating the complexities of system libraries across different operating systems (`Windows vs. Linux`) and CPU architectures (`AMD vs. ARM`) can be a complicated path. In this article, I try to share my understanding and learning of the concept, in the hopes that the next time you encounter any such issues, you are more aware of the internals.


<!-- TOC --><a href="#" name="layman-version-overview-of-library"></a>
## Layman version overview

<!-- TOC --><a href="#" name="how-they-work"></a>
### How They Work ?

***`The Toolshed`***: Your operating system (`Windows, Linux, macOS`) has a whole "toolshed" full of these shared libraries. They handle everything from graphics, sound, cryptography and networking to super-specific calculations or file formats.

***`Included Tool kit:`*** When you install a program, sometimes the installer brings all the "tools" it needs.  That's like `static linking` – the program has its own toolset embedded. But often, to save space and keep things up-to-date, it just makes a note like, "Hey, I need the SuperFancy Graphics Toolkit from the toolshed."

***`Borrowing Tool kit`***: You fire up that game or program. Before it launches fully, a little helper (the `dynamic linker`)  checks out the program's tool list and says, "Okay, where's that SuperFancy Graphics Toolkit?" It then finds the right version in your OS toolshed and makes sure your program can use it.


Included Tool kit is analogous to `Static Library`.
Borrowing Tool kit is analogous to `Dynamic/Shared Library`.

<!-- TOC --><a href="#" name="where-things-get-tricky-with-borrowing-tool-kit"></a>
### Where Things Get Tricky with Borrowing Tool kit

***`Missing Tools`***: Your friend's computer, running a different operating system or maybe an older version, might not have that SuperFancy Graphics Toolkit, or it has an outdated version.

***`Wrong Toolshed`***: You try running that Linux project on Windows, but the program wants libraries that are only in the Linux toolshed.

***`Different Toolshed`***: You try running that Linux project on Windows, using implementation of a similar dependency library for Windows, but the implementation differences lead to slightly different output.

***`DIY Headaches`***: In professional settings, this gets even weirder. Your fancy app works on your version of Linux, but the company server has a slightly different setup – a different toolkit version can lead to unexpected glitches.

<!-- TOC --><a href="#" name="the-beauty-of-borrowing-the-tool-kit"></a>
### The Beauty of Borrowing the Tool kit

Despite the compatibility headaches, shared libraries are amazing:

***`Updates`***: Security flaw in that SuperFancy Graphics Toolkit? Update the one in the toolshed, and poof, all the games and programs using it are patched.

***`Space advantage`***: Imagine if every game had its own copy of the graphics code. Your hard drive would scream! Shared libraries let everyone use that same, vetted toolkit.

***`Ease of Updates`***: Updating a shared library can provide new features or patches to all applications that use it, without needing to recompile them.

***`Interoperability`***: Shared libraries can help in standardizing code across applications, leading to more reliable and consistent behavior across different programs.


<!-- TOC --><a href="#" name="types-of-libraries-static-vs-dynamic"></a>
## Types of Libraries: Static vs Dynamic

In short, if the linking is done at `compile time` (i.e. `static linking`), it generates a `static library`.
And, if the linking is done at `run time`, it generates a `dynamic/shared library`.

<!-- TOC --><a href="#" name="static-library"></a>
### Static Library

- ***Packaging***: The linker combines all the necessary (internal & external) libraries and modules into a single executable file during the compilation process. This is done before the executable is run on any system.

- ***Less Flexibility***: If any of the linked libraries need updating (such as `for security patches or new features`), the entire application must be `recompiled` and `redistributed`.

- ***Isolation***: The application does not depend on external libraries being present on the system where it runs, which avoids `"dependency hell."`

On a linux distribution, you could run something like this to create a static library for a C program `myapp.c`:

```bash
// Compile with static linking
gcc -o myapp myapp.c -static-libgcc -static-libstdc++
```

<!-- TOC --><a href="#" name="dynamic-library"></a>
### Dynamic Library

- ***Packaging***: Instead of embedding library code into the executable, the compiler places `"stubs"` or placeholders. These stubs hold information about the library functions your code needs to use.
When your program runs, a special program called the `"dynamic linker"` kicks in. It searches for the needed dynamic libraries (e.g., `.dll` or `.so` files), loads them into memory, and fixes the `"stubs"` in your executable to point to the actual function code in the dynamic library.

- ***Ease of Updates***: Libraries can be updated independently of the application, which is particularly important for applying `security patches`.

- ***Resource Sharing***: Multiple applications can share the same library in memory, reducing the overall system memory footprint.

On a linux distribution, you could run something like this to generate a dynamic library for a C program `myapp,c`:

```bash
// Compile with dynamic linking
gcc -o myapp myapp.c -lmylib
```

<!-- TOC --><a href="#" name="differences-table"></a>
### Differences Table

| Characteristic | Static Library | Dynamic Library |
| :-| : | : |
| **When linking occurs** | Compile-Time Linking | When program runs |
| **Executable size** | Larger | Smaller |
| **Self-contained** | Yes | No (relies on external libraries) |
| **Update flexibility** | Requires recompilation, so difficult to update | Can use updated libraries without recompilation, so easier to update, but needs careful management of library versions to prevent compatibility issues. |
| **Memory usage** | Less efficient (if multiple programs use the same library) | More efficient (shared library can be used by multiple programs) |
| **Performance** | Better runtime performance | Slower, but does benefit from system-wide caching mechanisms |
| **Deployment** | Simple, since no external dependency | Needs installation of dependent libraries on the system |
| **Compatibility** | No external dependency, so "no dependency hole" | Requires careful management of library versions to prevent compatibility issues. |
| **Advantages** | Faster startup, guaranteed compatibility | Runtime	Smaller executables, easier updates, memory savings |
| **Disadvantages** | Larger executables, harder to update libraries | Slower startup, potential for version conflicts |
| **File Extension** | `.a` (Linux/MacoS), `.lib` (Windows) | `.so` (Linux), `.dylib` (MacOS), `.dll` (Windows) |

The choice between `compile-time` and `run-time` `linking` often depends on the specific needs of the application and the environment in which it operates. `Static linking` is preferred when you need simplicity and reliability in isolated environments, while `dynamic linking` is ideal for complex, resource-optimized environments where applications share many common libraries.

> Did you know ? MacOS provides another way of `dynamic linking structure` called `Frameworks`. While `.so` is a single executable file, `Framework` is a directory containing dynamic library + metadata + docs + assets and related files. [Read more about it here](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPFrameworks/Concepts/WhatAreFrameworks.html){:target="_blank"}.
{:.prompt-info}


> Did you also know ? "Shared Library linking" can be done either at [`load-time` or at `run-time`](https://stackoverflow.com/questions/2055840/difference-between-load-time-dynamic-linking-and-run-time-dynamic-linking){:target="_blank} ? `Load-Time Dynamic Linking` is when the OS loads the library into memory when the executable that depends on it is launched. `Run-Time Dynamic Linking` is when a program can load the library into memory during its execution, typically using a specific API provided by the OS.
{:.prompt-info}


<!-- TOC --><a href="#" name="compatibility-across-different-platforms"></a>
## Compatibility across different platforms

<!-- TOC --><a href="#" name="can-a-library-compiled-for-an-x86-linux-os-system-run-on-an-arm-macos-system-"></a>
### Can a library compiled for an `x86` Linux OS system run on an `ARM` macOS system ?

**NOPE**.
Libraries are compiled to machine code. The machine code instructions are completely different for an `x86/AMD` architecture vs an `ARM` architecture.
- `x86/AMD` architecture uses `CISC` or Complex Instruction Set Computers.
- `ARM` architecture uses `RISC` or Reduced Instruction Set Computers.

Outside the scope of this article. [This standford course page is a good overview if you want to read further on it](https://cs.stanford.edu/people/eroberts/courses/soco/projects/risc/risccisc/){:target="_blank"}.

<!-- TOC --><a href="#" name="can-a-library-compiled-for-an-x86-linux-os-system-run-on-an-x86-macos-system-"></a>
### Can a library compiled for an `x86` Linux OS system run on an `x86` macOS system ?

**NOPE:**
The `machine code` is the same for the same `CPU architecture`, and it doesn't depend on the `OS`.

However, the `machine code` in a program written for one `OS` won't necessarily run on a different `OS`, even if the processor is exactly the same. This is because [***`ABIs`***](https://stackoverflow.com/questions/2171177/what-is-an-application-binary-interface-abi){:target="_blank"} differ between `operating systems`.

Some aspects of the executable program are highly dependent on the `OS`:
- The format of the executable file varies from one `OS` to another.
- Systems may also differ in other details, such as memory arrangement, operating systems, or peripheral devices.

Because a program normally relies on such factors, different systems will typically not run the same machine code, even when the same type of processor is used. A Windows `.dll` won't function on Linux.

This is why you'll see separate binaries, depending on OS & architecture, for example in [netty-tcnative](https://repo1.maven.org/maven2/io/netty/netty-tcnative/2.0.65.Final/){:target="_blank"}.

<!-- TOC --><a href="#" name="strategies-for-limited-compatibility"></a>
### Strategies for Limited Compatibility

The libraries must be compiled specifically for each architecture to ensure proper operation. Developers often maintain separate builds of their libraries for different architectures to support a broad range of devices, from servers and desktops to mobile devices.

- `Recompilation`: The most reliable approach: obtain the source code of the library and recompile it specifically for the target `OS` and `architecture`.
- `Emulation and Translation Layers`: In some cases, tools exist to emulate one `architecture` on another (e.g., running `x86` software on `ARM` through emulation). This can introduce performance overheads.
- `Cross-Platform Frameworks`: Frameworks like `Qt` or `Electron` abstract away `OS/architecture` differences, but add their own dependencies.

<!-- TOC --><a href="#" name="conclusion"></a>
## Conclusion

Libraries play a pivotal role in modern software development. Understanding their types, trade-offs, and the challenges of cross-platform compatibility empowers us to make informed decisions about application architecture and dependency management. Mastering these details might seem daunting, but remember that even seasoned professionals sometimes find themselves scratching their heads over a missing library or an unexpected version conflict. While not an expert, but I have myself been frustrated with these issues numerous times.

Stay tuned for `Part 2`, where we'll delve into some real-world examples of `shared libraries`.
