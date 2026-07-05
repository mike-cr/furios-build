# FuriOS Build Helpers

Small host-side helpers for building Debian packages in a FuriOS `forky`
`arm64` container.

The tools are intended for rootless Podman on a non-FuriOS host. They mount the
package source tree into a disposable FuriOS container, configure FuriOS apt
sources, install build dependencies, and run `dpkg-buildpackage`.

## Requirements

On the host:

```sh
podman
debootstrap
qemu-aarch64-static   # only needed on non-arm64 hosts
```

Rootless Podman must already work, and the kernel overlay module/runtime
directory permissions must be correct.

The default image name is:

```text
localhost/furios-forky-arm64:latest
```

## Create The Container Image

Create or recreate the FuriOS rootfs and import it into Podman:

```sh
bin/furios-create-container --force
```

Verify:

```sh
bin/furios-shell dpkg --print-architecture
```

Expected:

```text
arm64
```

Useful overrides:

```sh
FURIOS_ROOTFS=/mnt/bulk/furios-forky-arm64 bin/furios-create-container --force
FURIOS_IMAGE=localhost/furios-forky-arm64:test bin/furios-create-container
FURIOS_USE_APT_CACHE=0 bin/furios-create-container --force
```

If apt-cacher-ng is available at `http://127.0.0.1:3142`, the create script
uses its `HTTPS///` URL rewrite mode for bootstrapping and writes cached apt
sources into the image.

## Enter A Package Build Shell

From any Debian package source tree:

```sh
FURIOS_BUILD=/path/to/furios-build
"$FURIOS_BUILD/bin/furios-shell"
```

The current project is mounted at:

```text
/src/<project-name>
```

The helper repo is always mounted read-only at:

```text
/furios-build
```

If `:Z` volume labeling causes trouble:

```sh
FURIOS_VOLUME_SUFFIX= "$FURIOS_BUILD/bin/furios-shell"
```

## Build A Package

Inside the container:

```sh
/furios-build/bin/furios-build --binary-only
```

For a full source + binary build:

```sh
/furios-build/bin/furios-build
```

Pass extra `dpkg-buildpackage` arguments after `--`:

```sh
/furios-build/bin/furios-build -- --build=any
```

The build helper runs:

```text
apt setup
apt build-dep -y .
fakeroot debian/rules clean
dpkg-buildpackage -us -uc
lintian ../<package>_<version>_<arch>.changes
```

For full source builds, the package must already satisfy its Debian source
format requirements. For example, `3.0 (quilt)` packages often need an
`../<source>_<upstream>.orig.tar.*` file. Use `--binary-only` for quick test
builds when you do not need source artifacts.

## One-Shot Builds

From a package source tree on the host:

```sh
FURIOS_BUILD=/path/to/furios-build
FURIOS_APT_CACHE=http://host.containers.internal:3142 \
  "$FURIOS_BUILD/bin/furios-shell" \
  /furios-build/bin/furios-build --binary-only
```

Or use the cached wrapper:

```sh
"$FURIOS_BUILD/bin/furios-build-cached" --binary-only
```

Full build through the cache:

```sh
"$FURIOS_BUILD/bin/furios-build-cached"
```

## Apt Cache

If `FURIOS_APT_CACHE` is set, apt sources are rewritten to apt-cacher-ng
`HTTPS///` URLs:

```text
http://host.containers.internal:3142/HTTPS///debian.furios.io
```

This mode can cache HTTPS-origin package files. It is different from
`FURIOS_APT_PROXY`, which configures apt to use apt-cacher-ng as a CONNECT
proxy and usually does not cache package payloads.

Common cached build command:

```sh
FURIOS_BUILD=/path/to/furios-build
FURIOS_APT_CACHE=http://host.containers.internal:3142 \
  "$FURIOS_BUILD/bin/furios-shell" \
  /furios-build/bin/furios-build --binary-only
```

## Environment

Common variables:

```text
FURIOS_IMAGE                  Podman image tag
FURIOS_SUITE                  Suite name, default: forky
FURIOS_APT_CACHE              apt-cacher-ng HTTPS rewrite base URL
FURIOS_APT_PROXY              apt proxy URL
FURIOS_CONFIGURE_APT_TRUSTED  Set to 0 to keep existing apt trust/source config
FURIOS_VOLUME_SUFFIX          Project volume suffix, default: :Z
FURIOS_TOOLS_VOLUME_SUFFIX    Tools volume suffix, default: :Z,ro
FURIOS_PROJECT_DIR            Project directory to mount instead of cwd/git root
```

Container creation variables:

```text
FURIOS_ROOTFS                 Rootfs path, default: ~/containers/furios-forky-arm64
FURIOS_HOST_APT_CACHE         Host-side apt-cacher-ng URL, default: http://127.0.0.1:3142
FURIOS_DEBOOTSTRAP_MIRROR     Explicit debootstrap mirror
FURIOS_USE_APT_CACHE          Set to 0 to disable apt-cacher-ng during creation
```
