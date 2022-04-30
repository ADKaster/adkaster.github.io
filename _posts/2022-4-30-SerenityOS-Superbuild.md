---
layout: post
title: SerenityOS's Build System In-Depth
---

The build system for the [Serenity Operating System](https://github.com/SerenityOS/serenity) looks daunting at first glace. Let's take a look under the hood :^)

---

Serenity is a complex software project. As on operating system monorepo, it has patches for the GNU and Clang compiler toolchains to target its Userspace and C library implementations, a bare metal Kernel to build, dozens of Userspace libraries, a runtime dynamic loader, and assorted command line utilities and GUI applications. In terms of scope, serenity could be compared to a *BSD project like FreeBSD, with a desktop environment and "full" set of GUI applications all built out of one monorepo, from scratch, in C++.

As the project has expanded its scope over the last 3 and a half years, it has outgrown its build system a few times. The current build system architecture tries to make the build as portable as possible to as many host operating systems and distributions as possible.

Before we get into the details of the current build system and the problems it tries to solve, however, some history is in order. How has the build system evolved over time?

## In the beginning: Makefiles and shell scripts

Like many Unix-oriented software projects, the original build system for serenity was a maze of Makefiles, and some bespoke scripts to drive them.

When Andreas published his first [monthly update video](https://www.youtube.com/watch?v=hE52D-zbX3g&list=PLMOpZvQB55bfp6ykOLayLqLrjcpv_Sw3P) on March 30, 2019, the build was very old school. As the project was a bit difficult to build [at that time](https://github.com/SerenityOS/serenity/tree/25f28a54a131c4aa188ba2c4c453c1b1648d02c6), let's take a look at how the build looked after a toolchain build helper script was added, all the way back in [pull request number 8](https://github.com/SerenityOS/serenity/tree/3761bc3ed7bc245e5ba4119136e6e33698b30300)! However, the build script still had a few bugs in it after that, so let's move forward [another few months](https://github.com/SerenityOS/serenity/tree/612e1c7023e6b828e9d3d2746cd885e92d4f3f04) until we get to a script that actually works from a fresh checkout (almost). The one modification I had to make at this commit was to change the ``install`` target in LibC/Makefile to depend on ``all`` instead of ``$(LIBRARY)``. 

You can try it yourself by checking out ``612e1c7023e6b828e9d3d2746cd885e92d4f3f04`` from serenity, installing the prerequisites from Meta/BuildInstructions.md, making the one edit to LibC/Makefile (install: all), and running the build. The build steps required are:

```console
# Make the mentioned edit to LibC/Makefile
cd Toolchain
./BuildIt.sh
source UseIt.sh
cd ../Kernel
./makeall.sh && sudo run
```

That sure is a mouthful, isn't it? Let's go over what's actually happening here. There's really 4 steps to the build at this stage:

1. Build the cross-compilation toolchain (BuildIt.sh)
1. Build the Kernel and Userland C++ files (Kernel/makeall.sh)
1. Install all the files into a sysroot, and build a root filesystem for Userland (Kernel/sync.sh, called from makeall.sh)
1. Point QEMU to the Kernel binary and rootfs, and boot the system (Kernel/run)

These basic steps haven't really changed much since May of 2019, but the details certainly have.

Running the build steps from this commit boots into an early, yet still familiar serenity QEMU window.

![VisualBuilder](/images/build/VisualBuilder.png)


### Cross-compiler? And what's a sysroot?

Before we go much further, it's important to understand what a cross-compiler and compiler "sysroot" actually is.

When 

### Makefile build, explained

At this point in time, the basics of the toolchain build were already in place:

1. Download and patch toolchain files. In this case, binutils 2.32 and gcc 8.3.0.
1. Build binutils and gcc targeting ``i686-pc-serenity``
1. Build and install compiler support machinery (libgcc)
1. Ensure LibM and LibC headers are installed into the sysroot for the build
1. Build the C++ standard library (libstdc++) and install it into the sysroot for the Kernel and Userland to link against.

---

