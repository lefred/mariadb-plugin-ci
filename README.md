# mariadb-plugin-ci

Reusable plugin releases from prepared MariaDB source/CMake SDK images. Release
jobs never clone MariaDB or configure it from scratch, and never request a server
or global build. Only the selected plugin target and its CMake dependencies build.

## Architecture

1. Run `prepare-sdk.yml` once per exact MariaDB release. It builds and publishes an
   image derived from `quay.io/mariadb-foundation/bb-worker:almalinux8-bintar`.
2. The image contains the exact `mariadb-X.Y.Z` tag and shallow recursive submodule snapshots at
   `/home/buildbot/mariadb-server`, its configured `build/` directory, and
   `/home/buildbot/bin/build-plugin` and `package-plugin`. No server compilation
   occurs during image preparation.
3. A plugin calls `build-plugin.yml`. A validation job expands `mariadb_versions`
   into a matrix. Each job uses the matching SDK, checks out the triggering plugin
   commit, adds it to the source tree, reruns CMake, and builds one target.
4. Packaging produces a tarball and SHA256 file. Matrix artifacts are collected,
   verified, and published to a GitHub Release for `v*` tags.

There is no implicit fallback when an image is missing: prepare/publish it first.
The default registry repository is `quay.io/lefred/mariadb-plugin-build`; this is a
configurable publication destination, not a claim that images already exist.

## Prepare images

Set repository Actions secrets `SDK_REGISTRY_USER` and `SDK_REGISTRY_PASSWORD`
for a registry account with push access, then dispatch `.github/workflows/prepare-sdk.yml`
with `mariadb_version` and `image_repository`. Public images are expected by the
release workflow. The summary reports the published digest.

Exact releases are mandatory; `11.8`, `11.8.x`, and `main` are rejected. The examples
below are requested version selections, not an automatically maintained list of
available releases. Each must exist as a MariaDB release tag and have an SDK image.

For local Podman use, from this repository:

```bash
podman build -f sdk/Containerfile \
  --build-arg MARIADB_VERSION=13.0.2 \
  --build-arg INCLUDE_GALERA=true \
  -t localhost/mariadb-plugin-build:13.0.2 .
```

Preparation exports `CMAKE_LIBRARY_PATH=/scripts/local/lib/` and
`WSREP_PROVIDER=/usr/lib64/galera-4/libgalera_smm.so`, then runs exactly:

```bash
cmake .. -DBUILD_CONFIG=mysql_release -DWITH_SYSTEMD=yes \
  -DCMAKE_BUILD_TYPE=RelWithDebInfo -DENABLE_DTRACE=ON
```

`INCLUDE_GALERA=true` checks the three Galera files, creates `cmake/galera.cmake`,
and inserts its include immediately after the specified cross-compilation block.
It fails if the release's block has changed. False leaves the upstream source
unchanged; this option controls Galera installation rules, not upstream WSREP support.
Use different image repositories for Galera/non-Galera variants to avoid tag collisions.

Common plugin dependencies (`clamav-devel libcurl-devel libarchive-devel
openssl-devel libargon2-devel`) are baked in by default. Override `EXTRA_PACKAGES` when building
an image; it replaces that list. For deterministic releases, pin `BASE_IMAGE` to
a digest, pin RPM versions or use a repository snapshot, retain the resulting
SDK digest, and use `sdk_images` below. Release tags and mutable image tags alone
do not guarantee bit-for-bit reproducibility.

## Viruscan caller

Copy [examples/viruscan.yml](examples/viruscan.yml) into the plugin's
`.github/workflows/release.yml`:

```yaml
name: Release Viruscan
on:
  push:
    tags: ['v*']
  workflow_dispatch:
permissions:
  contents: write
jobs:
  build:
    uses: lefred/mariadb-plugin-ci/.github/workflows/build-plugin.yml@main
    with:
      plugin_name: viruscan
      plugin_directory: viruscan
      build_target: viruscan
      plugin_library: viruscan.so
      mariadb_versions: |
        11.8.5
        12.3.3
        13.0.2
        13.1.1
      cmake_options: -DPLUGIN_VIRUSCAN=DYNAMIC -DWITH_SSL=system
      run_tests: false
```

Viruscan uses `-DWITH_SSL=system` so MariaDB's bundled wolfSSL compatibility
headers do not shadow the OpenSSL headers required by ClamAV. This overrides the
SSL choice when reconfiguring the plugin container; the prepared SDK still uses
the exact initial configuration above.

This builds the same triggering plugin commit for all four releases. For a single
version use `mariadb_versions: '13.0.2'`. A JSON string array is also supported.
Reusable workflow inputs are strings, not native YAML lists.

Use a workflow commit SHA instead of `@main` for pinned production callers. Override
`sdk_image` for your registry; tags are derived from the version. For immutable SDKs:

```yaml
      sdk_images: >-
        {"13.0.2":"quay.io/your-org/mariadb-plugin-build@sha256:REPLACE_WITH_DIGEST"}
```

Each map entry overrides the full image reference for that release. Unmapped
versions use `sdk_image:version`. The image's recorded release must match the matrix.

## Foundry builder (preview)

`builder: foundry` packages the plugin with [MariaDB Foundry](https://github.com/MariaDB/foundry)
instead of building it inside a configured server tree. The plugin is built
out-of-tree against an installed MariaDB Development component and packaged in
Foundry's format, for example:

```text
mariadb-plugin-viruscan-11.4.13-1.0-linux-x86_64.tar.gz   (lib/plugin/viruscan.so, .yml metadata, tests)
MariaDB-plugin-viruscan-11.4.13_1.0-1.el8.x86_64.rpm      (Requires: MariaDB-server = 11.4.13)
```

Each package gets a `.sha256` file, and tagged builds publish them to the GitHub Release.
The version in the filename comes from the plugin's CMake `VERSION`; `package_version`
does not apply. The SDK contains no server, so `run_tests` must be `false`.

Out-of-tree builds need `mariadb-plugin-config.cmake` (MDEV-40608). At the time of
writing it exists only on the MariaDB 11.4 branch (commit `3977a282`) and in no
release. Prepare a Foundry SDK from a ref that contains it; the image tag is the
server version that ref reports:

```bash
podman build -f sdk/Containerfile.foundry \
  --build-arg MARIADB_REF=3977a282b74c780aedef2390828a7e611a751cd5 \
  --build-arg FOUNDRY_REF=77a6e4cc861740cc40c781509eb893d86b757d41 \
  -t localhost/mariadb-plugin-foundry:11.4.13 .
```

or dispatch `prepare-foundry-sdk.yml` with the same refs. Use full commit SHAs
(short SHAs cannot be fetched) or tags such as `mariadb-13.0.3` once releases
include MDEV-40608. Only the Development component is compiled and installed into
`/home/buildbot/mariadb-devel` (about 30 seconds); the source tree is removed. A
branch build reports an unreleased version, so a later rebuild from the release
tag must not reuse its image tag without care.

See [examples/viruscan-foundry.yml](examples/viruscan-foundry.yml) for a caller. The
plugin's top-level `CMakeLists.txt` must support standalone builds. The pattern
below keeps in-tree builds working on older servers:

```cmake
IF(CMAKE_SOURCE_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR)
  CMAKE_MINIMUM_REQUIRED(VERSION 3.12)
  FIND_PACKAGE(mariadb-plugin REQUIRED)
ENDIF()

# Older MYSQL_ADD_PLUGIN misparses AUTHOR/DESCRIPTION; only the new macro accepts them.
IF(COMMAND MARIADB_ADD_PLUGIN)
  SET(METADATA AUTHOR "you" DESCRIPTION "What it does" VERSION 1.0)
  IF(EXTERNAL_PLUGIN)
    LIST(APPEND METADATA DEPENDS mariadb_private)  # empty DEPENDS fails out-of-tree
  ENDIF()
ENDIF()

MYSQL_ADD_PLUGIN(myplugin myplugin.cc MODULE_ONLY ${METADATA})

IF(EXTERNAL_PLUGIN)
  INCLUDE(CPack)  # Foundry builds the "package" target
ENDIF()
```

The `DEPENDS` and `CPack` lines work around gaps in the current upstream support. Foundry's own
`tidesql` recipe fails on the same `ADD_DEPENDENCIES` error. For a local check, run
the helper in the image with the plugin checkout mounted:

```bash
podman run --rm --user root -v "$PWD:/src:ro,z" localhost/mariadb-plugin-foundry:11.4.13 \
  bash -c 'cd /tmp && PLUGIN_SOURCE=/src PLUGIN_NAME=vmstat FOUNDRY_FORMATS="tgz rpm" foundry-build-plugin'
```

## Crypto EXTRA dependencies

Crypto EXTRA requires Argon2 development files and system OpenSSL. For SDK images
published before Argon2 was added to the defaults, use these caller inputs:

```yaml
      plugin_name: crypto_extra
      plugin_directory: crypto_extra
      build_target: crypto_extra
      plugin_library: crypto_extra.so
      cmake_options: -DPLUGIN_CRYPTO_EXTRA=DYNAMIC -DWITH_SSL=system
      extra_packages: libargon2-devel
      run_tests: false
```

Keep your existing `sdk_image` and `mariadb_versions` inputs. The package installs
before plugin configuration. Once both SDK images have been rebuilt and published
with the new default dependencies, `extra_packages` can be removed. Changing the
repository's defaults does not modify already-published images.

## Local plugin build and package (MariaDB 13.0.2)

After preparing the image above:

```bash
mkdir -p dist
podman run --rm --user root \
  -v "$PWD/dist:/out/dist:Z" \
  -e 'CMAKE_OPTIONS=-DPLUGIN_VIRUSCAN=DYNAMIC -DWITH_SSL=system' \
  localhost/mariadb-plugin-build:13.0.2 bash -c '
    set -euo pipefail
    git config --global --add safe.directory /home/buildbot/mariadb-server
    build-plugin https://github.com/lefred/mariadb-plugin-viruscan.git viruscan
    export GITHUB_WORKSPACE=/out
    export PLUGIN_SOURCE=/home/buildbot/mariadb-server/plugin/mariadb-plugin-viruscan
    export PLUGIN_NAME=viruscan PLUGIN_LIBRARY=viruscan.so MARIADB_VERSION=13.0.2
    export PACKAGE_VERSION=$(git -C "$PLUGIN_SOURCE" rev-parse --short=12 HEAD)
    export PLUGIN_PATH=/home/buildbot/mariadb-server/build/plugin/mariadb-plugin-viruscan/viruscan.so
    package-plugin
  '
```

Checksums contain archive basenames; verify from their directory:

```bash
(cd dist && sha256sum --check *.sha256)
```

The helper prints the actual library path; use it if a plugin changes its output
location. To persist an interactive environment, omit `--rm`, give the container a
name, and use `podman start -ai NAME` later. Do not reuse one mutable container for
concurrent builds or mount a different source/build tree over the prepared SDK.

Direct helper examples inside an SDK:

```bash
CMAKE_OPTIONS="-DWITH_SSL=system" build-plugin https://github.com/lefred/mariadb-plugin-viruscan.git
build-plugin https://github.com/lefred/mariadb-plugin-banquise-agent banquise_agent
```

Existing clones update with `git pull --ff-only`, followed by submodule updates.
Local URL builds follow the repository's default branch. For a reproducible local
plugin revision, check out the desired commit separately and set `PLUGIN_SOURCE`
to its absolute path; the helper links that checkout without updating it. CI always
uses this mode with the triggering commit.

Automatic target detection reads the top-level `MYSQL_ADD_PLUGIN` declaration,
lowercases the name, and verifies it against CMake target help. Multiple or computed
names require an explicit target. `CMAKE_OPTIONS` uses shell-like quoting without
shell evaluation. `PLUGIN_DIRECTORY` overrides the repository directory name;
`PLUGIN_LIBRARY` overrides the expected target.so output. A missing or ambiguous
library is an error.

## Packages and compatibility

Packages retain the existing structure:

```text
mariadb-plugin-viruscan-13.0.2-1.0.0-linux-x86_64/
├── lib/mariadb/plugin/viruscan.so
├── share/doc/viruscan/README.md
├── share/doc/viruscan/LICENSE
└── INSTALL.txt
```

README/README.md/README.txt and LICENSE/LICENSE.md/COPYING are included when present
in the plugin repository. A missing license is not invented. Runtime libraries
such as ClamAV are not bundled: install matching dependencies on the destination.
Artifacts target Linux x86_64 and the AlmaLinux 8 worker ABI.

Existing `plugin_name`, `plugin_directory`, `build_target`, `plugin_library`,
`cmake_options`, `test_suite`, `run_tests`, `package_version`, and
`artifact_retention_days` remain available. Directory and target are now optional;
library remains explicit. Two migration changes are necessary:

- Supply `mariadb_versions` with exact releases and prepare their images. The old
  implicit `main`/13.1 build is removed.
- `run_tests` retains its old true default, but requires `test_runtime`, an absolute
  path in a custom SDK to a matching prebuilt MariaDB binary distribution containing
  `mysql-test/mariadb-test-run.pl` and its runtime tools. Alternatively explicitly
  set `run_tests: false`. No server test infrastructure is compiled by this workflow.

MTR links the plugin suite from `mysql-test/<test_suite>` into the runtime and uses
the newly built library. The runtime must include everything that suite needs
(including auxiliary plugins/data). Provide that runtime in a derived image and
select it through `sdk_images`; automatic runtime downloading is not implemented.
Existing suites depending on the old build-tree layout may need adaptation.

`extra_packages` is an optional whitespace-separated list of RPM specifications
installed before plugin configuration. Prefer baking them into a pinned SDK to
avoid downloads and repository drift on every release.

## Validation and limits

```bash
python3 -m unittest discover -s tests -v
bash -n sdk/prepare sdk/prepare-foundry scripts/package-plugin
actionlint
```

Tests use a real CMake module and a deliberately failing default/server target to
check that only the plugin target is built, exercise target detection failures,
and verify archive contents and SHA256. SDK configuration and real plugin builds
also need validating against each selected upstream release. CMake may build
small generated-header/tool dependencies of a plugin; the helper does not request
`all`, `minbuild`, `mysqld`, or `mariadbd`. Plugins must define a standalone module
target rather than make their target depend on a complete server build.

Validated locally with MariaDB 13.0.2: SDK preparation with Galera enabled,
VMSTAT automatic target detection in the existing configured worker image, and a
cold Viruscan build/package/checksum using the new SDK with `WITH_SSL=system`.
The cold build checked that neither `build/sql/mariadbd` nor `build/sql/mysqld`
existed before or after. The Viruscan checkout tested was `4ac64a1eebe0`; it has a
README but no LICENSE file. Other releases, hosted Actions execution, registry
publication, and prebuilt-runtime MTR tests still need validation.
