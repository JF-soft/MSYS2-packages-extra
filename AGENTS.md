# AGENTS.md --- MSYS PKGBUILD Authoring Rules

## Purpose

This document defines how human developers and coding agents must
create, review, build, test, and maintain PKGBUILDs for packages
intended primarily for MSYS2.

The target environment for `MSYS2-packages-extra` is **MSYS**. Each
recipe in this repository must build for the MSYS subsystem; native
MinGW recipes belong in the sibling `MINGW-packages-extra` repository.

These instructions apply to package recipes and their supporting files,
including patches, install scripts, local build helpers, and package
metadata.

Note: recipes for **Arch Linux proper** (AUR) are governed by a separate
document. The conventions differ deliberately; do
not carry MSYS2 rules into Arch recipes or vice versa.

## 1. Authority and precedence

Apply packaging rules in this order:

1.  **MSYS2 official documentation and repository conventions** are
    authoritative.
2.  **Arch Linux packaging guidelines** apply where MSYS2 does not
    define a different rule.
3.  Upstream project documentation determines how the software is built,
    configured, tested, and installed, provided that it does not
    conflict with MSYS2 packaging requirements.
4.  Existing local recipes are useful examples, but must not be copied
    blindly. Verify that their conventions remain correct and
    applicable.

When two sources conflict, follow the higher-priority source and record
the reason when the conflict materially affects the recipe.

Do not assume that an Arch Linux PKGBUILD can be copied unchanged into
MSYS2. MSYS2 inherits the PKGBUILD format but has its own package
naming, environment, prefix, dependency, license, and build
conventions. In particular:

- MinGW packages use `arch=('any')` plus `mingw_arch`, not the Arch
  `arch=('x86_64')` convention.
- MSYS2 uses `spdx:` license identifiers and its own license file
  layout.
- MSYS2 uses `MINGW_PREFIX`, `MINGW_PACKAGE_PREFIX`, `MINGW_CHOST`,
  `RUST_CHOST`, and `MINGW_ARCH`.
- MSYS2 uses `makepkg-mingw` for native packages.
- MSYS2 performs Windows path conversion that Arch does not have.

## 2. Repository layout and file hygiene

### 2.1 Directory layout

Keep exactly one source package per directory. In these repositories,
package directories are direct children of the repository root; do not
add an intermediate `msys/` or `mingw/` directory.

```
<repository-root>/
  .gitignore
  AGENTS.md
  LICENSE
  README.md
  <package-directory>/
    PKGBUILD
    *.patch
    *.install
    LICENSE*
    LICENSES/
    REUSE.toml
```

The files below a package directory are optional and should be included
only when required by that recipe. Name each directory after its package
identity (for example `dash` or `python-click`), keep it lowercase and
stable, and do not combine unrelated recipes.

### 2.2 Generated files

Do not treat the following as source files and do not commit them:

- `src/`, `pkg/`, and `build-*` directories;
- built packages (`*.pkg.tar.zst`, `*.pkg.tar.zst.sig`);
- source-only packages (`*.src.tar.zst`);
- extracted upstream source trees;
- caches, logs, credentials, private keys, and local configuration.

If a recipe intentionally vendors a source archive or a small required
asset, document why it is stored in the repository. Prefer downloading
it from a verifiable upstream source.

Add or maintain a `.gitignore` so generated artifacts are not tracked
by accident.

### 2.3 Encoding and line endings

- Save text files as UTF-8 **without a BOM**.
- Use **LF** line endings, including for `PKGBUILD`, `.install`,
  `.patch`, `.hook`, and shell scripts. CRLF breaks Bash, `patch`, and
  many build tools.
- Keep the `PKGBUILD` a valid Bash script; do not store it with a
  Windows executable bit or shell wrapper.

## 3. Mandatory workflow before writing a PKGBUILD

Before creating or modifying a recipe, the developer or agent MUST:

1.  Identify the software's upstream project, canonical source
    repository, release process, and license.
2.  Confirm that the project supports Windows and can be built with a
    toolchain available in MSYS2. Do not assume MSVC-only software is
    suitable for MinGW.
3.  Search the MSYS2 package repositories for an existing package or
    equivalent dependency. Avoid duplicating an existing package.
4.  Inspect the current MSYS2 package recipe for the closest build
    system and dependency pattern.
5.  Check whether the package already exists in Arch Linux or the AUR.
    Use that recipe as a secondary reference for source selection, build
    options, and dependency discovery---not as the MSYS2 implementation.
6.  Identify the intended MSYS2 subsystem and environment:
    - `msys` for software that depends on the MSYS runtime
      (`msys-2.0.dll`).
    - `mingw` for native Windows software that does not depend on the
      MSYS runtime.
7.  For native packages, identify the exact target environments and the
    matching package prefixes.
8.  Determine whether runtime dependencies are direct dependencies of
    the resulting package, rather than merely build-time or transitive
    dependencies.
9.  Check upstream build and test instructions, feature flags, vendored
    dependencies, generated files, and license files.
10. Present a short implementation plan and unresolved questions before
    making substantial changes when working under a plan-first
    development workflow.

Do not invent package names, dependency names, source URLs, checksums,
feature flags, or install paths. Verify them against the MSYS2 package
database, repository recipes, upstream source, or build output.

## 4. Choose the correct subsystem and environment

### 4.1 Decision rule

All recipes in this repository target the MSYS subsystem. When deciding
whether a new package belongs here, justify why it needs the MSYS
runtime or integration rather than placing a native Windows package
here by default. Native MinGW recipes belong in
`MINGW-packages-extra`.

A package belongs here only if at least one of the following applies:

1. The software genuinely requires the MSYS POSIX compatibility runtime
   (`msys-2.0.dll`).
2. It is deliberately part of the MSYS environment itself: the POSIX
   toolchain and base tools, scripting runtimes, shells (including ports
   such as `xonsh` that integrate into the environment, for example via
   `/etc/shells`), editors, VCS and bridging or supporting tools that are
   hard to port to native Windows.

Being portable to Windows does not decide the subsystem. Conversely,
belonging to the environment does not require depending on
`msys-2.0.dll`. When in doubt, record why a package belongs in
`MSYS2-packages-extra` rather than mechanically applying a rule.

Upstream MSYS2 guidance is to use MinGW for software that can be built
as a native Windows package. MSYS packages need a subsystem-specific
reason. Never link a MinGW program against an MSYS library.

### 4.2 Current environments

The following are the environments currently listed by MSYS2. Recheck
the official environment documentation when adding support for a newly
introduced environment; do not add retired or deprecated targets.

| Environment | Prefix | Package prefix | Toolchain | C library |
| --- | --- | --- | --- | --- |
| `MSYS` | `/usr` | none | gcc | cygwin |
| `UCRT64` | `/ucrt64` | `mingw-w64-ucrt-x86_64` | gcc | ucrt |
| `CLANG64` | `/clang64` | `mingw-w64-clang-x86_64` | llvm | ucrt |
| `CLANGARM64` | `/clangarm64` | `mingw-w64-clang-aarch64` | llvm | ucrt |

Notes:

- `MINGW64` is deprecated (announced 2026-03-15).
- `CLANG32` was removed in 2024; `MINGW32` is being phased out. Do not
  add either as a new target.
- UCRT64 is the default native MinGW environment; this repository
  targets MSYS and does not use `mingw_arch`.
- Never claim that an environment was tested when it was not built.
  Report the environments that were actually validated.

### 4.3 Build shell and environment

For MinGW packages, the canonical flow is to start a shell in the
target environment (for example UCRT64) and run `makepkg-mingw`.
`makepkg-mingw` defaults `MINGW_ARCH` to the active `MSYSTEM`.

For MSYS packages, run `makepkg` from the MSYS shell.

Do not use `makepkg` and `makepkg-mingw` interchangeably:

- `makepkg` builds MSYS packages.
- `makepkg-mingw` builds MinGW packages and sets up the MinGW
  environment.

`makepkg-mingw` may also be invoked from the MSYS shell when
`MINGW_ARCH` is set explicitly, because it re-executes `bash` once per
environment. This is acceptable, but the exact invocation must be
recorded. Never run plain `makepkg` for a MinGW package.

When automation runs outside MSYS2 (for example PowerShell or `cmd`),
invoke an MSYS2 Bash explicitly. Do not run `makepkg` directly from a
native Windows shell.

The `MSYSTEM` variable decides the `PATH` that the login profile sets.
A harness or parent process that exports `MSYSTEM=UCRT64` will hand that
value down to `bash -l`, and an MSYS package build would then resolve
`python`, `gcc`, etc. against the MinGW tree (for example
`/ucrt64/bin/python` instead of `/usr/bin/python`). When building MSYS
packages from automation, force the environment explicitly and verify it
before building:

```bash
export MSYSTEM=MSYS   # for msys packages
bash -lc 'echo "$MSYSTEM"; which python'   # must print /usr/bin/python
```

Do not build MinGW packages in that same shell; start a UCRT64 shell or
re-export `MSYSTEM=UCRT64` for them (see section 4.3 above and the
`MINGW_ARCH` note below).

For every build, verify and record:

- the shell/environment used;
- the target architecture/environment;
- the compiler and linker selected;
- the values of `MINGW_PREFIX`, `MINGW_PACKAGE_PREFIX`, `MINGW_CHOST`,
  `RUST_CHOST`, and `MINGW_ARCH`, when applicable;
- the exact build command.

### 4.4 Variables that matter in recipes

| Variable | Meaning |
| --- | --- |
| `MSYSTEM` | Active environment, e.g. `UCRT64`. |
| `MINGW_PREFIX` | Installation prefix, e.g. `/ucrt64`. |
| `MINGW_PACKAGE_PREFIX` | Package-name prefix, e.g. `mingw-w64-ucrt-x86_64`. |
| `MINGW_CHOST` | MinGW host triplet, e.g. `x86_64-w64-mingw32`. |
| `CHOST` / `CARCH` | Host triplet and architecture used by makepkg. |
| `RUST_CHOST` | Rust target triplet, e.g. `x86_64-pc-windows-gnu`. |
| `MINGW_ARCH` | Space-separated list of environments for `makepkg-mingw`. |
| `MSYS2_ARG_CONV_EXCL` | Arguments excluded from Unix-to-Windows path conversion. |
| `MSYS2_ENV_CONV_EXCL` | Environment variables excluded from path conversion. |

`CARCH` is `x86_64` in UCRT64 even though the package metadata uses
`arch=('any')`. Do not confuse `CARCH` with the PKGBUILD `arch` array.

## 5. PKGBUILD style and metadata

### 5.1 General style

A PKGBUILD is a Bash script. It MUST be valid Bash and follow MSYS2
conventions.

- Use **two spaces** for indentation.
- Keep lines reasonably short; target fewer than 100 characters where
  practical.
- Quote variables that may contain spaces, especially `"$srcdir"` and
  `"$pkgdir"`.
- Preserve the conventional PKGBUILD metadata/function structure.
- Avoid unnecessary custom variables and functions.
- When a custom variable or function is necessary, prefix its name with
  `_` to avoid collisions with makepkg internals.
- Do not use makepkg internal helper functions such as `msg`, `msg2`,
  `plain`, `warning`, or `error` in recipes. Use normal shell output
  such as `printf` or `echo`. Imported upstream recipes that already
  use them may keep them, but do not introduce them in new recipes.
- Do not add empty metadata arrays or redundant fields.
- Do not add a package to its own `provides` or `conflicts`.
- Do not use `/usr/local`.
- Keep package descriptions concise (about 80 characters or less) and
  do not repeat the package name unnecessarily. For MinGW packages the
  suffix `(mingw-w64)` is conventional and helps distinguish them.
- Use the package's actual upstream version for `pkgver`.
- Start `pkgrel` at `1` for a new upstream version and increment it for
  packaging-only changes to that version. Reset it when `pkgver`
  changes.
- Use `epoch` only when version comparison genuinely requires it.

### 5.2 Naming conventions

For MinGW packages, use the MSYS2 environment-independent naming
convention:

```bash
_realname=foo
pkgbase=mingw-w64-${_realname}
pkgname=("${MINGW_PACKAGE_PREFIX}-${_realname}")
```

For MSYS packages:

```bash
_realname=foo
pkgname=${_realname}
```

Rules:

- Do not hard-code `mingw-w64-ucrt-x86_64-` in `pkgname`,
  `depends`, `provides`, `conflicts`, or `replaces`; use
  `${MINGW_PACKAGE_PREFIX}` instead.
- Use `${MINGW_PREFIX}` for installation paths, not `/ucrt64`.
- Use `_realname` for upstream/source names and for license directory
  names.
- Package names may contain lowercase alphanumerics and `@._+-`; they
  must not start with `-` or `.`.
- Do not suffix names with the upstream major version unless upstream
  itself requires parallel installation.

### 5.3 `arch` and `mingw_arch`

**MinGW packages:**

Use `mingw_arch` to list the environments supported by the package.
The array below is an example, not a fixed repository-wide requirement;
tailor it to each package's validated targets.

```bash
arch=('any')
mingw_arch=('ucrt64' 'clang64' 'clangarm64')
```

MSYS2 encodes the target environment in the package name and uses
`arch=('any')` for MinGW packages, including compiled binaries. Do not
"correct" this to `arch=('x86_64')` based on Arch Linux rules.

**MSYS packages:**

```bash
arch=('x86_64')
```

The MSYS repository ships x86_64 packages only, and this repository
builds only x86_64. Current upstream MSYS2 master recipes still list
`arch=('i686' 'x86_64')`; imported recipes may keep that, but do not
claim `i686` in a recipe you do not actually build. Use `arch=('any')`
for MSYS packages that are genuinely architecture-independent (for
example pure-Python packages without compiled extensions).

### 5.4 Versioning details

- `pkgver` must not contain a hyphen. Replace an upstream hyphen with
  an underscore and substitute it back in source URLs when necessary.
- Do not use `pkgver` for a Git branch name or a moving reference.
- For VCS sources, use `pkgver()` and a pinned commit or tag; see
  section 7.5.
- Keep `pkgver` and `pkgrel` synchronized with the actual source and
  the packaging changes.

### 5.5 MSYS2 extra metadata

MSYS2 recognizes `mingw_*` and `msys2_*` metadata. Add it when it is
useful and verifiable:

```bash
msys2_repository_url='https://github.com/owner/project'
msys2_changelog_url='https://github.com/owner/project/releases'
msys2_documentation_url='https://project.example/docs'
msys2_issue_tracker_url='https://github.com/owner/project/issues'
msys2_pgp_keys_url='https://project.example/signing-keys'
msys2_ignore_vulnerabilities=('CVE-2026-0000')
msys2_references=(
  'purl: pkg:pypi/foo'
  'purl: pkg:cargo/foo@1.2.3'
  'archlinux: foo'
  'aur: foo'
  'anitya: 1234'
  'cpe: cpe:/a:vendor:product'
)
```

Rules:

- `mingw_arch` is an array listed in section 5.3.
- `msys2_references` maps keys to optional values with `key: value`
  syntax.
- Use `purl:` for Python and Cargo packages. The old `pypi:` key has
  been removed; use `purl: pkg:pypi/<normalized-name>`.
- Normalize Python distribution names according to the Python packaging
  name normalization rules.
- Use `internal` only when the package has no external reference.
- Do not invent IDs, URLs, or CPEs.

### 5.6 Package relations

- List all direct first-level runtime dependencies in `depends`.
- Use `makedepends` for build-only tools and libraries.
- Use `checkdepends` only for dependencies needed by `check()`.
- Use `optdepends` only for genuinely optional runtime features, with a
  short description after `: `.
- Do not duplicate `depends` in `makedepends`.
- Versioned `provides` should include the version when dependents may
  require it.
- Use `${MINGW_PACKAGE_PREFIX}` in `provides`, `conflicts`, and
  `replaces` for MinGW packages.
- Do not add `$pkgname` to `provides`; pacman adds it implicitly.
- Do not add `$pkgname` to `conflicts`; a package cannot conflict with
  itself.
- Use `replaces` when a package is renamed or merged, not as a
  substitute for `conflicts`.
- For split packages, use `pkgbase` and per-package functions; each
  package may override `pkgdesc`, `license`, `depends`, and related
  fields.

## 6. Licenses

### 6.1 `license` array

MSYS2 uses SPDX license expressions with the `spdx:` prefix:

```bash
license=('spdx:MIT')
license=('spdx:MIT OR Apache-2.0')
license=('spdx:GPL-2.0-or-later')
license=('spdx:LGPL-2.1-only AND MIT')
```

Rules:

- Values starting with `spdx:` are SPDX license expressions.
- Multiple `spdx:` values are treated as combined with `OR`.
- Do not mix `spdx:` values with non-SPDX values in the same array.
- Values not starting with `spdx:` follow Arch rules; use `custom:` or
  `LicenseRef-<id>` for custom licenses.
- Use `LicenseRef-<idstring>` only when the license is not in the SPDX
  list and cannot be expressed otherwise.
- These identifiers are not Arch's own license list names. Do not
  convert `spdx:` expressions back to bare Arch identifiers such as
  `GPL` or `BSD`.
- If the project advertises a single license, it is acceptable to use
  that advertised license even if some bundled files carry other
  permissive licenses.

### 6.2 License files

Include the upstream license text in the package.

For MinGW packages, install it under the package's **real name**,
without the environment prefix:

```bash
install -Dm644 LICENSE \
  "${pkgdir}${MINGW_PREFIX}/share/licenses/${_realname}/LICENSE"
```

Example: for `mingw-w64-ucrt-x86_64-python-msgspec`, the directory is
`/ucrt64/share/licenses/python-msgspec/`, not
`/ucrt64/share/licenses/mingw-w64-ucrt-x86_64-python-msgspec/`.

For MSYS packages, use:

```bash
install -Dm644 LICENSE "${pkgdir}/usr/share/licenses/${pkgname}/LICENSE"
```

Preserve all license texts and notices that are required for the
installed files, including bundled components. For split packages, each
subpackage may have its own license field and license directory.

## 7. Sources, integrity, and patches

### 7.1 Source selection

- Prefer canonical upstream HTTPS release archives or source
  repositories.
- Prefer stable release sources over moving branches.
- Do not use specific mirrors that may disappear.
- Avoid downloading prebuilt upstream binaries and repackaging them as
  a substitute for a native MSYS2 build. MSYS2 explicitly discourages
  binary-only repackaging for package contributions.
- If upstream source archives omit required assets, identify the
  missing files and obtain them from a verifiable upstream source.
- Make source filenames unique in `SRCDEST`. When upstream uses a
  generic filename, use the `filename::URL` form:

```bash
source=("${_realname}-${pkgver}.tar.gz::https://example.org/download/${pkgver}.tar.gz")
```

- Use `_realname` in the local filename, not `${pkgname}` with an
  environment prefix.

### 7.2 Checksums and signatures

- Use the strongest checksum provided by upstream. At minimum use
  SHA-256.
- Run `updpkgsums` after changing source files or source URLs, then
  inspect the resulting checksum changes.
- Never fabricate checksums or claim that a checksum was verified
  without actually verifying it.
- `SKIP` is acceptable for detached signature files (`.sig`, `.sign`,
  `.asc`) and VCS sources. Do not use `SKIP` for a stable source
  archive or a patch when a checksum can be obtained.
- Use upstream PGP signatures and `validpgpkeys` when upstream provides
  a verifiable signing process.
- `validpgpkeys` entries must be full uppercase fingerprints without
  whitespace.
- Verify signing keys through upstream's official channels. Remove
  obsolete keys from `validpgpkeys` when they are no longer valid or
  used.

### 7.3 `noextract`

Use `noextract` when an archive must be installed as-is or cannot be
handled correctly by `bsdtar`, or when the archive has too many
top-level files and would collide with other sources:

```bash
noextract=("${_realname}-${pkgver}.tar.gz")
```

When using `noextract`, extract manually in `prepare()` and add any
required extraction tool to `makedepends`.

A common reason for `noextract` is an upstream archive containing real
symlinks. On Windows systems where native symlink creation is not
available, libarchive/`bsdtar` fails with `Invalid argument` and the
whole extraction aborts. GNU `tar` from the MSYS environment writes the
symlink as a regular file copy, which is acceptable for sources. Record
the reason in a comment next to `noextract`.

### 7.4 Patch handling

- Apply patches in `prepare()`.
- List every patch file in `source` and provide its checksum.
- Use `git apply` for patch files.
- In a Git checkout, use `git cherry-pick --no-commit` to backport a
  commit.
- Do not create commits in the source checkout during package
  preparation.
- `patch -Np1 -i` is acceptable for imported MSYS2 recipes and for
  patches that are not Git-formatted, but state the reason if it
  differs from the default.
- Use descriptive patch names following the MSYS2 convention:
  `###-<package>-<version>-<Purpose>.patch`, where the sequence starts
  at `001`.
- Document why each non-obvious patch is required and link to the
  relevant upstream issue or commit when available.
- Prefer upstreaming generally useful fixes instead of maintaining
  long-lived local patches.

### 7.5 VCS sources

- Pin Git sources to a release tag or a commit, not an unqualified
  branch.
- Prefer a verified commit hash over a mutable tag reference.
- Add the appropriate VCS client to `makedepends` (`git`, `subversion`,
  etc.).
- Use `SKIP` for the VCS source checksum, which is legitimate in this
  case.
- Provide a `pkgver()` function when the package tracks a VCS source,
  and make sure it produces a pacman-compatible version.

## 8. Dependencies

Dependencies are a critical correctness requirement.

### 8.1 Direct dependencies

- Declare every direct runtime dependency required by the installed
  package.
- Use exact MSYS2 package names.
- For MinGW packages, prefix dependency names with
  `${MINGW_PACKAGE_PREFIX}`.
- For MSYS packages, use the plain MSYS package name.
- Do not rely on transitive dependencies to provide a library or
  executable directly required by the package.

### 8.2 Build and check dependencies

- Declare build-only tools and libraries in `makedepends`.
- Do not duplicate `depends` entries in `makedepends`.
- Use `checkdepends` for dependencies needed only by `check()`.
- `base-devel` is assumed to be installed, but it does not contain a C
  compiler in MSYS2. Add `gcc` for MSYS packages or
  `${MINGW_PACKAGE_PREFIX}-cc` for MinGW packages explicitly.
- Prefer the environment-independent meta packages where they exist:
  `${MINGW_PACKAGE_PREFIX}-cc`, `-cmake`, `-meson`, `-ninja`,
  `-pkgconf`, and similar.
- Do not put build tools in `depends` unless they are also required at
  runtime.
- Do not put runtime libraries only in `makedepends`.

### 8.3 Optional dependencies

- Use `optdepends` only for genuinely optional runtime features.
- Give each entry a short description after `: `.
- Do not use `optdepends` for build requirements.

### 8.4 Runtime DLLs and PE dependencies

- Inspect the produced executables and libraries to identify dynamic
  dependencies. On Windows, use suitable tools such as `objdump -p`,
  `ntldd`, or other PE dependency inspection tools. Do not assume Linux
  `ldd` output is meaningful for PE binaries.
- Verify that all required DLLs are supplied by declared runtime
  dependencies or are intentionally installed by the package.
- Add `${MINGW_PACKAGE_PREFIX}-gcc-libs` (or the specific providing
  package) only when the PE imports actually require it. Do not add it
  blindly, and do not omit it when `libgcc_s_*.dll`,
  `libstdc++-6.dll`, or `libwinpthread-1.dll` is imported.
- MSYS2's `autodeps` mechanism is disabled by default in
  `makepkg_mingw.conf`; do not assume that missing dependencies will be
  filled in automatically.
- Do not copy dependency DLLs into the package to conceal a missing or
  incorrect dependency declaration.
- Investigate static versus dynamic linking explicitly. Do not assume a
  Rust crate or a native library is statically linked merely because
  the application is written in Rust.

### 8.5 Automagic dependencies and offline builds

Dependencies should cover the requirements for building all packaged
features. If a feature's dependencies are not packaged, disable the
feature explicitly with a configure or build option. Do not allow
unrelated software on the build machine to enable features that are
not reflected in `depends`.

Keep the build self-contained. All build inputs must be declared and
available before `build()` runs:

- Fetch dependency downloads in `prepare()` (for example
  `cargo fetch --locked --target "${RUST_CHOST}"`), never during
  `build()`; the language-specific sections define the exact patterns.
- Do not resolve dependencies dynamically during `build()`: no package
  managers, no virtualenv provisioning, no network downloads.
- When technically possible, a build must succeed with no Internet
  access. Document any exception and its reason.
- This is especially important for Rust (`--frozen`) and Python
  (`--no-isolation` with build requirements in `makedepends`).

## 9. Build phases

Use the standard makepkg phases appropriately:

- `prepare()`: apply patches, extract archives listed in `noextract`,
  and perform source preparation. Do not compile here.
- `build()`: compile the software.
- `check()`: run upstream tests when practical and meaningful.
- `package()`: install only the files that belong in the package
  staging directory (`"$pkgdir"`).

Do not perform the actual compilation in `package()` unless the
upstream build system or a justified packaging constraint requires it.

Do not write files directly to the live system. All installed files
must be staged below `"$pkgdir"`.

Prefer upstream-supported build and install mechanisms. If upstream's
install target is unsuitable, explain why and install the required files
explicitly.

Do not include temporary build artifacts, source trees, caches,
credentials, private keys, local configuration, or unrelated files in
the final package.

Makepkg already handles stripping, compression of man/info pages, and
purging of common unwanted files. Do not strip or purge manually unless
there is a documented reason. Use `options=('!strip')` only when
stripping would break the package (for example some Python packages).

### 9.1 Windows path conversion and relocation

MSYS2 converts arguments and environment variables that look like Unix
paths when it invokes native Windows programs. This conversion is
usually helpful but can corrupt values that are not filesystem paths.

Use `MSYS2_ARG_CONV_EXCL` for arguments such as `--prefix=`,
`-DCMAKE_INSTALL_PREFIX=`, or `-DCMAKE_PREFIX_PATH=`:

```bash
MSYS2_ARG_CONV_EXCL="--prefix=;-DCMAKE_INSTALL_PREFIX=" \
  cmake -GNinja -DCMAKE_INSTALL_PREFIX="${MINGW_PREFIX}" ...
```

Use `MSYS2_ENV_CONV_EXCL` when a path-like environment variable must
keep its Unix form.

The prefix passed to native build systems is intentionally the Unix
path such as `/ucrt64`. It is also a valid Windows drive-relative path,
so native tools can concatenate it with `DESTDIR`. Do not replace it
with a hard-coded `C:/...` path.

For MinGW packages, avoid baking absolute Windows paths into installed
files. Use relocation patches and the MSYS2 `pathtools` helper where
needed. Correct `.pc`, `*-config`, and CMake files during `package()`
if the build system wrote a build-machine path into them. `cygpath -m`
may be used for sed replacements; never bake a `cygpath -w` result into
the package.

### 9.2 Install scripts and hooks

- Avoid `.install` scripts unless they are genuinely needed.
- Do not end an install script with `exit`; that prevents the remaining
  functions from running.
- Keep install scripts idempotent and safe during upgrade and removal.
- Prefer pacman hooks for file-triggered actions where appropriate.
- MSYS package install scripts may update files such as `/etc/shells`;
  do so defensively and remove only the lines the package added.

## 10. Language-specific rules

### 10.1 Python packages

Inspect `pyproject.toml`, `setup.cfg`, `setup.py`, the test configuration,
and any `setuptools-scm` usage.

Python packages exist for both subsystems. The interpreter is per
environment, so the package depends on the environment it was built for:

| Subsystem | Interpreter | Prefix | `depends` entry |
| --- | --- | --- | --- |
| MSYS | `/usr/bin/python` | `/usr` | `python` |
| UCRT64 | `${MINGW_PREFIX}/bin/python` | `${MINGW_PREFIX}` | `${MINGW_PACKAGE_PREFIX}-python` |
| CLANG64 | `${MINGW_PREFIX}/bin/python` | `${MINGW_PREFIX}` | `${MINGW_PACKAGE_PREFIX}-python` |
| CLANGARM64 | `${MINGW_PREFIX}/bin/python` | `${MINGW_PREFIX}` | `${MINGW_PACKAGE_PREFIX}-python` |

Do not assume that every Python package belongs in MSYS: pure-Python
packages are commonly built as MinGW packages for the native
environments, and packages with C extensions are environment-specific
and must be built against the interpreter they declare as a dependency.

For MinGW packages, use the MinGW Python interpreter and prefix:

```bash
options=('!strip')

makedepends=(
  "${MINGW_PACKAGE_PREFIX}-python-build"
  "${MINGW_PACKAGE_PREFIX}-python-installer"
  "${MINGW_PACKAGE_PREFIX}-python-setuptools"
  "${MINGW_PACKAGE_PREFIX}-python-wheel"
)

build() {
  cd "${_realname}-${pkgver}"
  "${MINGW_PREFIX}/bin/python" -m build --wheel --no-isolation
}

package() {
  cd "${_realname}-${pkgver}"
  MSYS2_ARG_CONV_EXCL="--prefix=" \
    "${MINGW_PREFIX}/bin/python" -m installer \
      --prefix="${MINGW_PREFIX}" \
      --destdir="${pkgdir}" \
      dist/*.whl

  install -Dm644 LICENSE \
    "${pkgdir}${MINGW_PREFIX}/share/licenses/python-${_realname}/LICENSE"
}
```

Rules:

- `--no-isolation` requires the build backend and all build
  requirements to be present in `makedepends`.
- `--skip-dependency-check` may be used when the Python dependency
  metadata is known to be incomplete but the runtime dependencies are
  correctly packaged. Document why.
- Do not use `pip install` in `package()`.
- Prefer building in a copy such as `python-build-${MSYSTEM}` if the
  build modifies the source tree.
- If `setuptools-scm` is used without a Git checkout, set
  `SETUPTOOLS_SCM_PRETEND_VERSION="${pkgver}"`.
- Run the test suite in `check()` when practical. A virtual environment
  with `--system-site-packages` is a common pattern:

```bash
check() {
  cd "${_realname}-${pkgver}"
  "${MINGW_PREFIX}/bin/python" -m venv --system-site-packages test-env
  test-env/bin/python -m installer dist/*.whl
  test-env/bin/python -m pytest
}
```

- MSYS2 packages include `.pyc` and `.opt-1.pyc` files when possible.
  Create them after installation with:

```bash
MSYS2_ARG_CONV_EXCL="-p" \
  python -m compileall -o 0 -o 1 -q -s"${pkgdir}" -p"/" \
    "${pkgdir}${MINGW_PREFIX}"
```

For MSYS Python packages, use `/usr/bin/python`, `/usr` paths, and
`/usr/share/licenses/${pkgname}`; do not use `MINGW_PREFIX`.

### 10.2 Rust and Cargo packages

For MinGW Rust packages:

1.  Inspect `Cargo.toml`, `Cargo.lock`, workspace members, `build.rs`
    files, enabled features, and native `*-sys` crates.
2.  Prefer the upstream release source archive or a pinned Git source
    containing the required lockfile and license files.
3.  Use the MSYS2 Rust toolchain (`${MINGW_PACKAGE_PREFIX}-rust`). Do
    not use `rustup`, and do not depend on a toolchain installed
    outside pacman.
4.  Preserve and use the upstream `Cargo.lock`.
5.  Use `--locked` / `--frozen` so Cargo does not silently resolve
    newer dependency versions. `--frozen` is equivalent to
    `--locked --offline`.
6.  Do not use `cargo update` as a routine packaging step. If the
    lockfile is missing or inconsistent, investigate and document the
    reason before changing dependency resolution.
7.  Fetch dependencies during `prepare()` so the build can run offline:

```bash
prepare() {
  cd "${_realname}-${pkgver}"
  cargo fetch --locked --target "${RUST_CHOST}"
}
```

8.  Build with the release profile and the MSYS2-configured target:

```bash
build() {
  cd "${_realname}-${pkgver}"
  cargo build --release --frozen
}
```

Use `--target "${RUST_CHOST}"` only when needed; in that case the
output is under `target/${RUST_CHOST}/release/`.

9.  Run tests without `--release` unless upstream requires release
    mode. Arch's guidance is that release-mode tests can hide overflow
    checks and debug assertions:

```bash
check() {
  cd "${_realname}-${pkgver}"
  cargo test --frozen
}
```

10. Install the executable with `install`, or use `cargo install` when
    the package needs Cargo's installation logic:

```bash
cargo install \
  --offline \
  --no-track \
  --frozen \
  --path . \
  --root "${pkgdir}${MINGW_PREFIX}"
```

11. Verify the actual Windows target triple and output path. Do not
    assume the executable is emitted without an `.exe` suffix.
12. Inspect `cargo tree` and native build scripts to identify system
    libraries and vendored dependencies.
13. Prefer MSYS2 system libraries over bundled copies when the crate
    supports it and the MSYS2 package is available.
14. For crates such as `libgit2-sys`, `libssh2-sys`, and
    `openssl-sys`, inspect their documented environment variables and
    feature flags before deciding whether to use system libraries or
    vendored builds.
15. Install only the required executable, documentation, completion
    files, licenses, and runtime assets.
16. Do not copy Arch's `glibc` / `libgcc` dependency conventions into
    MSYS2. Inspect PE imports instead.

### 10.3 CMake, Meson, Autotools

- Use `MSYS2_ARG_CONV_EXCL` for install-prefix arguments.
- Pass `--prefix="${MINGW_PREFIX}"` (Meson/Autotools) or
  `-DCMAKE_INSTALL_PREFIX="${MINGW_PREFIX}"` (CMake).
- Install with `DESTDIR="${pkgdir}"` or the build system's equivalent.
- Prefer Ninja with CMake in MSYS2.
- For CMake packages, use `-DCMAKE_BUILD_TYPE=Release` when debug is
  disabled, following the existing MSYS2 recipes.
- Do not hard-code `/usr`, `/usr/local`, or a Windows drive path in a
  MinGW package.

## 11. Installation layout and package contents

For MinGW packages, install files under the target prefix, normally
using `"${pkgdir}${MINGW_PREFIX}/..."`.

Examples:

- Executables: `${MINGW_PREFIX}/bin`
- Libraries and import libraries: `${MINGW_PREFIX}/lib`
- Headers: `${MINGW_PREFIX}/include`
- Shared data: `${MINGW_PREFIX}/share`
- Documentation: `${MINGW_PREFIX}/share/doc/${_realname}`
- Licenses: `${MINGW_PREFIX}/share/licenses/${_realname}`
- Configuration: `${MINGW_PREFIX}/etc`

For MSYS packages, use the MSYS prefix (`/usr`) and inspect existing
MSYS recipes.

Include upstream license files in the package. Preserve all applicable
upstream license texts and notices, including licenses of bundled
components when required.

Avoid splitting a package unless there is a clear benefit, such as large
optional components or independently useful subpackages. Follow MSYS2's
preference to minimize unnecessary package splitting. Do not create
`-dev` or `-devel` packages for MinGW software.

Do not include empty directories, build trees, `.la` files, or static
libraries unless the package is explicitly expected to provide them.

## 12. Validation and testing

A successful compilation alone does not prove that a package is correct.
Validation has two layers: the **package itself** and the **environment**
it runs in. A package that installs without errors can still be broken
by its environment (wrong interpreter, wrong `PATH`, missing DLLs), so
both layers must be checked before declaring a package complete.

### 12.1 Package validation

1.  Run `bash -n PKGBUILD`.
2.  Run `shellcheck --shell=bash --exclude=SC2034,SC2154,SC2164
    PKGBUILD` when ShellCheck is available.
3.  Review the PKGBUILD manually for quoting, path, metadata, and phase
    errors.
4.  Run `updpkgsums` when source definitions change and inspect the
    diff.
5.  Build with the correct MSYS2 packaging tool and target environment.
6.  Prefer a clean build with synchronized dependencies.
7.  Run the upstream test suite through `check()` when practical.
8.  Inspect the resulting package archive and its file list with
    `bsdtar -tf` and `pacman -Qlp`.
9.  Check package metadata and dependency declarations with
    `pacman -Qip`.
10. Inspect PE imports and runtime DLL dependencies for native Windows
    binaries with `objdump -p` or `ntldd`, and confirm that every
    imported DLL is provided by a declared dependency or installed by
    the package itself.

### 12.2 Environment validation

11. Before building, verify that the target environment is really
    active and that tool resolution points at the right tree: for MSYS
    packages `echo "$MSYSTEM"` must print `MSYS` and, when Python is
    involved, `which python` must resolve to `/usr/bin/python`; for
    MinGW packages `echo "$MSYSTEM"` must be the target environment
    (`UCRT64`, ...) and `which python` must resolve to
    `${MINGW_PREFIX}/bin/python`. A shell launched from automation can
    inherit the wrong `MSYSTEM` value (see section 4.3).
12. Install the generated package with `pacman -U` in a suitable test
    environment.
13. Run the installed executable and its basic functionality **from the
    target environment shell**, checking interpreter and `PATH`
    resolution, and that external tools the software shells out to
    (shells, editors, VCS) are found in the environment or declared as
    dependencies.
14. Verify that uninstalling the package does not leave unexpected
    files, for example with `pacman -Qkk`.
15. Test in a clean, updated MSYS2 installation, preferably not in the
    default `C:\msys64` location, to catch hard-coded paths and missing
    runtime dependencies.
16. Rebuild after packaging-only changes with the appropriate
    repackage option rather than unnecessarily repeating a long
    compilation.
17. Report tests that could not be run; never imply they passed.

Useful commands include:

```bash
bash -n PKGBUILD
shellcheck --shell=bash --exclude=SC2034,SC2154,SC2164 PKGBUILD
updpkgsums
makepkg-mingw --cleanbuild --syncdeps --force --noconfirm
makepkg-mingw --install --noconfirm
pacman -U ./<package-file>.pkg.tar.zst
pacman -Qip ./<package-file>.pkg.tar.zst
pacman -Qlp ./<package-file>.pkg.tar.zst
pacman -Qkk <package-name>
```

For a repackage without recompiling, use the appropriate repackage
option, for example `makepkg -RdLf` or `makepkg-mingw -RdLf`, after
confirming the option with the installed tool's `--help`.

To build several MinGW environments explicitly:

```bash
MINGW_ARCH='ucrt64 clang64 clangarm64' \
  makepkg-mingw --cleanbuild --syncdeps --force --noconfirm
```

Use `makepkg` instead of `makepkg-mingw` for an MSYS package. Check the
installed tool's `--help` and current MSYS2 documentation before relying
on specific options.

## 13. Arch Linux rules adopted as secondary guidance

Apply these Arch guidelines unless they conflict with MSYS2
requirements:

- Quote variables that may contain whitespace.
- Keep PKGBUILD logic simple and avoid unnecessary custom
  variables/functions.
- Prefix unavoidable custom variables/functions with `_`.
- Use correct and complete direct dependency declarations.
- Keep optional runtime dependencies in `optdepends`.
- Verify source integrity and use valid checksums/signatures.
- Keep package metadata accurate and descriptions concise.
- Include upstream license files.
- Use conventional package filesystem locations, adapted to the MSYS2
  prefix.
- Prefer working with upstream and contributing generally useful fixes
  upstream.
- Keep package version and release numbering consistent.
- Do not claim reproducibility unless the source, dependency
  resolution, toolchain, and build process support it.

Arch-specific paths, runtime dependencies, ELF inspection commands,
system services, hooks, and Linux-only assumptions MUST NOT be copied
into a MinGW PKGBUILD without proving that they apply to Windows and
MSYS2. In particular:

- Do not use `ldd` or `readelf` as the primary check for PE binaries.
- Do not add `glibc` or `libgcc` because an Arch recipe does.
- Do not use `arch=('any')` for MSYS packages merely because the
  software is portable.
- Do not use `/usr/share/licenses` in a MinGW package.

## 14. Agent behavior and reporting

When creating or changing a package, the agent MUST:

- Explain which existing recipes and official guidelines were used as
  references.
- State the selected subsystem and target environment.
- State the values of the relevant MSYS2 variables.
- Explain non-obvious dependency and linking decisions.
- Distinguish verified facts from assumptions.
- Avoid changing upstream source behavior merely to make packaging
  easier without explaining the change.
- Avoid adding compatibility hacks before reproducing and understanding
  the build failure.
- Avoid silently disabling tests, integrity checks, or compiler
  warnings.
- Never replace a source build with a prebuilt binary without explicit
  approval.
- Never claim a package is tested, installable, or correct unless the
  relevant commands were actually run successfully.
- Summarize the build command, test results, package output, verified
  environments, and known limitations.

If a build fails, report the first meaningful error, the environment and
toolchain used, and the next diagnostic step. Do not guess at a fix and
present it as established.

## 15. Quick checklist

Use this checklist before declaring a recipe complete:

- [ ] Recipe targets the MSYS subsystem and is directly under the
      repository root; native MinGW recipes belong in
      `MINGW-packages-extra`.
- [ ] Upstream source, version, and license verified.
- [ ] `_realname` / `pkgbase` / `pkgname` follow MSYS2 naming.
- [ ] `arch` is correct for MSYS; do not set `mingw_arch` for an MSYS
      recipe.
- [ ] `license` uses `spdx:` expressions.
- [ ] License files are installed under the real package name.
- [ ] Sources are canonical, unique, and checksummed; `SKIP` only where
      legitimate.
- [ ] Patches are in `source`, applied in `prepare()`, and documented.
- [ ] `depends`, `makedepends`, `checkdepends`, and `optdepends` are
      complete and non-duplicated.
- [ ] Runtime PE imports were inspected and their providers are
      declared.
- [ ] `MSYS2_ARG_CONV_EXCL` is used where native tools receive Unix
      paths.
- [ ] No makepkg helper functions (`msg`, `msg2`, `warning`, `error`)
      were introduced.
- [ ] `bash -n` and, when available, ShellCheck pass.
- [ ] `updpkgsums` was run after source changes.
- [ ] Clean build and tests were run in the declared environment.
- [ ] Package contents, metadata, PE imports, install, and removal were
      verified.
- [ ] The report states which environments were actually built and
      tested.

## 16. Official references

### MSYS2 --- primary

- Creating Packages: https://www.msys2.org/wiki/Creating-Packages/
- Creating a new Package: https://www.msys2.org/dev/new-package/
- Updating an existing Package: https://www.msys2.org/dev/update-package/
- Package Guidelines: https://www.msys2.org/dev/package-guidelines/
- License Metadata: https://www.msys2.org/dev/package-licensing/
- PKGBUILD extensions: https://www.msys2.org/dev/pkgbuild/
- Automated Build Process: https://www.msys2.org/dev/build-process/
- Environments: https://www.msys2.org/docs/environments/
- Package Naming: https://www.msys2.org/docs/package-naming/
- Filesystem Paths: https://www.msys2.org/docs/filesystem-paths/
- Porting: https://www.msys2.org/wiki/Porting/
- Official MSYS package repository:
  https://github.com/msys2/MSYS2-packages
- Official MinGW package repository:
  https://github.com/msys2/MINGW-packages
- MSYS2 package database: https://packages.msys2.org/

### MSYS2 templates

- CMake, Autotools, Meson, Python, Rust, and Cargo templates:
  https://github.com/msys2/MINGW-packages/tree/master/mingw-w64-PKGBUILD-templates

### Arch Linux --- secondary

- Arch package guidelines:
  https://wiki.archlinux.org/title/Arch_package_guidelines
- Creating packages:
  https://wiki.archlinux.org/title/Creating_packages
- PKGBUILD reference:
  https://wiki.archlinux.org/title/PKGBUILD
- Rust package guidelines:
  https://wiki.archlinux.org/title/Rust_package_guidelines
- PKGBUILD(5): https://man.archlinux.org/man/PKGBUILD.5
- makepkg(8): https://man.archlinux.org/man/makepkg.8

Re-check these references when making substantial changes to this
document or when packaging behavior appears to have changed. Upstream
documentation and repository conventions evolve; this file is a project
policy, not a substitute for checking the current official guidance.
