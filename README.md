# dcmtk

Container images with [DCMTK](https://dicom.offis.de/dcmtk), the DICOM toolkit from OFFIS. Its command-line tools dump, convert, compress, edit and render DICOM files, and send, receive and query them over the DICOM network protocol. DCMTK is compiled from its release tag on Ubuntu and Alpine, for `linux/amd64` and `linux/arm64`, and the images are rebuilt when DCMTK publishes a release and when the base image changes.

This is an unofficial build of the open source DCMTK, not affiliated with or endorsed by OFFIS. Report problems with the image in this repository. DCMTK bugs go to the DCMTK team at bugs@dcmtk.org, and questions to the [DCMTK forum](https://forum.dcmtk.org/).

## Quick start

Print the contents of a DICOM file:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/dcmtk dump image.dcm
```

The entrypoint is `dcmtk`, which runs the DCMTK tool named in its first argument. `dump` is a short name for `dcmdump`, and `dcmtk dcmdump image.dcm` works the same way. Run the image without arguments to list the tools and their short names. Every tool is also on `PATH`, so `--entrypoint dcmdump` works too. A few older single-purpose tools, such as `pdf2dcm`, `dcm2pdf`, `cda2dcm`, `stl2dcm`, `dcmgpdir` and `dcm2pnm`, are on `PATH` but unknown to `dcmtk`, so run them with `--entrypoint`.

Render an image as PNG, and turn a JPEG photo into a DICOM Secondary Capture file:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/dcmtk toimg +on image.dcm image.png
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/dcmtk img2dcm photo.jpg photo.dcm
```

## Tools

| Task | Tools (short names in parentheses) |
|---|---|
| Inspect and edit | `dcmdump` (`dump`), `dcmodify` (`modify`), `dcmftest`, `dsrdump` for structured reports, `drtdump` for radiotherapy objects |
| Convert | `dcmconv` (`convert`), `dcm2json` (`tojson`) and `json2dcm`, `dcm2xml` (`toxml`) and `xml2dcm`, `dump2dcm`, `dsr2html`, `dsr2xml`, `xml2dsr`, `dcmencap` and `dcmdecap` for encapsulated PDF, CDA and 3D model (STL, MTL, OBJ) files |
| Compress | `dcmcjpeg` and `dcmdjpeg` (JPEG), `dcmcjpls` and `dcmdjpls` (JPEG-LS), `dcmcrle` and `dcmdrle` (RLE); `dcmconv +td` writes deflated files |
| Images | `dcm2img` (`toimg`) writes PNG, TIFF, BMP, PNM, JPEG and JPEG-LS; `img2dcm` reads JPEG and BMP; `dcmscale`, `dcmquant`, `dcmicmp` |
| Network | `echoscu` (`echo`), `storescu`, `dcmsend` (`send`), `findscu` (`find`), `movescu` (`move`), `getscu` (`get`), `termscu`, `storescp`, `dcmrecv`, `dcmqrscp`, `wlmscpfs` |
| Media and signatures | `dcmmkdir` (`mkdir`) for DICOMDIR files, `dcmsign` (`sign`) |
| Presentation states and print | `dcmpsmk`, `dcmpschk`, `dcmp2pgm`, `dcmpsprt`, `dcmprscu`, `dcmprscp`, `dcmpsrcv`, `dcmpssnd`, `dcmmklut`, `dcmmkcrv` |

Each tool prints its options with `--help`. DCMTK has no JPEG 2000 or HTJ2K codec in its open source release, so files in those transfer syntaxes can be dumped and sent but not decompressed or rendered.

## Network

The examples use port 11112, which is registered for DICOM next to port 104. Unlike 104, it needs no privileges, also with `--network host`. To answer on port 104 of the host, publish the port with `-p 104:11112`.

Receive files on port 11112 and store them in `incoming/`:

```sh
mkdir -p incoming
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD/incoming:/work" -p 11112:11112 \
  ghcr.io/randomcontainers/dcmtk storescp 11112
```

`storescp` accepts every storage SOP class in the uncompressed transfer syntaxes; `+xa` adds the compressed ones, and `-aet` sets its AE title, which is `STORESCP` by default. It accepts files from anyone who can reach the port, and the connection is not encrypted. Publish the port as `-p 127.0.0.1:11112:11112` to accept only connections from the same host, or use the `+tls` options.

Check that a server answers, then send every DICOM file under `study/` to it:

```sh
docker run --rm ghcr.io/randomcontainers/dcmtk echo -aec PACS pacs.example.org 11112
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/dcmtk \
  send -aec PACS --scan-directories --recurse pacs.example.org 11112 study
```

The network tools are built with OpenSSL, so their TLS options (`+tls` and the others listed under `--help`) work.

## Data and configuration files

The tools load the DICOM data dictionary `dicom.dic` and `private.dic`, the dictionary of private tags that DCMTK knows, from `/usr/local/share/dcmtk-<version>`, which is also reachable as `/usr/local/share/dcmtk`. `DCMDICTPATH` replaces both with a colon-separated list of dictionary files.

Example configuration files for `storescp`, `storescu`, `dcmqrscp`, the presentation state tools and logging are in `/usr/local/etc/dcmtk/`, and the example worklist database for `wlmscpfs` and the dataset templates for `img2dcm` are in `/usr/local/share/dcmtk/`. DCMTK installs these into directories named after its version, `/usr/local/share/dcmtk-<version>` and `/usr/local/etc/dcmtk-<version>`; `/usr/local/share/dcmtk` and `/usr/local/etc/dcmtk` are links to them that keep their names across releases.

DCMTK converts character sets with its own converter, oficonv, whose tables are built into the library. Conversions such as `dcmconv +U8` therefore give the same result on Ubuntu and Alpine and need no iconv or ICU data.

## What is in the image

- The DCMTK command-line tools and the `dcmtk` command in `/usr/local/bin`, and DCMTK's shared libraries in `/usr/local/lib`.
- OpenSSL for TLS and `dcmsign`, libtiff and libpng for `dcm2img`, libxml2 for the XML tools, and zlib, all from the distro. `dcmsign`, `dcmp2pgm`, `dcmpsprt`, `dcmprscu` and `dcmprscp` also load OpenSSL's legacy provider, which the images include (`openssl-provider-legacy` on Ubuntu, part of `libcrypto3` on Alpine).
- The data dictionaries, configuration files and templates described above.

Not included: JPEG 2000 (DCMTK's OpenJPEG support is only used by its non-public JPEG 2000 module), TCP wrappers (libwrap), the man pages, the headers and CMake files for building against DCMTK, and the conversion table files that DCMTK installs in `csmapper` and `esdb`, with `mkcsmapper` and `mkesdb`, which generate them. The library has the tables built in and does not read those files. The CMake options and the output of `dcmdump --version` are in `/usr/local/share/randomcontainers/dcmtk/buildinfo`.

## Default or slim

DCMTK's default image adds no other tools, so `latest` and `slim` are the same image, with the contents listed above. Use `latest` to run it and the `slim` tags as a base for your own image.

## Tags

`<version>` is a DCMTK release such as `3.7.1`. `<minor>` and `<major>` are its shorter forms, `3.7` and `3`, and follow the newest release in that series. Each row lists the default tag and its `slim` twin, which point to the same image.

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

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current DCMTK version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

Dumping and sending only read files. The converters, `dcmodify` (which also keeps a `.bak` copy unless you pass `-nb`), `dcmmkdir` and the receivers write files, so run those with the flag above. `storescp` and `dcmrecv` store what they receive in their working directory unless `--output-directory` names another one.

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` move to each new DCMTK release and are rebuilt when the base image changes. For example, an image that also loads a dictionary of your own private tags:

```dockerfile
FROM ghcr.io/randomcontainers/dcmtk:slim-ubuntu@sha256:...
COPY site.dic /usr/local/share/site.dic
ENV DCMDICTPATH=/usr/local/share/dcmtk/dicom.dic:/usr/local/share/dcmtk/private.dic:/usr/local/share/site.dic
```

To install distro packages, switch to `USER root` and back to `USER 1000:1000` afterwards. The entrypoint is `["tini", "--", "dcmtk"]`; set your own `ENTRYPOINT` if your image runs something else, such as `storescp` with your options. To pick up new DCMTK releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

Everything the image adds is under `/usr/local`. `/usr/local/share/randomcontainers/dcmtk/` holds the version, the source URLs, the build options, the license files and `runtime-deps`, the list of distro packages DCMTK needs at run time.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/dcmtk:latest \
  --repo randomcontainers/dcmtk --signer-repo randomcontainers/ci
```

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/dcmtk:latest --format '{{ json .SBOM }}'
```

DCMTK is compiled here, so it is not in the SBOM and image scanners do not check it. Its version is in `/usr/local/share/randomcontainers/dcmtk/version` and the commit in `buildinfo` next to it.

OFFIS publishes DCMTK's release tarballs without checksums or signatures, and the tags are not signed, so each version is pinned to the commit of its tag. The build clones the `DCMTK-<version>` tag of [DCMTK/dcmtk](https://github.com/DCMTK/dcmtk), the GitHub mirror of DCMTK's own repository, and stops unless the tag points to the commit recorded in `package.yml`, so a tag that is moved after the version was picked up fails the build. The build also stops unless CMake found OpenSSL, libtiff, libpng, libxml2 and zlib, the tools report them in `--version`, and `dcmtk --help` finds every tool it lists.

## Updates

The project checks the tags of [DCMTK/dcmtk](https://github.com/DCMTK/dcmtk/tags) every 15 minutes. It follows tags because upstream has published some GitHub releases weeks after tagging them. A release is picked up once its tag is 24 hours old; snapshot tags such as `DCMTK-3.6.5+_20210308` are skipped. The new version and the commit of its tag are then committed to `package.yml` and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes and at least every 7 days, so distro security fixes reach the current tags.

DCMTK publishes about one release a year, and no separate bug-fix releases between them. A fix that lands in DCMTK's repository after a release reaches these images only with the next release tag; the images carry no patches of their own.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_COMMIT=<commit from package.yml> \
  -t dcmtk:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs.

## Licenses

DCMTK is licensed under the BSD 3-Clause license by OFFIS e.V. and the other authors named in its `COPYRIGHT` file. Code from other projects is compiled into it under other terms:

- BSD-2-Clause and BSD-3-Clause: oficonv, which is derived from the iconv code of FreeBSD by the Citrus Project, and smaller pieces such as Pascal Getreuer's color conversion code.
- ISC: oficonv's fallback `strlcpy`, which is only compiled on a C library that lacks one. glibc and musl both have `strlcpy`, so it is not in these images, but the label keeps ISC as a precaution.
- BSD-4-Clause: oficonv's Shift JIS converter, from Phase One, Inc., whose notice has an advertising clause.
- BSD-4.3TAHOE: the floating point formatting in `OFStandard::ftoa`, from the University of California.
- IJG: the JPEG codecs, from the Independent JPEG Group's library, version 6b.
- HPND-Pbmplus: the image scaling and color quantization code, from Jef Poskanzer's pbmplus and NetPBM.
- BSD-2-Clause and Apache-2.0: oflog, which is based on log4cplus.
- BSD-3-Clause: CharLS, the JPEG-LS codec.
- MIT: the jsmn JSON parser and the whereami code (offered under MIT or WTFPL; used here under MIT).
- `LicenseRef-DCMTK-or-AFPL`: the XML parser in `ofstd`, by Frank Vanden Berghen, which is under BSD-style terms for projects related to DCMTK and under the Aladdin Free Public License otherwise. These images are builds of DCMTK and use it under the BSD-style terms.
- `LicenseRef-OFFIS-RSNA-1993`: code from the 1993 RSNA DICOM demonstration, under two notices (OFFIS, Oldenburg University and CERIUM; RSNA and Washington University) that allow use, copying and derivative works as long as the notice is kept.
- `LicenseRef-Withers-base64`: Bob Withers' Base64 code, free for any use as long as his copyright notice is kept.
- `LicenseRef-Unicode-Mappings`: the character set conversion tables built into oficonv, most of which were generated from mapping files that Unicode, Inc. published with a notice allowing the mapping information to be used in products that support Unicode. `NOTICE.unicode` has both versions of that notice. The GB user-defined area, GBK extension and GB18030 tables carry the Citrus Project's BSD-2-Clause notice instead.

This software is based in part on the work of the Independent JPEG Group. This product includes software developed by Phase One, Inc., and software developed by the University of California, Berkeley.

The license files are in `/usr/local/share/randomcontainers/dcmtk/licenses/`: `COPYRIGHT`, which has DCMTK's license and the notices of most of the code above, `README.ijg`, `LICENSE.charls`, `LICENSE.log4cplus` (with the Apache License 2.0), `NOTICE.oficonv`, which collects the notices at the top of each oficonv source file, and `NOTICE.unicode`, the Unicode, Inc. notices of the mapping files the tables were generated from. The image's license label is `BSD-3-Clause AND BSD-2-Clause AND BSD-4-Clause AND BSD-4.3TAHOE AND Apache-2.0 AND IJG AND HPND-Pbmplus AND ISC AND MIT AND LicenseRef-DCMTK-or-AFPL AND LicenseRef-OFFIS-RSNA-1993 AND LicenseRef-Withers-base64 AND LicenseRef-Unicode-Mappings`. `/usr/local/share/randomcontainers/dcmtk/source` names the commit that was compiled. The build applies no patches. OpenSSL, libtiff, libpng, libxml2, zlib and the other Ubuntu and Alpine packages in the image keep their own licenses. The SBOM lists them.

DICOM is the registered trademark of the National Electrical Manufacturers Association for its standards publications relating to digital communications of medical information. The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
