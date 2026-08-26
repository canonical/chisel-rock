# Chisel rock

This repository contains the [rock](https://documentation.ubuntu.com/server/explanation/virtualisation/about-rock-images/) definitions for [Chisel](https://github.com/canonical/chisel), a software tool for carving and cutting Debian packages.

Chisel derives a minimal Ubuntu-like Linux distribution using a release database that defines "slices" of existing packages. Slices let you cherry-pick just the files you need from the Ubuntu archives and combine them into a new filesystem, which can then be packaged into an OCI-compliant container image or similar.

## Chiselled image

The rock is built with `bare` as its base and contains only the `chisel` binary, the directory skeleton from `base-files`, and the CA certificate bundle Chisel needs to reach the [slice definitions](https://github.com/canonical/chisel-releases) and the Ubuntu archives. There is no shell and no package manager in the image.

## Available versions

* [chisel-1.4 (Ubuntu 26.04)](./chisel/1.4-26.04/rockcraft.yaml)

## Example

Chisel writes a root filesystem to whatever `--root` points at, and it does not create that directory itself, so the useful thing to do is to mount one:

```bash
mkdir rootfs
docker run --rm -v "$PWD/rootfs:/rootfs" chisel:1.4-26.04 \
  exec chisel cut --release ubuntu-24.04 --root /rootfs \
  base-files_base ca-certificates_data
```

The container entrypoint is [`pebble`](https://github.com/canonical/pebble), and this rock declares no services, so commands are run through `pebble exec` - that is what the `exec` in the command above is.

Downloaded archive indices and `.deb` files are cached under `/var/cache/chisel`. Mount a volume there to reuse the cache between runs:

```bash
docker volume create chisel-cache
docker run --rm -v chisel-cache:/var/cache/chisel -v "$PWD/rootfs:/rootfs" \
  chisel:1.4-26.04 exec chisel cut --release ubuntu-24.04 --root /rootfs base-files_base
```

To cut from a Pro archive, mount your credentials at `/etc/apt/auth.conf.d`:

```bash
docker run --rm -v /etc/apt/auth.conf.d:/etc/apt/auth.conf.d:ro \
  -v "$PWD/rootfs:/rootfs" chisel:1.4-26.04 \
  exec chisel cut --release ubuntu-24.04 --root /rootfs some-pro-package_slice
```

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
