---
title: "Home"
weight: 10
---

# go-zpkg {#go-zpkg}

A **dependency-free, pure-Go** (`CGO_ENABLED=0`) package manager — a single
static binary that installs **and runs** packages on a `FROM scratch` image,
with no runtime dependencies of its own: one static ~9 MB binary, no libc,
no interpreter.

It is not a fork of anything. It was written from a **bibliography** — pkgx,
Nix, Guix, Spack, Homebrew, Debian's `apt`, Red Hat's `rpm`, and Gentoo and
Arch where their answers were sharpest; [Bibliography]({{< relref
"filiation.md" >}}) names what each one gave. **It is not compatible with
pkgx** and does not try to be.

The recipe format and the distribution are ours: `package.hcl` with its own
schema, and a signed registry. ⛔ The **pantry is not yet**: `PantryBase`
still reads `pkgxdev/pantry` at resolve time with our recipes merged over it,
which is a dependency being wound down — the sovereign pantry is written, not
converted, because that corpus carries no licence to convert
(go-zpkg/packages#431, #440).

Four things it does, all four ordinary in Nix, Guix and Spack: build options
on a specification, two builds of one version in one store, a hashed identity
for a build, and a distribution that can be verified. See
[Addressing a build]({{< relref "addressing-a-build.md" >}}) for how a build
gets an identity and how you name the one you want.

The `z` is ZFS's: the last one you need.

## ⛔ Three commands, and why `zpkx` and `zpks` are two of them {#three-commands}

| | |
| --- | --- |
| `zpkm` | **installs** a package, into a store you keep |
| `zpkx` | **runs** one, on the fly, without installing it |
| `zpks` | **answers about** one: `ls`, `search`, `why`, `catalog`, `store`, `browse`, `compat`, `env`, `completion` |

The last two ship from one repository and share one resolver. They are
separate commands because of a grammar, not a preference.

The grammar has no verbs: the first argument that is not a flag is
a **package**, and `zpkx` runs it. Nine verbs had been added in
that position, and each took a name away from the registry. Two took names
that exist — measured against the published linux/x86-64 catalogue on
2026-10-18, **`gnu.org/coreutils` provides both `ls` and `env`** — so `zpkx
ls` listed a catalogue instead of running GNU ls, and there was no way to
ask for the other one.

```console
$ zpkx ls --version          # GNU coreutils: the first word is a package
ls (GNU coreutils) 9.11
$ zpks ls gnu.org            # what is available under a node
$ zpkx -Q ls                 # which project provides a command
gnu.org/coreutils
```

A reserved word could not have fixed it: `vercel.com/pkg` provides the
command `pkg`, and any bare word can become a package tomorrow, because the
point of the first position is that the **registry** decides what is in it.
Only a token the grammar already reserves is safe — a flag, `zpkx -Q`, or a
different binary.

## Install {#install}

**Linux / macOS** — one line, naming the release you want:

```sh
curl -fsSL https://go-zpkg.github.io/install.sh | sh -s -- zpkm v0.9.0
```

**Windows** (PowerShell) — `irm | iex` passes no arguments, so the version goes
in the environment:

```powershell
$env:ZPKM_VERSION='v0.9.0'; irm https://go-zpkg.github.io/install.ps1 | iex
```

The installer downloads the static binary for your os/arch from that
[release](https://github.com/go-zpkg/zpkm/releases), verifies it against the
release `SHA256SUMS`, and puts `zpkm` on your `PATH` (`$HOME/.local/bin`, or
`%LOCALAPPDATA%\Programs\go-zpkg` on Windows; set `ZPKM_INSTALL` to override on
Unix).

⛔ **`SHA256SUMS` is a checksum of itself.** It says the binaries match a
manifest published beside them, and whoever replaced the binaries would
replace the manifest. The half that cannot be forged from the release page is
GitHub's **provenance attestation** — signed through Sigstore, saying that the
repository's own release workflow built that file from that commit:

```sh
gh attestation verify -R go-zpkg/zpkm zpkm-linux-amd64
```

Every file of every `zpkx`, `zpks`, `zpkb`, `zpkm` and `mirror` release
published from 2026-10-20 carries one. Earlier releases carry none, and
`gh attestation verify` answers `HTTP 404` for those rather than failing
closed, so a 404 on an older binary means *unattested*, not *tampered with*.

The version is named on purpose: this line copied today and the same line
copied in six months install the same bytes, and a bad release does not reach
everyone who happens to install that hour. To track releases instead, say so —
`sh -s -- zpkm latest`, or `ZPKM_VERSION=latest`, which is also what a bare
`| sh` has always done. Re-running is the updater; it skips the download when
the target version is already installed.

To install `zpkx` (the runtime) or `mirror` instead, name it:
`sh -s -- zpkx v0.32.0`, `sh -s -- mirror v0.4.0`.

**Go users**:

```sh
go install github.com/go-zpkg/zpkm@latest
```

Once installed, `zpkm install lz4.org` verifies each artifact against the signed
registry by default.

## Usage {#usage}

```
zpkm install|i    <pkg>[@version] ...   install to /usr/local (root) or ~/.local
zpkm uninstall|rm <pkg> ...             remove an installation
zpkm shim|stub    <pkg> ...             create a shim in <prefix>/bin
zpkm list|ls                            list what's installed
zpkm outdated                           list outdated installations
zpkm update|up|upgrade                  update installations to latest
zpkm pin          <pkg>@version ...     install pinned to an exact version
zpkm run|x        <pkg> [-- args...]    run a pkg (works FROM scratch)

flags: -h/--help  -v/--version  -p/--pin  -P/--prefix DIR  -s/--from-scratch
env:   ZPKG_DIR     package store (default: ~/.zpkg)
       ZPKG_DIST    package source (default: oci://ghcr.io/go-pkgx/packages, signed)
       ZPKG_VERIFY  verify signatures, fail-closed (default: on)
       ZPKM_PREFIX default install prefix (ideal for FROM scratch)
```

The command surface is `install`/`uninstall`/`shim`/`list`/`outdated`/
`update`/`pin`, with `~/.local` for an unprivileged install and `/usr/local`
otherwise.


## FROM scratch {#from-scratch}

A scratch image whose only file is the `zpkm` binary can install and run real
packages, with no system libc:

```dockerfile
FROM scratch
COPY zpkm /zpkm
ENV ZPKG_DIR=/zpkx
ENTRYPOINT ["/zpkm"]
```

```sh
$ docker run --rm zpkm-scratch run gnu.org/bash -- --version
GNU bash, version 5.3.0(1)-release (aarch64-unknown-linux-gnu)
```

See [FROM scratch]({{< relref "from-scratch.md" >}}) for how the closure is resolved, the
conformance matrix, and the pantry-wide audit.

## Environments {#environments}

`zpkx` also carries `zpke`, a module system: named environments declared in
HCL2, `load`/`unload`/`purge` that are exact rather than incremental, and a
converter that reads a site's existing Lmod and Environment Modules files.

```console
$ eval "$(zpks env init)"
$ zpke load cfd
```

See [Environments and HPC]({{< relref "environments.md" >}}).

## WebAssembly {#webassembly}

`wasip1/wasm` is a platform of the factory like any other, with one property
that changes everything downstream: a `wasm32-wasip1` module is **statically
linked**, so one file is the whole program and there is no closure to resolve.
Do not try to resolve one: a `wasip1/wasm` catalogue entry can name a package
that is not published for that platform and that the module does not contain.

See [wasip1/wasm]({{< relref "wasm.md" >}}).
