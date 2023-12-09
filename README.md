# dockhere

This utility makes is easier to work with remote Docker containers for building projects.

## Requirements

- `docker` command available (assumed to be linked to `podman`)
- Python 3+ with the dependencies from `requirements.txt` installed

## Installation

Simply place `dockhere` somewhere on your path after installing the dependencies and get started.

If you want to use a virtualenv for `dockhere`'s dependencies, you can add a script such as the following to your path:

```shell
#!/bin/bash
exec /path/to/venv/bin/python /path/to/dockhere "$@"
```

You will probably want to create a config file at `~/.dockhere.yaml` that has your default `volume` and `volumeMap`
configuration (see the "Volumes" section below).

## Config

`dockhere -h` will provide an overview of available command-line options, and `config-example.yaml`
has a full list of available configuration file options.

By default, `dockhere` will load `~/.dockhere.yaml` and `.dockhere.yaml` (in the current directory) and
merge their config together with the default config. It will respect CLI flags first, then the local config
file (or the file specified by `--config` on the command-line), then `~/.dockhere.yaml`, before falling
back to the defaults.

## Usage

Simply running `dockhere` in any directory will attempt to launch a container (by default, `ubuntu:latest`)
in interactive mode in the current working directory (if available).

### Remote Usage

This was designed with `podman-remote` in mind, as it is extremely useful for running Docker containers on
macOS using Apple Silicon. Using your virtualization method and software of choice (Apple Virtual with
UTM is the tested configuration), you should setup your `podman` host as you need. You can then follow
the [directions from podman][remote-tutorial] on setting up your remote client on the machine where you
will run `dockhere`. When running in a container with host-only or NAT connectivity, using TCP is often
easiest since it does not rely on SSH keys.

### Registries

By default, `dockhere` will lookup containers from `docker.io`. You can override this with the `registries`
and `defaultRegistry` configuration options, or by setting the `--registry`/`-r` flag on the CLI. If you specify
a registry on the command-line (or `defaultRegistry` in the config) that does not exist in `registries`, it is
treated as a URL to the registry.

### Volumes

`dockhere` is designed to launch containers in the CWD, and as a result the filesystem must be mounted
onto both the Docker host (if using `podman-remote`) and the container itself. It will perform a lookup of
the current working directory against the map `volumeMap` to try to find the first matching path. As a
result, if you have multiple overlapping maps, you should put the **most specific** one first. You should
also ensure that the volumes present in this map are also mounted to your container by adding them to the
`volumes` list as well. You should also always include trailing slashes in your `volumeMap` to avoid matching
on the start of name rather than on directory boundaries.

The `volumes` list supports all available command-line options (such as mounting a volume `ro`), as they are passed
to the underlying Docker client verbatim.

#### Example

Assuming:

- you are running with `podman` as a remote client
- your client is configured with the desired host as the default
- your host is running in an Apple Virtual VM locally
- your current directory is `/Users/youruser/src`
- you have shared the `/Users/youruser/src` directory with the VM
- you have mounted the shared filesystem to `/media/host`

```yaml
volumes:
  - /media/host:/host
volumeMap:
  /Users/youruser/src/: /host/src/
```

This will instruct `dockhere` to:

- mount the **host** path `/media/host` to the **container** path `/host`
- convert **client** paths starting with `/Users/youruser/src/` into **container** paths with `/host/src/`

In summary, `volumes` is a `host:container` mapping, whereas `volumeMap` is a `client:container` mapping - both are
essential to proper functioning.

[remote-tutorial]: https://github.com/containers/podman/blob/main/docs/tutorials/remote_client.md

### Multiarch

`podman` supports specifying a `--arch` CLI option, and so `dockhere` will automatically use this argument in all
cases.

#### Rosetta

If you are running on macOS on an Apple Silicon CPU, and your host VM is using Apple Virtualization, you can use
Rosetta to have native support for multiarch Docker containers. You will need to consult the documentation for your
virtualization client for information on how to enable it, but [UTM's documentation][utm-rosetta] is very detailed on
how to do this.

If you are not running on a platform that has a `update-binfmts` binary available, you can manually set it up:

1. Ensure that the `binfmt_misc` kernel module is loaded (add to your OS-specific module list, such as `/etc/modules`):
    ```shell
    modprobe binfmt_misc
    ```
2. Add the `binfmt_misc` and `rosetta` filesystem mounts to `/etc/fstab` (and mount with `mount -a`):
    ```shell
    # /etc/fstab
    binfmt_misc   /proc/sys/fs/binfmt_misc  binfmt_misc   rw,relatime   0 0
    rosetta       /mnt/rosetta              virtiofs      ro,nofail     0 0
    ```
3. Register Rosetta as a handler with `binfmt`:
    ```shell
    echo ":rosetta:M::\x7fELF\x02\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\x3e\x00:\xff\xff\xff\xff\xff\xfe\xfe\x00\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff\xff:/mnt/rosetta/rosetta:CF" > /proc/sys/fs/binfmt_misc/register
    ```
    Note: this needs to be done as root on boot. Add this to `/etc/rc.local`, `/etc/local.d/rosetta.start`, etc as
    appropriate for your OS.

Once the Rosetta binfmt handler is registered, `podman` should be able to run `amd64` arch containers by using the 
`--arch` flag.

[utm-rosetta]: https://docs.getutm.app/advanced/rosetta/#enabling-rosetta