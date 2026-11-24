# MSYS2-packages-extra

Additional package recipes for building software in the MSYS2 environment.
Each package has its own directory containing a `PKGBUILD` and any supporting
files.

## Building a package

From an MSYS2 shell with the required build dependencies installed:

```bash
cd <package-directory>
makepkg -sLf
```

Install the resulting package with `pacman -U`.

## License

The BSD-3-Clause license in [LICENSE](LICENSE) applies to original contributions
in this repository. Package recipes and other third-party material retain their
respective notices and licenses. The `license` field in each `PKGBUILD`
describes the packaged software, not the license of this repository.
