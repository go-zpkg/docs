---
title: "Build isolation"
weight: 80
---

# Build isolation {#build-isolation}

Packages are only as trustworthy as the environment that produced them. The factory
builds in two phases, one shipped and one experimental.

## Phase A — debian container (shipped) {#phase-a-debian-container-shipped}

Builds run inside a pinned `debian:stable-slim` container rather than on the drifting
GitHub-runner host. This gives:

- a **controlled, known glibc floor** instead of whatever the runner ships;
- **no host-tool leakage** into the build — only the declared base plus zpkx-sourced
  dependencies are present;
- reproducible output across runs.

This is the isolation model in use for every published package today.

## Phase B — FROM-scratch zpkx-glibc toolchain {#phase-b-from-scratch-zpkx-glibc-toolchain-experimental}

Phase A still links against the container's glibc. Phase B goes further: build against
zpkx's **own** glibc toolchain so packages are truly self-contained and run on a
literally-empty `FROM scratch` base with no system libc at all.

It is **shipped, and it is the default**. The factory's `sovereign` input says
so in its own description: *blank* builds inside the `FROM scratch` rootfs
`zpkb builder` stages, "so neither the host's libc nor its compiler takes
part", and `0` is what falls back to the runner's distribution.

- `zpkb build --libc=zpkx` and `zpkb factory --libc=zpkx` target the
  `gnu.org/glibc` artifact — its crt objects, its libc, its dynamic linker —
  instead of the container's;
- `--glibc <version>` pins that sysroot to an exact glibc line;
- `zpkb builder` stages the rootfs itself: static Go binaries plus toolchain
  artifacts from the signed registry, nothing else;
- design note:
  [`zpkb/docs/from-scratch-toolchain.md`](https://github.com/go-zpkg/zpkb/blob/main/docs/from-scratch-toolchain.md).

**Corrected 2026-10-27.** This page previously described Phase B as
experimental, living on branch `feat/zpkx-glibc-toolchain`, and gated behind
a `BK_ZPKG_LIBC=1` environment variable. All three were out of date: the work
is on `main`, the gate is the `--libc` flag above, and no such variable appears
in zpkb's code — only inside the design note that first proposed it, where the
rename has since spelled it `ZBK_ZPKG_LIBC`. The old spelling is kept in this
paragraph because the paragraph is about what this page used to say.

### What a sovereign toolchain does not reach {#what-a-sovereign-toolchain-does-not-reach}

The sysroot, the `-isystem` directories and the runtime choice reach a build
**through the compiler driver** — `cc`, which is zpkb's own shim and re-execs
`$ZBK_CC`, or `$CC`. A tool that drives the compiler some other way sees none
of it.

`bindgen` is that tool: it calls libclang in-process. A Rust crate generating
bindings therefore got the container's idea of a libc, or none at all —

```text
/zpkx/openssl.org/v3.6.4/include/openssl/asn1.h:27:10:
  fatal error: 'stdio.h' file not found
```

— in a build where openssl 3.6.4 had been found, probed and configured
correctly a few lines earlier. Since zpkb v0.43.0 sovereign mode also exports
`BINDGEN_EXTRA_CLANG_ARGS`, the variable bindgen's own documentation names
for exactly this, carrying the header-finding flags only and appended to
whatever the recipe set.

See [Running zpkx packages on FROM scratch]({{< relref "from-scratch.md" >}}) for why a system-linked
package needs closure completion to run on scratch, and what Phase B removes.
