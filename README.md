# Chisel rock

This repository contains the [rock](https://documentation.ubuntu.com/server/explanation/virtualisation/about-rock-images/) definitions for [Chisel](https://github.com/canonical/chisel).

**Chisel** is a software tool for carving and cutting Debian packages! It can derive a minimal Ubuntu-like Linux distribution using a release database that defines "slices" of existing packages. Slices enable developers to cherry-pick just the files they need from the Ubuntu archives, and combine them to create a new filesystem which can be packaged into an OCI-compliant container image or similar.

## Chiselled image

This image was itself created with Chisel, to reduce its content to only the essentials. It carries the `chisel` binary, the directory skeleton from `base-files`, and the CA certificate bundle Chisel needs to reach the archives - no shell and no package manager. Details on the content can be found under the `base-files_*` and `ca-certificates_*` slices in the [chisel-releases](https://github.com/canonical/chisel-releases) repository.

## Available versions

* [chisel-1.4 (Ubuntu 26.04)](./chisel/1.4-26.04/rockcraft.yaml)

## Using Chisel

Chisel is invoked using `chisel <command>`, and relies on a [database of slices](https://github.com/canonical/chisel-releases) that are indexed per Ubuntu release. It installs them into the directory given by `--root`, which it does not create itself, so mount one:

```bash
mkdir rootfs
docker run --rm -v "$PWD/rootfs:/rootfs" chisel:1.4-26.04 \
  exec chisel cut --release ubuntu-24.04 --root /rootfs \
  libgcc-s1_libs libssl3t64_libs
```

In this example, Chisel would look into the Ubuntu Noble archives, fetch the provided packages and install only the desired slices into the *rootfs* folder, according to the slice definitions available in the ["ubuntu-24.04" chisel-releases branch](https://github.com/canonical/chisel-releases/tree/ubuntu-24.04).

The container entrypoint is [`pebble`](https://github.com/canonical/pebble), and this rock declares no services, so commands are run through `pebble exec` - that is what the `exec` above is.

Downloaded archive indices and packages are cached under `/var/cache/chisel`. Mount a volume there to reuse the cache between runs:

```bash
docker volume create chisel-cache
docker run --rm -v chisel-cache:/var/cache/chisel -v "$PWD/rootfs:/rootfs" \
  chisel:1.4-26.04 exec chisel cut --release ubuntu-24.04 --root /rootfs libssl3t64_libs
```

## Support for Pro archives

> [!IMPORTANT]
> To chisel a Pro package you need to have a Pro-enabled host.

Chisel picks the credentials up from `/etc/apt/auth.conf.d/`, which this image provides as an empty mount point:

```bash
docker run --rm -v /etc/apt/auth.conf.d:/etc/apt/auth.conf.d:ro \
  -v "$PWD/rootfs:/rootfs" chisel:1.4-26.04 \
  exec chisel cut --release ubuntu-24.04 --root /rootfs some-pro-package_slice
```

The default permissions of the credentials file on the host need to be changed so that Chisel can read it. The location can also be configured using the environment variable `CHISEL_AUTH_DIR`.

## Documentation

The [Chisel documentation](https://ubuntu.com/chisel/docs/latest/) provides further details on slicing a package, package slices, the command reference, and much more.

## Building and testing locally

Requires [Rockcraft](https://snapcraft.io/rockcraft) and LXD:

```bash
make check-setup

cd chisel/1.4-26.04
rockcraft pack
rockcraft test
```

There are two spread suites: `spread/general/` drives the rock under Docker, and `spread/k8s/` bootstraps a single-node k8s and runs a cut as a Job, which is what exercises cluster DNS and egress from inside the pod. `make test-all` runs both for every rock in this repository.

## Sync with template

This repository was created from [rocks-template](https://github.com/canonical/rocks-template). To pull in later template changes:

```bash
make sync-with-template
```

## CI configuration

Building, testing and publishing is driven by the workflows under `.github/workflows/` and configured in `.github/ci.yaml`. See the [rocks-template README](https://github.com/canonical/rocks-template/blob/main/README.md) for the full reference of that file.
