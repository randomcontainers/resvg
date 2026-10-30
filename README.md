# resvg

Container images with [resvg](https://github.com/linebender/resvg), which renders SVG files to PNG, and usvg, which comes with it and writes a simplified copy of an SVG file with shapes and text turned into paths. resvg reads SVG and SVGZ files, draws text with the fonts it finds through the fontconfig configuration, and draws PNG, JPEG, GIF and WebP images embedded in or linked from the file. The images build both programs from the source of each resvg release with a pinned Rust release, and include the DejaVu fonts. They are built on Ubuntu and Alpine for `linux/amd64` and `linux/arm64`, and rebuilt when resvg publishes a release and when the base image changes.

This is an unofficial build, not affiliated with or endorsed by the resvg project. Report problems with the image in this repository and problems with resvg itself in the [resvg issue tracker](https://github.com/linebender/resvg/issues).

## Quick start

Render `logo.svg` in the current directory as a PNG 1200 pixels wide:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/resvg -w 1200 logo.svg logo.png
```

Without size options the PNG has the size that the SVG file gives, at 96 dpi. `-w` or `-h` sets the width or height in pixels and scales the other side to match, and `-z 2` doubles the size. `--background white` fills the transparent areas. `--export-id ID` draws only the element with that id, and `--query-all` prints each element that has an id with its bounding box, one per line, as `id,x,y,width,height`. Without arguments the image prints resvg's help.

`-` reads the SVG file from standard input and `-c` writes the PNG to standard output, so no mount is needed:

```sh
docker run --rm -i ghcr.io/randomcontainers/resvg - -c < drawing.svg > drawing.png
```

resvg then prints a warning about `--resources-dir`, the directory in which it looks for images that the file links by a relative path. It defaults to the directory of the input file, and only matters when the file links images.

## Text and fonts

resvg finds fonts through the fontconfig configuration in `/etc/fonts`, and the image has the DejaVu fonts: DejaVu Sans, DejaVu Serif and DejaVu Sans Mono. resvg does not take the generic font families from fontconfig. It maps `serif` to Times New Roman, `sans-serif` to Arial, `monospace` to Courier New, `cursive` to Comic Sans MS and `fantasy` to Impact, and text without a `font-family` is set in Times New Roman. None of these fonts is in the image. Text whose family has no font, and for which the serif family has none either, is left out of the PNG, and resvg prints a warning such as `No match for 'sans-serif' font-family.`

Name the DejaVu fonts for the generic families:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/resvg \
  --serif-family 'DejaVu Serif' --sans-serif-family 'DejaVu Sans' --monospace-family 'DejaVu Sans Mono' \
  card.svg card.png
```

resvg tries the serif family last for every family it cannot find, so with `--serif-family` set, text in Arial, Helvetica or any other font the image lacks is drawn in DejaVu Serif instead of being left out. `--font-family` sets the family for text without a `font-family`. Family names must match the name in the font file exactly; `--list-fonts` prints the generic families and every font file with the family names it holds. The DejaVu fonts have no Chinese, Japanese or Korean characters.

For other fonts, mount a directory of font files and pass `--use-fonts-dir`:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/resvg \
  --use-fonts-dir fonts --sans-serif-family 'Noto Sans' card.svg card.png
```

`--use-font-file` loads a single file, and `--skip-system-fonts` leaves out the fonts in the image. Fonts installed in an image built on `slim` are found through fontconfig (see [Extending the slim image](#extending-the-slim-image)).

## usvg

usvg reads an SVG file and writes it again in the reduced form that resvg draws from. Styles are resolved into attributes and shapes and text become paths, so the result no longer needs the fonts. The image runs resvg; pass `--entrypoint usvg` for usvg:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint usvg ghcr.io/randomcontainers/resvg \
  --sans-serif-family 'DejaVu Sans' in.svg out.svg
```

usvg takes the same font options as resvg, with the same defaults. `--preserve-text` keeps text as `<text>` elements, and `-` and `-c` read from standard input and write to standard output.

## Untrusted files

An SVG file can link images and other SVG files by path, relative to `--resources-dir` or absolute, and resvg draws them into the PNG. It does not fetch URLs. A file from someone else can therefore pull PNG, JPEG, GIF, WebP and SVG files that the container can read into the output. Large files, filters and large output sizes can also take a lot of memory and time. For files from unknown sources, mount only the directory the job needs and take away what the container does not need:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  --network none --read-only --cap-drop ALL --security-opt no-new-privileges \
  --memory 1g --cpus 1 --pids-limit 64 \
  ghcr.io/randomcontainers/resvg untrusted.svg untrusted.png
```

## What is in the image

- `resvg` and `usvg` in `/usr/local/bin`, built from the release tag with upstream's release profile and resvg's default features: SVGZ, text, system fonts, and PNG, JPEG, GIF and WebP images. They are linked against the distro's C library and libgcc_s: glibc on Ubuntu, musl on Alpine. The build also runs upstream's tests, which draw 1,730 SVG files and compare each with a reference PNG, on each platform the image is built for.
- The DejaVu fonts and fontconfig from the distro: `fonts-dejavu-core` and `fontconfig` on Ubuntu, and `font-dejavu`, which depends on `fontconfig`, on Alpine. resvg reads fontconfig's configuration files itself and does not load the fontconfig library.

Copying `resvg` or `usvg` from `/usr/local/bin` into another image with `COPY --from` works only on a base with the same C library, at the same or a newer version, and libgcc_s. resvg then draws text only with the fonts that image has.

`/usr/local/share/randomcontainers/resvg/buildinfo` has the commit, the Rust release with the URL and SHA-256 of each archive it was installed from, the output of `rustc -vV`, the cargo-auditable version, the versions resvg and usvg report, and every crate the binaries were built from, with its version, whether it is linked in or used only during the build, and its license.

## Default or slim

resvg's default image adds no other tools, so `latest` and `slim` are the same image, with the contents listed above. Use `latest` to run it and the `slim` tags as a base for your own image.

## Tags

`<version>` is a resvg release such as `0.48.1`. `<minor>` and `<major>` are its shorter forms, `0.48` and `0`, and follow the newest release in that series. Each row lists the default tag and its `slim` twin, which point to the same image.

| Tags | Base |
|---|---|
| `latest`, `slim` | Ubuntu |
| `<version>`, `<version>-slim` | Ubuntu |
| `<minor>`, `<minor>-slim`, `<major>`, `<major>-slim` | Ubuntu |
| `ubuntu`, `slim-ubuntu` | Ubuntu |
| `<version>-ubuntu`, `<version>-slim-ubuntu` | Ubuntu |
| `<minor>-ubuntu`, `<minor>-slim-ubuntu`, `<major>-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04`, `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `slim-alpine` | Alpine |
| `<version>-alpine`, `<version>-slim-alpine` | Alpine |
| `<minor>-alpine`, `<minor>-slim-alpine`, `<major>-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24`, `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current resvg version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

resvg has not reached 1.0, and a minor release can change its options, so the `0` tags move to releases that may break a script. Pin `<minor>` or the exact version in scripts and CI.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Each image contains resvg and usvg built for its own architecture and is built natively on GitHub-hosted runners, without emulation.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. resvg writes only the output file, so the image also runs with `--read-only`. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` move to each new resvg release and are rebuilt when the base image changes. Switch to root to install more, then back. This adds the Noto fonts, for scripts that DejaVu does not cover:

```dockerfile
FROM ghcr.io/randomcontainers/resvg:slim-ubuntu@sha256:...
USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends fonts-noto-core fonts-noto-cjk \
 && rm -rf /var/lib/apt/lists/*
USER 1000:1000
```

resvg finds fonts installed this way through the fontconfig configuration; pass their family names with the options under [Text and fonts](#text-and-fonts). On Alpine, start from `slim-alpine` and use `apk add --no-cache font-noto font-noto-cjk`. The entrypoint is `["tini", "--", "resvg"]` with `--help` as the default argument; set your own `ENTRYPOINT` if your image runs a script. To pick up new resvg releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

Everything the image adds is under `/usr/local`. `/usr/local/share/randomcontainers/resvg/` holds the version, the source URLs, the build details, the license files and `runtime-deps`, the list of distro packages resvg needs at run time: the fonts and fontconfig, and on Alpine also musl and libgcc_s by their `so:` names.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/resvg:latest \
  --repo randomcontainers/resvg --signer-repo randomcontainers/ci
```

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/resvg:latest --format '{{ json .SBOM }}'
```

resvg and usvg are built with [cargo-auditable](https://github.com/rust-secure-code/cargo-auditable), which stores the list of crates compiled into each of them in the binary's `.dep-v0` section. Syft, Grype and Trivy read that list, so SBOM tools and image scanners see those crates. The Rust standard library is not in the list, and scanners do not report its advisories; `buildinfo` names the Rust release that built resvg, and [Updates](#updates) says when it moves.

resvg publishes no checksums or signatures for its releases, so each version is pinned to the commit of its tag. The build clones the `v<version>` tag of [linebender/resvg](https://github.com/linebender/resvg) and stops unless the tag points to the commit recorded in `package.yml`. cargo downloads the crates in the tag's `Cargo.lock` and checks each against the checksum there, and the build that follows has no network access. resvg has no `.cargo/config.toml`, and the build stops if a release adds one, since its settings would apply to the build.

Rust comes from static.rust-lang.org. The release manifest named in `package.yml` is checked against the SHA-256 recorded there and against its signature, made with the Rust release key (fingerprint `108F 6620 5EAE B0AA A8DD 5E1C 85AB 96E6 FA1B E5FE`, kept in [keys/rust.gpg](keys/rust.gpg)). Each archive installed from it is checked against the SHA-256 that the manifest lists. cargo-auditable, which runs only during the build, is compiled from its crates.io crate, checked against the SHA-256 in `package.yml`, with the crates listed in its own `Cargo.lock`.

## Updates

The project checks the releases of [linebender/resvg](https://github.com/linebender/resvg/releases) every 15 minutes. A release is picked up once it is 24 hours old. The new version and the commit of its tag are then committed to `package.yml` and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built.

The Rust release that builds resvg follows Rust's stable channel. Each new stable release is committed to `package.yml` once it is a day old, and the images are rebuilt with it. cargo-auditable is pinned by hand. A release waits for a person when it is tagged without the `v`, as 0.45.0 was, when it fails upstream's tests on one of the platforms, when it adds a crate under a license that the image's label does not cover or a crate without a license file, when it moves siphasher to another version (see [Licenses](#licenses)), or when it adds a `.cargo/config.toml`.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes and at least every 7 days, so distro security fixes reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_COMMIT=<commit from package.yml> \
  --build-arg RUST_TOOLCHAIN_VERSION=<rust-toolchain version from package.yml> \
  --build-arg RUST_TOOLCHAIN_SHA256=<rust-toolchain sha256 from package.yml> \
  --build-arg CARGO_AUDITABLE_VERSION=<cargo-auditable version from package.yml> \
  --build-arg CARGO_AUDITABLE_SHA256=<cargo-auditable sha256 from package.yml> \
  -t resvg:local .
```

Use `Dockerfile.alpine`, with the same arguments, for the Alpine image. The build downloads 115 to 135 MB of Rust archives on Ubuntu and about 170 MB on Alpine, and the crates in `Cargo.lock`. `--build-arg JOBS=<n>` limits the number of crates cargo compiles in parallel.

## Licenses

resvg and usvg are available under the Apache License 2.0 or the MIT license, at your option (`Apache-2.0 OR MIT`). The binaries also contain the Rust crates resvg depends on. Most of them are under `MIT OR Apache-2.0` or MIT, and the others offer a choice of licenses that includes MIT, except arrayref, which is under BSD-2-Clause, tiny-skia and tiny-skia-path, which are under BSD-3-Clause, and slotmap, which is under Zlib. The Rust standard library and the crates it uses are linked in too. They are under `MIT OR Apache-2.0` or another choice that includes MIT, except compiler-builtins, which is under `MIT AND Apache-2.0 WITH LLVM-exception`, and the Unicode data tables in core, which are under Unicode-3.0.

The image's license label is `MIT AND BSD-2-Clause AND BSD-3-Clause AND Zlib AND Unicode-3.0 AND Apache-2.0 WITH LLVM-exception`, with MIT chosen wherever a crate offers it. The build stops when a crate linked into resvg or usvg has a license that this label does not cover. The crate list that cargo-auditable stores in the binaries names the bytemuck_derive proc macro and the crates it uses as runtime dependencies, because cargo-auditable takes a proc macro crate with test targets for a library. They are used only while the binaries are built, but the build checks them as linked crates, and the label covers them: unicode-ident among them includes Unicode data under Unicode-3.0.

The license files are in `/usr/local/share/randomcontainers/resvg/licenses/`: resvg's `LICENSE-MIT` and `LICENSE-APACHE` from the release tag, with the copies in the resvg and usvg crates in `workspace/`; `crates/<crate>-<version>/` with the license files of every crate that the binaries record, including those used only while they are built, and `crates/CRATES.tsv`, which lists each of them with its license; and `rust/` with the license files of rust-lang/rust, the standard library and the crates it vendors, from the `rust-src` archive of the Rust release that built resvg (all of them except LLVM's libunwind, which the dynamically linked binaries do not contain). The `COPYING` file of siphasher names license files that the crate does not include. This repository keeps those two texts in [licenses](licenses), and the build copies them into the image. The DejaVu fonts, fontconfig and the other Ubuntu and Alpine packages in the image keep their own licenses.

The source for each image:

- resvg and usvg: built from the `v<version>` tag of [linebender/resvg](https://github.com/linebender/resvg). `/usr/local/share/randomcontainers/resvg/source` names the commit of that tag, the Rust release manifest, the `rust-src` archive with the source of the Rust standard library, and the cargo-auditable crate. The crates are on [crates.io](https://crates.io/) at the versions in that commit's `Cargo.lock`, and `buildinfo` lists those resvg and usvg were built from.
- Build scripts: this repository at the commit in the image's `org.opencontainers.image.revision` label.
- Ubuntu packages: the source packages on [Launchpad](https://launchpad.net/ubuntu) for the versions listed in the SBOM. `apt-get source <package>=<version>` fetches a version that is still in the Ubuntu archive.
- Alpine packages: Alpine has no source packages. For the versions listed in the SBOM, the source is the APKBUILD and patches in [aports](https://gitlab.alpinelinux.org/alpine/aports/-/tree/3.24-stable), branch `3.24-stable`, and the archives on [distfiles.alpinelinux.org](https://distfiles.alpinelinux.org/distfiles/v3.24/).

The files in this repository are available under the MIT license, see [LICENSE](LICENSE), except the license texts in `licenses/`, which keep their own terms.

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
