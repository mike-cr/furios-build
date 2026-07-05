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
furios-shell
```

The current project is mounted at:

```text
/src/<project-name>
```

If `:Z` volume labeling causes trouble:

```sh
FURIOS_VOLUME_SUFFIX= furios-shell
```

## Build A Package

For an interactive shell, apt setup can be streamed in from the host when
needed:

```sh
furios-shell bash -s < "$(command -v furios-apt-setup)"
```

For a generic one-shot build from the host:

```sh
furios-build-cached --binary-only
```

Pass extra `dpkg-buildpackage` arguments after `--`:

Pass extra `dpkg-buildpackage` arguments after `--` when using
`furios-build-cached`.

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
FURIOS_APT_CACHE=http://host.containers.internal:3142 \
  furios-build-cached --binary-only
```

Or use the cached wrapper:

```sh
furios-build-cached --binary-only
```

Full build through the cache:

```sh
furios-build-cached
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
FURIOS_APT_CACHE=http://host.containers.internal:3142 \
  furios-build-cached --binary-only
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
FURIOS_PROJECT_DIR            Project directory to mount instead of cwd/git root
```

Container creation variables:

```text
FURIOS_ROOTFS                 Rootfs path, default: ~/containers/furios-forky-arm64
FURIOS_HOST_APT_CACHE         Host-side apt-cacher-ng URL, default: http://127.0.0.1:3142
FURIOS_DEBOOTSTRAP_MIRROR     Explicit debootstrap mirror
FURIOS_USE_APT_CACHE          Set to 0 to disable apt-cacher-ng during creation
```
