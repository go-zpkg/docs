---
title: "The break: what we dropped from pkgx, and what it fixed"
weight: 16
---

# The break {#the-break}

[Filiation]({{< relref "filiation.md" >}}) says what go-zpkg inherits from
[pkgx](https://pkgx.sh) and what pkgx cannot do by design. This page is the
other half: the inherited things we **removed**, each with the defect that
made removing it the cheaper option.

Nothing here is a complaint about pkgx. The first four are reasonable
decisions for a tool that builds one version of each package once, and wrong
ones for a tool that must hold two builds of one version and say which is
which. The fifth is not a design decision at all — it is a missing licence,
and it is the one that makes the others unavoidable.

## 1. A store path keyed on the version alone {#store-path}

**Inherited:** `~/.pkgx/<project>/v<version>/`, with `v1`, `v1.2` and `v*`
aliases beside it. pkgx's own documentation: *"All packages are
encapsulated in individual, versioned folders in `~/.pkgx`."*

**The defect.** A path keyed on the version is a function of the version
alone, so two builds of one version have one address. Installing the second
met the first and the installer refused:

```
already holds other content … the bottle carries v3.3.11
```

That refusal is correct behaviour on a wrong layout. `+cuda` and `~cuda` are
not a sequence, there is no newer one, and nothing in the path said which was
there.

It was not hypothetical. A rebuild is published as `3.3.11_1` and still
unpacks to `v3.3.11`, so installing it into a store holding the original
renamed onto an occupied directory, reported success, and installed nothing —
the caller then ran a binary under a prefix that had never been created and
got a bare `no such file or directory`.

**Now:** every installed tree is `v<version>-h<hash>`. The hash is the
build's identity — platform, resolved version, the canonical form of the
recipe, and the resolved prefix of every dependency. Two builds of one
version are two directories, and each says which it is.

**What survives, and why.** `v<version>` remains **as a symlink**, because it
is in the published binaries. Read out of a real bottle:

```console
$ otool -l …/zlib.net/v1.3.2/lib/libz.1.3.2.dylib
  name @rpath/zlib.net/v1.3.2/lib/libz.1.dylib
```

Every binary that links zlib names that path. A store without it breaks all
of them at load time, far from the cause. The hash goes on the real
directory; the published path keeps resolving.

That seam closes when the registry is rebuilt: a bottle built with a hashed
prefix carries the hash in its own `@rpath` references, and needs no alias at
all.

## 2. Somebody else's store directory {#store-directory}

**Inherited:** `~/.pkgx`, and `~/.pkgx/config.hcl2` beside it.

**The defect.** `~/.pkgx` is *pkgx's* store. Once our runtime is `zpkgx`,
`pkgx` can be installed on the same machine and the same PATH —
which is the entire reason for the rename. Two implementations writing one
directory, under two layouts that are no longer even the same, is the
collision the rename existed to remove, left in the one place it does the
most damage.

**Now:** `~/.zpkg`, `~/.zpkg/config.hcl2`, and the store's own marker file is
`.zpkg-platform`. **Nothing is migrated, on purpose**: an existing `~/.pkgx`
is as much pkgx's as ours, and nothing in it says which bottles are
whose. `ZPKG_DIR=~/.pkgx` keeps using the old one for anyone who wants that.

## 3. A vendor namespace that is not ours {#namespaces}

**Inherited:** `application/vnd.pkgx.bottle.layer.v1.tar+zstd`,
`application/vnd.pkgx.source.layer.v1`, `dev.pkgx.source.uri`.

**The defect.** Checked against pkgx's own corpus rather than assumed:
pkgx distributes **plain HTTPS tarballs** from
`dist.pkgx.dev/<pkg>/<os>/<arch>/` with `.sha256sum` and `.asc` beside them.
It defines **no OCI registry, no artifact type and no media type at all**.

The OCI packaging is entirely ours — our factory invented it — and we had
minted its names inside a vendor space whose owner never agreed to them. A
name that looks like yours because you are the only one using it is not
yours.

Our own keys had the same flaw one level down: the annotations were
`org.go-pkgx.*`, and the rename turned them into `org.go-zpkg.*`. A
reverse-DNS key is a claim on a **domain**, and nobody here has registered
`go-pkgx.org` or `go-zpkg.org`. The rename carried an unbacked claim across
rather than fixing it.

**Now:** `application/vnd.go-zpkg.*` for media types and
`io.github.go-zpkg.*` for annotations — the reverse-DNS of a host that
demonstrably answers, which is the convention OCI itself uses
(`org.opencontainers.image.*`) applied honestly.

**Read compatibility is asymmetric, and deliberately so.** Layers are matched
by their codec suffix (`tar+zstd`), never by the whole string, so both
spellings read without the reader being told about either. Annotations are
matched by exact key, so the older spellings are still read — including the
intermediate `org.go-zpkg.*`, which shipped in five releases. The failure
mode is why: a missed annotation key does not raise an error, it quietly
concludes the bottle has no glibc flavour, no declared sonames and no
recorded spec.

## 4. A distribution that cannot be verified {#verification}

**Inherited:** `https://dist.pkgx.dev`, tarballs over HTTPS.

**The defect.** HTTPS authenticates the *server*, not the *artifact*: it says
the bytes came from that host today, not that a particular build produced
them.

**Now:** an OCI registry with a Sigstore signature and an SLSA provenance
statement per bottle, and verification **on by default** — a bottle that
cannot be verified is not installed, rather than installed with a warning.

A second distribution can still be named, with `ZPKG_UPSTREAM_DIST`, and it is
**empty by default**: a project we do not carry says so rather than resolving
to somebody else's bottle. This page used to name `https://dist.pkgx.dev` here
as the thing to point it at, paired with `ZPKG_VERIFY=0`. That pairing is
gone. It was reasonable while this was a mirror with a gap-filler; it is wrong
now that the two are no longer compatible (see §5), and telling someone to
turn verification off in order to reach it was the worst version of the
advice.

## 5. The recipes themselves {#the-pantry}

**Inherited:** `pkgxdev/pantry`, read over HTTPS at resolve time and used as
the **base** that our own overlay is merged over.

**The defect is not technical.** `pkgxdev/pantry` carries **no licence**: the
repository declares none, there is no `LICENSE` or `COPYING` file, and neither
its `README.md` nor its `AGENTS.md` states one. No licence does not mean public
domain — the default is exclusive copyright, and GitHub's terms grant a visitor
only the right to view and fork a public repository *through the Service*, not
to redistribute its contents.

The Apache-2.0 on `pkgxdev/pkgx` does not reach it. That licence's own §1 is
why: *"'Work' shall mean the work of authorship … made available under the
License, **as indicated by a copyright notice that is included in or attached
to the work**."* The pantry carries no such notice, and §4(a) would in any case
oblige a redistributor to ship a copy of the licence, which it does not.
`pkgxdev/brewkit` is in the same position.

So a tool that reads those recipes to build from, and publishes the result, is
building on something nobody granted. For comparison, the systems whose
packaging we read for reference do grant it: `Homebrew/homebrew-core` is
BSD-2-Clause, `NixOS/nixpkgs` is MIT, `spack/spack-packages` is Apache-2.0.

**Now:** our recipes are **written**, not converted, from each project's own
build system — its `configure`, `CMakeLists.txt`, `Makefile`, `INSTALL`,
`README`. Each records which of those files it was derived from:

```hcl
provenance {
  read      = ["README", "configure", "Makefile.in", "zlib.h", "ChangeLog"]
  consulted = ["homebrew-core Formula/z/zlib.rb",
               "nixpkgs pkgs/development/libraries/zlib/default.nix",
               "pkgsrc devel/zlib/Makefile"]
  note      = "./configure --prefix then make install, as the README documents"
}
```

The two lists are kept apart because the distinction is the whole point.
`read` is what the build **derives from**. `consulted` is what was **learned
from**: reading how six other distributions solved the same build is
bibliography, and their packaging is where the hard-won facts live — which
`configure` flag a project needs on darwin, which dependency it does not
declare, which test hangs under qemu. A fact about the software has no author.
What is not taken is their expression.

It is a citation list rather than a confession, and it is what makes the
distinction auditable: a recipe that cites six distributions and derives from
the project's own `configure` is visibly doing the right thing, while one that
cites nothing and happens to match Debian's rules line for line is not — and
nobody could tell them apart without the field.

**Two gates enforce it**, both per tree, because a pantry recipe has no
reason to carry any of this:

- `zbk lint --require-provenance` fails a recipe of ours that records no
  provenance.
- `zbk lint --require-standalone` fails a recipe that cannot resolve **alone**
  — one that needs another recipe merged underneath it. That is the question
  that decides whether a tree can be a pantry at all, and the answer today is
  uncomfortable: of 228 recipes across the three trees that are ours, **31**
  stand alone. The rest are overlay *amendments*, which was the right shape for
  a tree merged over somebody else's and is unusable as a base.
  `--max-fragments N` is the ratchet that lets the check run before the work is
  done: more than `N` fails as a regression, and **fewer than `N` fails too**,
  because a ceiling nobody lowers stops describing the tree.

## What this costs {#cost}

Every bottle published before the break has to be **rebuilt**, not copied. A
copy would carry the old internal paths, the old media types and no identity
— it would move the defects rather than end them.

And now the recipes have to be **written** as well as the bottles rebuilt,
which is the larger half and the one with no mechanical shortcut: a lossless
conversion of an unlicensed recipe is a *more* faithful derivative, not a less
one. 31 of roughly 1 700 are done.

That is still the reason the break happens now rather than later. The registry
is the largest thing that would have to change, and it is cheaper to do it once
than to carry five inherited defects indefinitely and explain them to everyone
who meets one.
