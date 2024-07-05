---
title: "Porting youtube-unthrottle to OpenBSD"
date: 2024-07-04T21:45:00-07:00
---

### Motivation

Given that
[youtube-unthrottle]({{< relref "2024-06-29-youtube-unthrottle" >}})
parses HTML and executes JavaScript fragments, it seemed appropriate to
explore sandboxing techniques that might mitigate bugs in handling
untrusted inputs.

Along these lines, I'd recently been reading about OpenBSD's
[pledge()](https://man.openbsd.org/pledge.2) and
[unveil()](https://man.openbsd.org/unveil.2) APIs on
[LWN](https://lwn.net/Articles/767137).
These APIs seemed to be designed to make application-layer sandboxing
straightforward to implement, such that a program can voluntarily drop
privileges early in main() to ensure later code, even if part of an
exploit, is limited in its powers. Instead of the system administrator
configuring a sandbox around the application through blackbox-style
restrictions (like a syscall filter), the application developer makes the
sandbox "just work" out of the box.

Sidenote: I'm both the developer and administrator in this scenario, but
I'm (probably) more capable as the former than I am as the latter.

### Starting with Landlock

As an initial experiment with sandboxing, I started with another
technology I learned about through LWN,
[Landlock](https://lwn.net/Articles/859908).
Although the Landlock API appeared more complex than unveil(), it had the
advantage of being supported on Linux, making it available for immediate
use in my dev environment. The
[API guidance](https://docs.kernel.org/userspace-api/landlock.html)
and permissively-licensed
[sample code](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/samples/landlock/sandboxer.c)
were clear and helpful. With some guess-and-check work, I ended up with a
(seemingly) working
[sandbox API](https://gitlab.com/ewoo/youtube-unthrottle/-/merge_requests/5/diffs#e6fd75f89c08b27acdc44904d8a9ec45cd0154f9)
for youtube-unthrottle, with an associated
[Linux implementation](https://gitlab.com/ewoo/youtube-unthrottle/-/merge_requests/5/diffs#3a9491604e2d44812c7067207d1c15b8afc71360_0_117)
and barebones test rig:

```sh
$ ./youtube-unthrottle --try-sandbox
landlock.c:150: ruleset_apply() succeeded
sandbox.c:30: sandbox verify: allowed /tmp
sandbox.c:30: sandbox verify: allowed /etc/resolv.conf
sandbox.c:30: sandbox verify: allowed /etc/ssl/certs/ca-certificates.crt
sandbox.c:45: sandbox verify: blocked /etc/passwd
sandbox.c:67: sandbox verify: allowed connect()
landlock.c:150: ruleset_apply() succeeded
sandbox.c:30: sandbox verify: allowed /tmp
sandbox.c:38: sandbox verify: blocked /etc/resolv.conf
sandbox.c:38: sandbox verify: blocked /etc/ssl/certs/ca-certificates.crt
sandbox.c:45: sandbox verify: blocked /etc/passwd
sandbox.c:67: sandbox verify: blocked connect()
```

### Onward to OpenBSD

Based on this positive result, I believed it would be worthwhile to port
to OpenBSD, in order to see if a second implementation of the sandbox
module would work as expected.

This required an OpenBSD dev environment. I am happy to report that I
found the setup experience pleasantly straightforward, owing much to the
helpful ArchWiki articles for
[KVM](https://wiki.archlinux.org/title/KVM#Checking_support_for_KVM) and
[QEMU](https://wiki.archlinux.org/title/QEMU#Creating_new_virtualized_system),
as well as the OpenBSD
[install guide](https://www.openbsd.org/faq/faq4.html#Download).
These prerequisite tasks also gave me a reason to test GitLab's built-in
wiki feature to
[record the process](https://gitlab.com/ewoo/youtube-unthrottle/-/wikis/OpenBSD-VM-Dev-Environment).

After this setup, the sandbox implementation for OpenBSD proceeded
smoothly, involving a few
[platform ifdefs and API calls](https://gitlab.com/ewoo/youtube-unthrottle/-/merge_requests/6/diffs#29f8b29be2a5ce2a72602cc233576378a3151d82_88_93).

### Musings

Before starting this work, the pledge() and unveil() APIs struck me as
elegant; the practical experience of programming with them confirmed the
initial impression. It seems there really is something to the idea of
developing a free software OS's kernel and userspace together, eschewing
binary compatibility, and rebuilding the world (so to speak) when
fundamental kernel APIs change.

I wonder, in particular, if OS developers in such an environment can aim
for simplicity and minimalism of APIs, knowing they can learn from future
results and continue changing the kernel, without the burden of supporting
every old variant of the functionality in question.

I would not expect this approach to work for an OS that aims to host
mainly proprietary software, where the OS developer cannot unilaterally
refactor and rebuild all userspace consumers after making a syscall design
change. Still, it's good to know that for the price of a proprietry
ecosystem atop one's system (or the potential for one), it is possible to
gain a potentially richer, more fertile learning platform.

### Cleanup and Build Systems

I did encounter at least one hitch in my OpenBSD dev environment: the
build toolchain. In particular, I was a bit confused by whether the
various warnings and behaviors I encountered were the result of using
clang (the default C compiler on OpenBSD) compared to gcc on Linux, using
an older version of clang instead of a more recent version, and so forth.

None of this was too surprising, as I had initially written a Makefile
that likely only worked with a recent version of gcc and depended on GNU
Make specifically (incompatible with other Make implementations).

I worked around this initially, by installing gcc (aka `egcc`) and GNU
Make (aka `gmake`) on my OpenBSD VM. This was good enough to unblock my
experimentation with pledge() and unveil(), allowing me to confirm that
it did in fact make sense to port to OpenBSD.

Once I knew I'd want to run on OpenBSD going forward, it made sense to go
back and consider making the build process on OpenBSD more maintainable.
Doing so meant the project build would need to handle both OS-dependent
build decisions and compiler-dependent (even compiler-version-dependent)
options.

The project Makefile had already become a bit
[idiosyncratic and brittle](https://gitlab.com/ewoo/youtube-unthrottle/-/merge_requests/6/diffs#836efb6e25a091dcb4ff8e1dbb2f0be6a5cbf14c_17_7)
by this point, even just to support gcc on OpenBSD (not even clang).

Taken all together, it probably made sense to adopt a meta-build system.
I got the sense this is a significant decision for large, multi-platform,
multi-person projects, but for this small program, I figured CMake plus
the native OS package manager (`pacman`, `apt`, `yum`, `pkg_add`, etc)
would be sufficient. The former seems to accomodate all but the largest
C/C++ monorepos and codebases, and the latter should (I hope) be easy to
swap with a proper dependency manager like
[CPM](https://github.com/cpm-cmake/CPM.cmake) or
[Conan](https://github.com/conan-io/conan)
if this project's third-party dependencies ever grow considerably.

As a bonus, switching from a Makefile to a
[CMakeLists.txt](https://gitlab.com/ewoo/youtube-unthrottle/-/merge_requests/9/diffs#9a2aa4db38d3115ed60da621e012c0efc0172aae)
to improve the maintainability of the OpenBSD build in general also
delivered support for OpenBSD clang in particular. It seemed virtuous to
enable building with the OS's default build toolchain, to be consistent
with platform idioms if nothing else.

Happily, by adding support for clang on OpenBSD through CMake, the project
also gained support for clang on Linux and thereby support for
[clang under GitLab CI](https://gitlab.com/ewoo/youtube-unthrottle/-/merge_requests/9/diffs#587d266bb27a4dc3022bbed44dfa19849df3044c_17_20).
Having these positive results flow naturally from one initial decision was
definitely encouraging.

If the project ever adopts clang-specific tooling, like
[thread safety analysis](https://clang.llvm.org/docs/ThreadSafetyAnalysis.html)
or
[libfuzzer](https://llvm.org/docs/LibFuzzer.html),
these most recent changes might pay further dividends. Time will tell!
