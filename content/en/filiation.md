---
title: "Bibliography: what this was read from, and what each one could not give"
weight: 15
---

# Bibliography {#filiation}

go-zpkg is not a fork of anything. It was written from a **bibliography** —
[pkgx](https://pkgx.sh), [Nix](https://nix.dev), [Guix](https://guix.gnu.org),
[Spack](https://spack.readthedocs.io), [Homebrew](https://docs.brew.sh),
Debian's `apt` and Red Hat's `rpm`, plus Gentoo and Arch where their answers
were the sharpest. Each one was read for what it does well, and no single one
of them is the parent.

pkgx is the entry people assume is a parentage, so it is the one worth stating
plainly: this project began by reading pkgx's format and still reads its
pantry today, which is a **dependency being wound down**, not a lineage. It is
not compatible with pkgx and does not try to be — the
[break](../the-break/) page lists what was dropped and why.

⛔ **And one entry is readable where pkgx's is not.** `Homebrew/homebrew-core`
is BSD-2-Clause, `NixOS/nixpkgs` is MIT, `spack/spack-packages` is
Apache-2.0 — all three grant what `pkgxdev/pantry` does not grant at all. That
is why a recipe of ours can *cite* them, per package, in its `provenance`
block:

```hcl
consulted = ["homebrew-core Formula/z/zlib.rb",
             "nixpkgs pkgs/development/libraries/zlib/default.nix",
             "pkgsrc devel/zlib/Makefile"]
```

A bibliography is only usable if its entries can be read. That is the whole
difference between a source and a dependency.

Four things this needs are unreachable from pkgx's design rather than missing
from its implementation, and all four are ordinary in Nix, Guix and Spack.
This page says which, and what each one costs. The same reading is what
decided the digest-at-record-time rule (Guix's `guix refresh`), the rebuild
counter (Homebrew's `rebuild`, Debian's `+bN`), the fingerprint requirement
(Arch's `validpgpkeys`) and the concrete spec (Spack's).

The honest summary: pkgx is excellent at *one build of each version*. Every
difficulty below is a consequence of that one assumption, and it is an
assumption an HPC site cannot make.

## 1. A package specification with no room for build options {#no-variants}

pkgx's own corpus — `docs.pkgx.sh/llms-full.txt`, read 2026-11-04 —
describes a grammar of **a name and a version constraint**, and nothing else:

```
node                  nodejs.org            github.com/gabime/spdlog
node^20.1.3           deno=1.35.3           "postgres>=12<14"
+openssl                                    # add to the environment
```

Against that page: *"The document describes no syntax for build options,
variants, or feature flags. It describes no syntax for compiler selection."*
In the C example, `+clang` is an ordinary `+pkg` addition — a package added to
the environment, not a compiler chosen for a build.

Compare Spack, whose
[spec syntax](https://spack.readthedocs.io/en/latest/spec_syntax.html) is built
around exactly this:

```
+debug                 # enable a boolean variant
~debug                 # disable it
compression=gzip       # single-valued
fabrics=verbs,ofi      # multi-valued: at LEAST these
fabrics:=verbs,ofi     # multi-valued: ONLY these
```

and Guix, whose
[package transformation options](https://guix.gnu.org/manual/html_node/Package-Transformation-Options.html)
do the same job from the other end — `--with-input` swaps a dependency through
the whole graph, `--with-c-toolchain` rebuilds a package and its dependents
with a different toolchain (the manual warns that this *"minimizes
incompatibility risks but cannot entirely eliminate them"*, naming the C++
standard library and OpenMP runtimes), and `--with-configure-flag` appends to
a package's configure flags.

This is not decoration. `+cuda`, `~mpi`, `fabrics=verbs,ofi` are the vocabulary
an HPC site uses to say what it wants, and without them there is no way to ask
for it.

go-zpkg adopts Spack's spelling rather than inventing one, with one
disambiguation forced by the filiation: **`~` means two things**. In pkgx it is
a version range (`~5.44` is `>=5.44 <5.45`); in Spack it disables a variant
(`~cuda`). A digit after the tilde is a range, a letter is a variant.

## 2. A store path that cannot hold two builds of one version {#one-build-per-version}

The same pkgx page states the layout plainly:
`~/.pkgx/<fully-qualified-name>/v<version>/`, with `v*`, `v1` and `v1.2`
symlinks beside it. Deno 1.2.3 is at `~/.pkgx/deno.land/v1.2.3`.

A path keyed on the version **is a function of the version alone**, so two
builds of one version have one address. There is no bug to fix here; there is
simply nowhere to put the second one. It is the direct consequence of §1 —
with no way to *ask* for two builds, there was never a reason to *store* two.

Nix solves it with a digest in the path: a store path is a 20-byte digest in a
custom Base32 (*"20 arbitrary bytes become 32 ASCII characters"*) followed by a
hyphen and a readable name —
`/nix/store/b6gvzjyb2pg0kjfwrjmg1vfhh54ad73z-firefox-33.1`
([store path specification](https://nix.dev/manual/nix/2.28/store/store-path)).

Spack solves it with a DAG hash, computed from the JSON serialization of a
concrete spec's node dicts, and uses it *"both to compare specs and to create
unique installation directories for every possible configuration"*. Its manual
shows two `mvapich2@2.2%gcc` installations distinguished only by hash
(`qmt35td`, `er3die3`), and `/\<hash\>` substitutes for a spec in any command.

go-zpkg extends the pkgx layout rather than replacing it:

```
<store>/<project>/v3.3.11                      identity not recorded
<store>/<project>/v3.3.11-habcdefghij012345    THIS build of 3.3.11
```

The bare form is kept and it is not a default. It is the honest statement that
a bottle arrived without a spec annotation — mirrored, or built before the
factory published one — and that nothing on disk can say which build it is.
Minting a hash for it would put a claim in the path that no witness supports.

The hash is 16 characters of lowercase base32 because an OCI tag may contain
`[a-zA-Z0-9_][a-zA-Z0-9._-]{0,127}` and **not** `+`, so Nix's and Spack's
spellings of a spec cannot survive a round trip through a registry tag. The
same suffix is used in the tag and in the directory name, from one constant, so
the two cannot drift into two spellings of one idea.

## 3. An identity that is only a name {#identity}

Once a build is addressable, it needs to be *identifiable*: what makes this
bottle this bottle. go-zpkg hashes a **concrete spec** — platform, declared
variants, resolved closure, recipe plan digest, graft origin — in a
length-prefixed canonical form, so no two different specs can serialise to the
same bytes.

That is Spack's answer, arrived at the same way. Spack's hash
[was changed](https://github.com/spack/spack/pull/28504) to include build, run
and link dependencies *and* a canonical hash of the `package.py` file; the
older hash had excluded build dependencies. The lesson it encodes — that the
recipe is part of the identity, not merely its inputs — is one we took rather
than relearned.

pkgx has no such object, and could not use one: with no variants and no place
to put a second build, the identity of a package *is* its name and version.

## 4. A distribution that cannot be verified {#verification}

pkgx's `dist.pkgx.dev` serves tarballs over HTTPS. HTTPS authenticates the
*server*, not the *artifact*: it says the bytes came from that host today, not
that a particular build produced them.

go-zpkg publishes to an OCI registry with a Sigstore signature and an SLSA
provenance statement per bottle, and verification is on by default — a bottle
that cannot be verified is not installed, rather than installed with a warning.
Mirrored bottles are distinguishable from built ones by their provenance:
ours carries `externalParameters` and a versioned builder id, a mirror's
carries `{}` and an unversioned one.

pkgx remains reachable, as a **choice** rather than a default:
`ZPKG_DIST=https://dist.pkgx.dev` with `ZPKG_VERIFY=0`, which is the honest
pairing — unsigned distribution, verification off, said out loud.

## What this costs, and what we keep {#cost}

**Dropped:** four inherited decisions, each with the defect that made
dropping it cheaper than keeping it — [The break]({{< relref "the-break.md" >}})
goes through them one at a time. The largest:

The shared store. go-zpkg installs into `~/.zpkg`, not `~/.pkgx`.
Two reasons, and the second is the stronger: a hash in the path is not
something pkgx can read, so the trees could not stay interchangeable anyway —
and `~/.pkgx` is *pkgx's* directory, which pkgx can be installed into on
the same machine. Sharing it was the collision the rename existed to remove,
left in the one place it does the most damage. Nothing is migrated: an
existing `~/.pkgx` is as much pkgx's as ours, and `ZPKG_DIR=~/.pkgx` keeps
using it for anyone who wants that.

**Kept, for now:** the bottle format, the dependency grammar, the relocatable
prefix — and the pkgx pantry itself, which `PantryBase` still reads at resolve
time with our own recipes merged over it.

⛔ **That last one is a dependency being wound down, and the plan that used to
be written here is superseded.** This page said the pantry's recipes were
"converted to HCL2 rather than forked, so a periodic resync stays possible".
That is no longer the decision. `pkgxdev/pantry` carries **no licence** — `git
ls-files` over all 2453 of its files returns no `LICENSE` and no `COPYING`
(go-zpkg/packages#431) — so there is no established basis on which to convert
it at all. The sovereign pantry is **written, not converted**: each recipe
from the project's *own* build files, with a `provenance` block naming them
and a signature-verified digest (go-zpkg/packages#440).

The distance is measured rather than asserted: `zbk lint --require-standalone`
counts how many of our recipes resolve without the pantry underneath, and
`--max-fragments N` is a ratchet that refuses both an increase **and** a
ceiling nobody lowers.

**Not kept:** compatibility. `package.yml` and the JSON Schema derived from
libpkgx's parser are on the way out with the rest of it; `package.hcl` and its
own schema are the format this project has.

**Not claimed:** go-zpkg does not have Nix's or Guix's functional purity. But
"not sandboxed" and "no time machine" are both vaguer than the code deserves,
so here is what is actually there and what is actually missing.

### The build *is* isolated, and here is how much {#isolation}

A factory build does not run on the host. `builder/sovereign-build.sh` enters
`unshare --mount --pid --fork` and then `chroot`s into a rootfs staged
entirely from our own bottles — that is what "FROM scratch build" means here:
neither the host's libc nor the host's compiler takes part, and the toolchain
is a reviewed, committed list (`builder/toolchain.txt`, now also a `members`
set in the pantry overlay) rather than whatever the distribution ships.

What that does **not** include, measured by searching for it rather than
assuming: no network namespace, no user namespace, no seccomp profile. A build
can reach the network. Nix's sandbox denies it outright and admits only
fixed-output derivations whose hash is declared in advance.

The gap is narrower than that sounds, because sources are pinned by content:
an archive by digest, a git recipe by commit (`bk`'s `source.go`,
`fetch/fetch.go`). What is missing is the *enforcement* — nothing stops a
build step from fetching something nobody declared.

### What a time machine would still need {#time-machine}

`zbk lock` is already most of the way, and it was written against Guix's own
statement of the problem — *"to reproduce a profile bit-for-bit, manifests
alone might not be enough"*. It records the pantry revision, the overlay
revision, every resolved version, a per-project spec hash over
platform + version + parsed recipe + dependencies, and the version of `bk`
that produced it.

It records more than a Guix manifest has to, and for a reason: a pkgx recipe's
`versions:` asks GitHub for tags **at resolve time**, so pinning the pantry
revision is not enough. The same pantry commit resolved `tcl` to 9.0.4 and
then to 9.1.0 four hours apart. The resolved version has to be written down or
it is not pinned at all.

Four things stand between that and `guix time machine`:

1. **The lock pins inputs, not output bytes.** `zbk lock`'s own documentation
   says so rather than implying otherwise: a bottle digest would pin them, and
   a lock taken before anything is built has no output to name.
2. **Nothing measures reproducibility.** No build runs twice and compares. Until
   it does, "the same inputs give the same bottle" is a hope with no instrument
   behind it — and `SOURCE_DATE_EPOCH` is not set anywhere.
3. **The lock does not name the builder image** it was built in, only the
   toolchain list that image is made from.
4. **Nothing re-executes the recorded `bk`.** The version is written down;
   fetching that version and running it is still a human step. `guix time
   machine` does exactly this and it is the part that makes the rest usable.

(1) and (3) are bookkeeping. (2) is the one that decides whether any of it is
true, and it is the next thing to build.

## Where each source is visible in the code {#visible}

Read as a bibliography: each line names what was taken and from where, which
is also the list of what a sovereign tree still has to replace.

| | | |
| --- | --- | --- |
| `bottle/pkgx.go` | the version grammar, constraints and bottle protocol | pkgx |
| `bottle/concrete.go` | the concrete spec and its hash | Spack |
| `bottle/storepath.go` | the store layout, with a hash where there is one | pkgx, plus Nix's reason for the hash |
| `bottle/builds.go` | the rebuild counter `_N` | Homebrew's `rebuild`, Debian's `+bN` |
| `bottle/oci.go` | the bottle as an **OCI artifact** rather than a tarball on a host | Homebrew's bottles, `oci.go:9` says "Homebrew-style" |
| `provenance { consulted }` | the packaging read for reference, named per recipe | homebrew-core (BSD-2), nixpkgs (MIT), spack-packages (Apache-2.0), pkgsrc — all readable, unlike `pkgxdev/pantry` |
| `zbk sourcepin` | the digest checked when it is RECORDED, not at every build | Guix's `guix refresh`; the mandatory fingerprint is Arch's `validpgpkeys` |
| `PantryBase` | **the pkgx pantry, read directly — the dependency being wound down** | pkgx |
| `ZPKG_DIST` | the distribution, defaulting to the signed registry | ours |
