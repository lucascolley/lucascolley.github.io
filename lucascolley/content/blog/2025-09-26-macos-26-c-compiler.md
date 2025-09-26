---
title: "Upgrading macOS broke my C compiler... so I* fixed it!"
tags: ["conda-forge"]
---

*with the indispensable help and guidance of the [conda-forge] core team!

[conda-forge]: https://conda-forge.org

### Patient zero

Yesterday, disaster struck — after upgrading my main machine to macOS 26 (thinking *what could possibly go wrong?*), I was suddenly unable to build [SciPy].
Reviewing pull requests certainly becomes a bit more difficult when you are faced with this error whenever you try to test changes on your machine:

[SciPy]: https://scipy.org

```console
❯ pixi run build
✨ Pixi task (build in build): spin build --setup-args=-Dblas=blas --setup-args=-Dlapack=lapack --setup-args=-Duse-g77-abi=true: (Build SciPy (default settings))
$ meson setup build --prefix=/usr -Dblas=blas -Dlapack=lapack -Duse-g77-abi=true
The Meson build system
Version: 1.9.0
Source dir: /path/to/scipy/scipy
Build dir: /path/to/scipy/scipy/build
Build type: native build
Project name: scipy
Project version: 1.17.0.dev0+git20250925.e4d0fa1

meson.build:1:0: ERROR: Executables created by c compiler ccache arm64-apple-darwin20.0.0-clang are not runnable.
```

At a glance, the error seemed to be telling me that my C compiler (`clang`, from conda-forge) was creating executables which cannot run on my machine.
Since this was the first time I had tried compiling C code since upgrading to macOS 26, I figured that the upgrade was probably the cause of the problem.
Now the tricky part: figure out how to fix it!

At this stage, it wasn't clear whether it was just my machine that needed some additional setup following the upgrade, or whether there was some incompatibility between the compiler distributed on conda-forge and the newly released OS version.

I took a look at the logs from [`meson`] (SciPy's build system) to see if that would help clear things up.
Unfortunately, I didn't understand much of the extended error message I found there:

[`meson`]: https://mesonbuild.com

```txt
Sanity check: `/path/to/scipy/scipy/build/meson-private/sanitycheckc.exe` -> -6
stderr:
dyld[87666]: missing LC_LOAD_DYLIB (must link with at least libSystem.dylib) in /path/to/scipy/scipy/build/meson-private/sanitycheckc.exe
dyld[87666]: missing LC_LOAD_DYLIB (must link with at least libSystem.dylib)
-----------

meson.build:1:0: ERROR: Executables created by c compiler arm64-apple-darwin20.0.0-clang are not runnable.
```

It was time to reach out to [the conda-forge community on Zulip] for help.
From a brief look through recent threads, it seemed that I was indeed 'patient zero' of the problem.
Quickly enough, I was no longer alone, with [@pavelzw] a confirmed 'patient one'.
Thanks to [Pixi], reproducing the problem was as simple as cloning the repository I was working in, and executing a build task with `pixi run build`.

[the conda-forge community on Zulip]: https://conda-forge.zulipchat.com
[@pavelzw]: https://github.com/pavelzw
[Pixi]: https://pixi.sh

### Fixing the compiler

With the problem reproduced and no further clues on how to fix it, I reached out to some members of the conda-forge core team (and experts on the conda-forge compiler toolchains), [@isuruf] and [@h-vetinari].
After just a few hours of intermittent debugging — with them suggesting commands to execute and me reporting the results — [@isuruf] was able to suggest the one-line patch we needed to make the compiler work again:

[@isuruf]: https://github.com/isuruf
[@h-vetinari]: https://github.com/h-vetinari

```patch
From 0bfca2d07168b6959fe97d115b8c48db1ce9132c Mon Sep 17 00:00:00 2001
From: Lucas Colley <lucas.colley8@gmail.com>
Date: Thu, 25 Sep 2025 21:24:29 +0100
Subject: [PATCH 4/4] avoid stripping out C stdlib too zealously

this avoids failures on macOS 26 like `must link with at least
libSystem.dylib`
---
 cctools/ld64/src/ld/passes/dylibs.cpp | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)

diff --git a/cctools/ld64/src/ld/passes/dylibs.cpp b/cctools/ld64/src/ld/passes/dylibs.cpp
index 8f8ee46..d129024 100644
--- a/cctools/ld64/src/ld/passes/dylibs.cpp
+++ b/cctools/ld64/src/ld/passes/dylibs.cpp
@@ -306,7 +306,7 @@ void doPass(const Options& opts, ld::Internal& state)
 		if ( aDylib->explicitlyLinked() && aDylib->deadStrippable() && !aDylib->providedExportAtom() && !aDylib->neededDylib() )
 			aDylib->setWillBeRemoved(true);
 		// set "willRemoved" bit on any unused explicit when -dead_strip_dylibs is used
-		if ( opts.deadStripDylibs() && !aDylib->providedExportAtom() && !aDylib->neededDylib() )
+		if ( opts.deadStripDylibs() && !aDylib->providedExportAtom() && !aDylib->neededDylib() && strncmp(aDylib->installPath(), "/usr/lib/libSystem.", 19) != 0 )
 			aDylib->setWillBeRemoved(true);
 		// <rdar://problem/48642080> Warn when dylib links itself
 		if ( (opts.outputKind() == Options::kDynamicLibrary) && !aDylib->willRemoved() ) {
```

I explore what the problem was and why this patch fixed it below, in [What was the problem?](#what-was-the-problem).
But first, how did we go from this suggested patch, to deploying that patch and updating the compiler on my machine?

The patch was to [https://github.com/tpoechtrager/cctools-port](), a port of Apple [`cctools`] for other operating systems, which includes the 64-bit linker for macOS, [`ld64`].
Importantly, conda-forge obtains the `cctools` and `ld64` dependencies of `clang` from this port when installing `clang` on an Apple Silicon machine.

[`cctools`]: https://github.com/apple-oss-distributions/cctools
[`ld64`]: https://github.com/apple-oss-distributions/ld64

In conda-forge, the pipelines which transform source code into [conda packages] are contained in [feedstocks], like [https://github.com/conda-forge/cctools-and-ld64-feedstock]().
In this case, the feedstock takes the `tpoechtrager/cctools-port` source code as the input (`source`), and outputs two main conda packages, `ld64` and `cctools`.

[conda packages]: https://prefix.dev/blog/what-is-a-conda-package
[feedstocks]: https://conda-forge.org/docs/maintainer/understanding_conda_forge/feedstocks/

Feedstocks have a systematic approach to patching which makes it easy to include `git` patches on top of the input source code.
Patch files like the one shown above are stored in the feedstock, alongside the feedstock "recipe" and any build scripts, and are automatically applied to the source code before any building steps defined in the recipe take place.

After generating the patch locally with [`git format-patch`], I submitted [a pull request to the feedstock](https://github.com/conda-forge/cctools-and-ld64-feedstock/pull/90).
This pull request both added the patch file to the recipe, and bumped the `build` number, to signify that the new versions of the produced packages, despite corresponding to the same version of the source code, will be 'updated' builds.

[`git format-patch`]: https://git-scm.com/docs/git-format-patch

With the pull request submitted, [conda-forge's CI build farm] kicked into action, with builds running across all target architectures on Azure Pipelines.
The last step was to check that the patch really fixed the problem, by testing out the updated `ld64` package locally on my macOS 26 machine.
Thanks to the automated build farm, I didn't need to build `ld64` locally: instead, I could download the conda package artifact which was produced by the CI build matching my architecture, and use that conda package locally.

[conda-forge's CI build farm]: https://dev.azure.com/conda-forge/feedstock-builds/_build

With Pixi, I was able to confirm that the conda package I had downloaded from CI worked with the task of building [NumPy], via the following diff to [`rgommers/pixi-dev-scipystack`]:

[NumPy]: https://numpy.org
[`rgommers/pixi-dev-scipystack`]: https://github.com/rgommers/pixi-dev-scipystack

```diff
diff --git a/numpy/pixi.toml b/numpy/pixi.toml
index e298297..a385277 100644
--- a/numpy/pixi.toml
+++ b/numpy/pixi.toml
@@ -5,4 +5,4 @@ description = "Fundamental package for array computing in Python"
 authors = ["Ralf Gommers <ralf.gommers@gmail.com>"]
-channels = ["https://prefix.dev/conda-forge"]
-platforms = ["osx-arm64", "linux-64", "win-64"]
+channels = ["file:///Users/lucascolley/Downloads/conda_artifacts_20250925.4.1_osx_arm64_cross_platformosx-arm64llvm_version19.1-cctools-and-ld64-feedstock_conda_artifacts_20250925", "https://prefix.dev/conda-forge"]
+platforms = ["osx-arm64"]

@@ -17,2 +17,4 @@ wheel = { cmd = "python -m build -wnx -Cbuild-dir=build-whl && mv dist/*.whl ../
 [dependencies]
+ld64 = { path = "/Users/lucascolley/Downloads/conda_artifacts_20250925.4.1_osx_arm64_cross_platformosx-arm64llvm_version19.1-cctools-and-ld64-feedstock_conda_artifacts_20250925/osx-arm64/ld64-955.13-he86490a_2.conda" }
+ld64_osx-arm64 = { path = "/Users/lucascolley/Downloads/conda_artifacts_20250925.4.1_osx_arm64_cross_platformosx-arm64llvm_version19.1-cctools-and-ld64-feedstock_conda_artifacts_20250925/osx-arm64/ld64_osx-arm64-955.13-h6922315_2.conda" }
 compilers = ">=1.9.0,<2"
```

All this diff does is specify that:
- the artifact I downloaded is a source from which I want to be able to pull conda packages
- I want to pull `ld64` and `ld64_osx-arm64` from that downloaded artifact, rather than pulling the existing versions from conda-forge.

With the fix confirmed, we were able to merge the pull request to the feedstock, and that was the end of the day for me!
By the time I woke up the next day, the new versions including the patch were distributed on conda-forge, and automatically pulled by Pixi when I asked it for the latest version of `clang` (implicitly, via transitive dependencies in my [workspaces]).

[workspaces]: https://pixi.sh/dev/first_workspace/

### What was the problem? {#what-was-the-problem}

While I don't have the expertise for a full deep-dive into the problem, here is the gist of what I understand went down:
1. conda-forge's `clang` sets the `-Wl,-dead_strip_dylibs` flag to strip unused dylibs by default
2. in macOS 26, Apple introduced a sanity check that C executables/dylibs at least link to the C standard library, when the SDK version with which the executable/dylib was compiled is `>=26`
3. this breaks executables like those used in the `meson` sanity check, which don’t link to the C standard library

Here is a reminder of the error message I saw in the `meson` log:

```txt
Sanity check: `/path/to/scipy/scipy/build/meson-private/sanitycheckc.exe` -> -6
stderr:
dyld[87666]: missing LC_LOAD_DYLIB (must link with at least libSystem.dylib) in /path/to/scipy/scipy/build/meson-private/sanitycheckc.exe
dyld[87666]: missing LC_LOAD_DYLIB (must link with at least libSystem.dylib)
-----------

meson.build:1:0: ERROR: Executables created by c compiler arm64-apple-darwin20.0.0-clang are not runnable.
```

Effectively, `clang` is being told off for not linking with `libSystem.dylib` in the executable it produced.
`libSystem.dylib` is the [C standard library] dynamic library file on macOS.

[C standard library]: https://en.wikipedia.org/wiki/C_standard_library

While most C executables will link to the standard library, this is not the case for the executable used in the `meson` sanity check, which is compiled from the following C file:

```c
int main(void) { int class=0; return class; }
```

Meson performs a simple sanity check compilation like this for the compilers it discovers before it tries to use them in a real build.

Our fix avoided the error by not stripping `libSystem.dylib`, even when it is unused and `-Wl,-dead_strip_dylibs` is set, by adding the following to the condition for a dylib to be removed:

```cpp
&& strncmp(aDylib->installPath(), "/usr/lib/libSystem.", 19) != 0
```

### What did we learn?

In the process of writing this post, I definitely learned a thing or two about compilers, linkers, dylibs, and such.

But the main takeaway for me is just how fast we were able to fix this (on the face of it, disastrous!) problem.
It speaks to the power of free and open-source software — and especially to community-owned software distributions like conda-forge — that if somebody (or some big company) breaks your software, you can go and fix it yourself (*/with the indispensable help of the conda-forge core team, who keep conda-forge running, often on a volunteer basis!*), right down to the fundamental tools like the C compiler!

Thanks again to [@isuruf], [@h-vetinari], [@pavelzw], and [@wolfv], who all helped in some way to get this fix in before it became a problem in the real world!

[@wolfv]: https://github.com/wolfv
