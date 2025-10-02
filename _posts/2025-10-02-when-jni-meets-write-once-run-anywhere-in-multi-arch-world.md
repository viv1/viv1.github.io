---
layout: post
title: "When JNI Meets 'Write Once, Run Anywhere': Navigating Java's Multi-Architecture Reality"
date: 2025-10-02 09:30
category: Java
tags: ["java", "jni", "native-code", "architecture", "cross-platform", "performance", "practical", "UnsatisfiedLinkError"]
description: Java's "Write Once, Run Anywhere" works perfectly—until you need native libraries. Here's how modern Java applications handle the multi-architecture reality of JNI.
---

- [Introduction](#introduction)
- [The "Write Once, Run Anywhere" Philosophy](#the-write-once-run-anywhere-philosophy)
   * [The Problem It Solved](#the-problem-it-solved)
   * [Java's Brilliant Solution](#javas-brilliant-solution)
- [The Exception: When Architecture Matters](#the-exception-when-architecture-matters)
   * [Enter JNI: The Bridge to Native Code](#enter-jni-the-bridge-to-native-code)
   * [The Problem: Native Libraries Are Platform and Architecture-Specific](#the-problem-native-libraries-are-platform-and-architecture-specific)
- [How Libraries Solve the Multi-Arch Problem](#how-libraries-solve-the-multi-arch-problem)
   * [The "Bundle Everything" Pattern](#the-bundle-everything-pattern)
   * [The "Platform/Architecture-Specific JAR" Pattern](#the-platformarchitecture-specific-jar-pattern)
   * [When to Use Each Pattern](#when-to-use-each-pattern)
- [Conclusion](#conclusion)
   * [Further Reading](#further-reading)
- [References](#references)

<!-- TOC end -->

## Introduction

One of Java's most celebrated promises is **"Write Once, Run Anywhere" (WORA)**. The idea is simple and powerful: write your code once, compile it, and run it on any platform—Windows, Linux, macOS, x86, ARM—without modification. This promise has made Java the language of choice for enterprise applications, Android development, and countless other use cases.

For pure Java code, this promise holds beautifully. But there's a caveat.

When you venture into the world of **JNI (Java Native Interface)**, which is the bridge between Java and native code written in C, C++, or Rust—the cross-platform simplicity disappears. Suddenly, we need to worry about platform-specific binaries, and get treated with the cryptic `UnsatisfiedLinkError` error messages.

In this article, we'll explore:

- How Java is normally platform-independent
- When and why that independence breaks down with JNI
- How modern libraries cleverly solve the multi-architecture challenge
- Peak under the hood of a production multi-architecture library

## The "Write Once, Run Anywhere" Philosophy

Java's WORA promise isn't just marketing—it's a fundamental architectural decision that revolutionized software development in the mid-1990s.

### The Problem It Solved

Before Java, cross-platform software required painful compromises:

1. **Separate codebases** for each platform (Windows, Mac, Unix)
2. **Abstraction layers** like POSIX, but still compile separately for each platform
3. **Interpreted languages** like Perl or Python, which were slower and had limited ecosystem

### Java's Brilliant Solution

Java introduced an intermediate layer: **bytecode**. Instead of compiling directly to machine code, Java compiles to a universal, platform-neutral instruction set that can run on any machine with a JVM.

```
Source Code (.java)  →  Bytecode (.class)  →  JVM (x86/ARM/etc)  →  Machine Code
   [Developer]           [Compiler]            [Runtime]              [CPU]
```

When you compile Java code with `javac`, it creates `.class` files containing bytecode—universal instructions that aren't specific to x86, ARM, or any particular CPU architecture. These bytecode instructions (like `aload_0`, `getstatic`, `invokevirtual`) are understood by the JVM, not directly by the hardware.

The magic happens at runtime: each platform has its own JVM implementation (compiled for x86-linux, aarch64-darwin, etc.) that reads this universal bytecode and translates it to native machine code via Just-In-Time (JIT) compilation. The same `.jar` (think of it as a zipped folder of .class files) file can run on Linux x86_64, macOS ARM64, or Windows x86_64 without any modification.

This architecture provides  **True portability**, since the same `.jar` file runs everywhere.


## The Exception: When Architecture Matters

Now comes the interesting part: **when does this beautiful abstraction break down?**

### Enter JNI: The Bridge to Native Code

JNI (Java Native Interface) allows Java code to call functions written in C, C++, Rust, or other languages that compile to native machine code.

**Why would we want to do this?**

1. **Performance**: Critical operations like cryptography are often faster in native code
2. **Hardware Access**: Low-level hardware operations that Java can't do
3. **Legacy Integration**: Calling existing C/C++ libraries
4. **System APIs**: Accessing OS-specific features not exposed in Java

> I feel it's important to mention that using JNI is not always beneficial. JNI calls incur an overhead. So, if JNI is invoked frequently, for low-work operations, it's not going to help reduce latency. We should only use it if the Java native operation is much slower. 
{:.prompt-info}


### The Problem: Native Libraries Are Platform and Architecture-Specific

When you compile native code, you must compile it for each target platform (Windows, MacOS, Linux, etc) and target architecture (x86-64, ARM64, armv7, etc). 

We can run this simple test to explore this ourselves:

```bash
% mvn dependency:get -Dartifact=org.xerial:sqlite-jdbc:3.44.1.0 # Download sqlite jdbc jar from maven
% jar xf ~/.m2/repository/org/xerial/sqlite-jdbc/3.44.1.0/sqlite-jdbc-3.44.1.0.jar # Extract its contents

% # Checkout its target platform & architecture
% file native/Mac/x86_64/libsqlitejdbc.dylib
native/Mac/x86_64/libsqlitejdbc.dylib: Mach-O 64-bit dynamically linked shared library x86_64

% file native/Mac/aarch64/libsqlitejdbc.dylib
native/Mac/aarch64/libsqlitejdbc.dylib: Mach-O 64-bit dynamically linked shared library arm64

% file native/Windows/x86_64/sqlitejdbc.dll
native/Windows/x86_64/sqlitejdbc.dll: PE32+ executable (DLL) (console) x86-64 (stripped to external PDB), for MS Windows

% file native/Windows/x86/sqlitejdbc.dll
native/Windows/x86/sqlitejdbc.dll: PE32 executable (DLL) (console) Intel 80386 (stripped to external PDB), for MS Windows

% file native/Windows/armv7/sqlitejdbc.dll
native/Windows/armv7/sqlitejdbc.dll: PE32 executable (DLL) (GUI) ARMv7 Thumb, for MS Windows

% file native/Windows/aarch64/sqlitejdbc.dll
native/Windows/aarch64/sqlitejdbc.dll: PE32+ executable (DLL) (GUI) Aarch64, for MS Windows
```

The x86 `.so` file won't work on ARM, and vice versa. The Linux `.so` won't work on Windows. When you try it, you will get the famous `java.lang.UnsatisfiedLinkError`. 

The "Write Once, Run Anywhere" promise is broken, in a way.


## How Libraries Solve the Multi-Arch Problem

Modern Java libraries that use JNI have developed clever patterns to maintain cross-platform compatibility.

### The "Bundle Everything" Pattern

The most common solution: **include all native libraries for all platforms in a single JAR**. We saw this in the last section with `sqlite-jdbc`. 

At runtime, the Java code detects the platform and extracts the appropriate native library:

1. **Detect OS and architecture** from system properties
2. **Extract the correct native library** from the JAR to a temporary location
3. **Load the library** using `System.load()`
4. **Clean up** temporary files on JVM exit

The JAR file is larger (containing multiple native libraries), but the developer experience is seamless.

### The "Platform/Architecture-Specific JAR" Pattern

An alternative approach: **publish separate JARs for each platform**, and let the build tool select the right one at build time.

Instead of one fat JAR containing all native libraries, libraries publish multiple artifacts:

For instance, checkout `netty-tcnative`:

```bash
% curl -s https://repo1.maven.org/maven2/io/netty/netty-tcnative/2.0.74.Final/ | grep -o 'netty-tcnative-[^"]*\.jar' | sort -u
netty-tcnative-2.0.74.Final-javadoc.jar
netty-tcnative-2.0.74.Final-linux-x86_64-fedora.jar
netty-tcnative-2.0.74.Final-linux-x86_64.jar
netty-tcnative-2.0.74.Final-osx-aarch_64.jar
netty-tcnative-2.0.74.Final-osx-x86_64.jar
netty-tcnative-2.0.74.Final-sources.jar
netty-tcnative-2.0.74.Final.jar
```

**One slight hiccup:**

If we specify the platform manually in our `pom.xml`, it becomes platform-specific:

```xml
<!-- BAD: Hard-coded platform -->
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-tcnative</artifactId>
    <version>2.0.74.Final</version>
    <classifier>linux-x86_64</classifier>  <!-- Won't work on macOS! -->
</dependency>
```

Our `pom.xml` now only works on Linux x86_64. Developers on macOS can't build, and we can't deploy to ARM servers.

**The Solution: Maven's os-maven-plugin**

The [os-maven-plugin](https://github.com/trustin/os-maven-plugin) automatically detects your build platform and selects the correct JAR:

```xml
<!-- Add the plugin -->
<build>
    <extensions>
        <extension>
            <groupId>kr.motd.maven</groupId>
            <artifactId>os-maven-plugin</artifactId>
            <version>1.7.1</version>
        </extension>
    </extensions>
</build>

<!-- Use auto-detected classifier -->
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-tcnative</artifactId>
    <version>2.0.74.Final</version>
    <classifier>${os.detected.classifier}</classifier>  <!-- Auto-detected! -->
</dependency>
```

The plugin sets `${os.detected.classifier}` based on your build environment. This way, the same `pom.xml` works on every developer's machine and in every CI/CD environment!

### When to Use Each Pattern

**Use "Bundle Everything" when:**
- Distributing desktop applications to unknown end-users
- We want a single universal artifact that works everywhere
- We deploy the same JAR to multiple different architectures
- Simplicity is more important than JAR size

**Use "Platform-Specific JARs" when:**
- Building server applications deployed to known platforms
- Using containerization (each container image is already platform-specific)
- JAR size matters (e.g., AWS Lambda cold start times)
- We have a consistent CI/CD pipeline per architecture

## Conclusion

Java's "Write Once, Run Anywhere" promise isn't a myth, it just comes with a practical asterisk when we venture into the native code territory. But with the patterns we've explored, modern Java libraries have found elegant ways to maintain cross-platform compatibility even when using architecture-specific native code.

### Further Reading

The Java ecosystem continues to evolve:

- **[Project Panama](https://openjdk.org/projects/panama/)** (Foreign Function & Memory API): Aims to replace JNI with a safer, simpler API
- **[GraalVM Native Image](https://www.graalvm.org/latest/reference-manual/native-image/)**: Compiles Java to native binaries, changing the paradigm entirely
- **Improved JIT compilation**: Narrowing the performance gap between Java and native code

Despite these advances, understanding JNI and multi-architecture challenges remains crucial for Java developers working with high-performance applications, legacy integrations, or specialized hardware.

If you liked this article, you might like my other article on [Simplifying the complicated world of Libraries ](https://wewake.dev/posts/simplifying-libraries-part-1/)

## References

- [Java Native Interface Specification](https://docs.oracle.com/en/java/javase/17/docs/specs/jni/index.html){:target="_blank"}
- [Netty TCNative GitHub Repository](https://github.com/netty/netty-tcnative){:target="_blank"}
- [os-maven-plugin GitHub Repository](https://github.com/trustin/os-maven-plugin){:target="_blank"}
- [Maven Classifier Documentation](https://maven.apache.org/pom.html#dependencies){:target="_blank"}
- [Project Panama: Foreign Function & Memory API](https://openjdk.org/projects/panama/){:target="_blank"}
- [SQLite JDBC Driver](https://github.com/xerial/sqlite-jdbc){:target="_blank"}
- [JNI Best Practices](https://developer.android.com/training/articles/perf-jni){:target="_blank"}
- [Understanding Java Bytecode](https://docs.oracle.com/javase/specs/jvms/se17/html/index.html){:target="_blank"}
- [Simplifying the complicated world of Libraries ](https://wewake.dev/posts/simplifying-libraries-part-1/)