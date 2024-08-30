---
title: "Gathering Code Coverage for Sandboxed Processes"
date: 2024-08-29T23:00:00-07:00
---

### Motivation

While gathering code coverage for
[youtube-unthrottle]({{< relref "2024-06-29-youtube-unthrottle" >}}),
I encountered an interesting catch-22: tests exercising
[sandboxing features]({{< relref "2024-07-04-porting-to-openbsd" >}})
based on
[seccomp-bpf](https://gitlab.com/ewoo/youtube-unthrottle/-/blob/main/src/seccomp.c)
and
[Landlock](https://gitlab.com/ewoo/youtube-unthrottle/-/blob/main/src/landlock.c)
could not write coverage data to disk because of the sandbox restrictions themselves.

### Code Coverage Approach #1: gcc

The default behavior of gcc- and gcov-based code coverage is to open and
write `*.gcda` coverage files on process exit.

My first thought, upon encountering a failure to write these `*.gcda`
files, was to investigate whether gcc supports options or APIs for
customizing this behavior: writing the coverage data elsewhere, opening
the coverage files earlier in process startup, or similar.

Searching on this topic led me to gcc's documentation on
[freestanding environments](https://gcc.gnu.org/onlinedocs/gcc/Freestanding-Environments.html),
which described how to work with systems that lack a filesystem to
store the resulting coverage data (seemingly equivalent to a sandboxed
program that cannot make filesystem-related syscalls).

However, I was discouraged by the following:

1. custom linker script requirement
2. explicit assumption of GNU linker in particular, with an the implied
   possibility of incompatibility with other linkers like
[gold](https://sourceware.org/git/?p=binutils-gdb.git;a=blob;f=gold/README),
[lld](https://lld.llvm.org),
and [mold](https://github.com/rui314/mold)
3. circular-seeming test procedure, involving a no-op `*.gcda` output
   from one run being passed to a second run via stdin
4. unclear encoding/decoding requirements for in-memory coverage data

### Code Coverage Approach #2: clang

Given the above, I was not confident that a gcc-based solution would be
maintainable in the longer term. At the very least, it would take some
time and experimentation for me to build a working understanding of what
each step in the gcc-based approach actually does (not obvious to me,
even after some re-reading).

Put another way, although gcc and gcov made up most of my prior
experience with C/C++ code coverage (plus some work with Coverity), the
freestanding environment requirements would be putting me in new
territory regardless of past experience with the tools more generally.
As a result, gcc and gcov had no practical familiarity advantage.

Putting it all together, research into other approaches seemed wise.

Searching for alternatives led to clang's equivalent support for
[freestanding environments](https://clang.llvm.org/docs/SourceBasedCodeCoverage.html#using-the-profiling-runtime-without-a-filesystem).

Happily, clang's code coverage procedure for freestanding environments
seemed more straightforward than gcc's, simple enough to be excerpted
here:

> The first step is to export `__llvm_profile_runtime` [...] to disable
> the default static initializers. Instead of calling the `*_file()`
> APIs [...], use the following to save the profile directly to a buffer
> under your control:
>
> Forward-declare `uint64_t __llvm_profile_get_size_for_buffer(void)`
> and call it to determine the size of the profile. You'll need to
> allocate a buffer of this size.
>
> Forward-declare `int __llvm_profile_write_buffer(char *Buffer)` and
> call it to copy the current counters to `Buffer`, which is expected to
> already be allocated and big enough for the profile.

Following these instructions and then writing the buffered data to a
file opened early in process startup (before any sandboxing has
occurred) produced a working solution in short order.

### Results

Coverage hooks:
[coverage_open()](https://gitlab.com/ewoo/youtube-unthrottle/-/merge_requests/22/diffs#62dce53b8e55ca7ae75181dfcdcbffa6c6f83061_0_47),
[coverage_write_and_close()](https://gitlab.com/ewoo/youtube-unthrottle/-/merge_requests/22/diffs#62dce53b8e55ca7ae75181dfcdcbffa6c6f83061_0_80)

Example test integration:
[./tests/sandbox/seccomp.c](https://gitlab.com/ewoo/youtube-unthrottle/-/merge_requests/22/diffs#1967e780d83df64fe354ac262d791fb9e321ac17_217_218)

Interactive usage instructions:
[README.md](https://gitlab.com/ewoo/youtube-unthrottle/-/blob/7200009a34faaa7b320aa4c2fa2e0c575d669a27/README.md#L127-135)

CI integration:
[./scripts/coverage.sh:21-41](https://gitlab.com/ewoo/youtube-unthrottle/-/blob/7200009a34faaa7b320aa4c2fa2e0c575d669a27/scripts/coverage.sh#L21-41),
[.gitlab-ci.yml:61-66](https://gitlab.com/ewoo/youtube-unthrottle/-/blob/7200009a34faaa7b320aa4c2fa2e0c575d669a27/.gitlab-ci.yml#L61-66)
