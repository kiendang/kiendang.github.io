---
layout: post
title:  "Run Docker containers leveraging NVIDIA GPUs without root privilege"
date:   2019-02-14 18:32:04 +0800
categories: jekyll update
---

Docker containers can be run without root privilege using [usernetes](https://github.com/rootless-containers/usernetes). This will be available in upstream docker soon™ following [moby/moby#38050](https://github.com/moby/moby/pull/38050). Use this guide for now.

*NOTE: In order to run this, user must have a range of [subuid(5)](http://man7.org/linux/man-pages/man5/subuid.5.html)s and [subgid(5)](http://man7.org/linux/man-pages/man5/subgid.5.html)s available to them, i.e they must be present in `/etc/subuid` and `/etc/subgid`. subuid and subgid range can be added by editting `/etc/subuid` and `/etc/subgid` directly or by running `sudo usermod --add-subuids <from>-<to> --add-subgids <from>-<to> <user>`, e.g `sudo usermod --add-subuids 65536-100000 --add-subgids 65536-100000 user`*

- [Rootless containers using usernetes](#running-rootless-containers-using-usernetes)
- [Rootless containers leveraging NVIDIA GPUs](#rootless-containers-leveraging-nvidia-gpus)
  - [Get NVIDIA Container Toolkit](#get-nvidia-container-toolkit)
  - [Configure NVIDIA Container Toolkit for rootless containers](#configure-nvidia-container-toolkit-for-rootless-containers)
  - [Run rootless containers leveraging NVIDIA GPUs](#run-rootless-containers-leveraging-nvidia-gpus)

## Running rootless containers using usernetes

```sh
# grab a build from https://github.com/rootless-containers/usernetes/releases
wget https://github.com/rootless-containers/usernetes/releases/download/v20190603.1/usernetes-x86_64.tbz
tar xjvf usernetes-x86_64.tbz
cd usernetes
```

Run dockerd server

```sh
./run.sh default-docker-nokube
```

You can now run rootless containers.

If you already have upstream docker installed system-wide

```sh
# docker -H unix://$XDG_RUNTIME_DIR/docker.sock <cmd>
docker -H unix://$XDG_RUNTIME_DIR/docker.sock run --rm -it busybox ls
```

or

```sh
export DOCKER_HOST="unix://$XDG_RUNTIME_DIR/docker.sock"
docker run --rm -it busybox ls
```

If you don't

```sh
# ./dockercli.sh <cmd>
./dockercli.sh run --rm -it busybox ls
```

## Rootless containers leveraging NVIDIA GPUs

You will need to disable `cgroup` in `nvidia-container-runtime` since `cgroup` is not yet supported in docker rootless mode.

### Get NVIDIA Container Toolkit

If you already have `nvidia-container-toolkit` installed, continue to [next step](#configure-nvidia-container-toolkit-for-rootless-containers).

If not, you need to get `nvidia-container-runtime>2.0.0`, `nvidia-container-toolkit`, `libnvidia-container` and `libnvidia-container-tools`. You can either download prebuilt packages:
- [`nvidia-container-runtime` and `nvidia-container-toolkit`](https://github.com/NVIDIA/nvidia-container-runtime/tree/gh-pages)
- [`libnvidia-container` and `libnvidia-container-tools`](https://github.com/NVIDIA/libnvidia-container/tree/gh-pages)

or build from source, more details [here](https://github.com/NVIDIA/nvidia-container-runtime). Either way, put the binaries somewhere in your `PATH`.

### Configure NVIDIA Container Toolkit for rootless containers

`cgroup` needs to be switched off in `nvidia-container-runtime`.

In case you can ask for a small favor from your sysadmin, just need to find the line that says `#no-cgroups = false` in `/etc/nvidia-container-runtime/config.toml`, uncomment it and set to `true`, i.e `no-cgroups = true`, then continue to [next step](#run-rootless-containers-leveraging-nvidia-gpus).

If not, create a `config.toml` file with the following content

```toml
disable-require = false
#swarm-resource = "DOCKER_RESOURCE_GPU"

[nvidia-container-cli]
#root = "/run/nvidia/driver"
#path = "/usr/bin/nvidia-container-cli"
environment = []
#debug = "/var/log/nvidia-container-runtime-hook.log"
#ldcache = "/etc/ld.so.cache"
load-kmods = true
no-cgroups = true
#user = "root:video"
ldconfig = "@/sbin/ldconfig.real"
```

Create a `nvidia-container-runtime-hook` file under `usernetes/bin`

```sh
#!/bin/sh

/usr/bin/nvidia-container-runtime-hook -config=<absolute-path-to-config.toml> "$@"
```

*The `#!/bin/sh` is important here. Without it you'll probably get an error that contains something like `exec format error`*

and make it executable `chmod +x nvidia-container-runtime-hook`.

### Run rootless containers leveraging NVIDIA GPUs

#### Register `nvidia` runtime

Create a config file at `~/.config/docker/daemon.json` with the following content:

```json
{
    "runtimes": {
        "nvidia": {
            "path": "nvidia-container-runtime",
            "runtimeArgs": []
        }
    }
}
```

#### Run dockerd server

```sh
./run.sh default-docker-nokube
```

#### Run docker client

```sh
docker -H unix://$XDG_RUNTIME_DIR/docker.sock run --gpus all --rm -it nvidia/cuda:10.0-devel nvidia-smi
```

or

```sh
export DOCKER_HOST="unix://$XDG_RUNTIME_DIR/docker.sock"
docker run --gpus all --rm -it nvidia/cuda:10.0-devel nvidia-smi
```

or

```sh
./dockercli.sh run --gpus all --rm -it nvidia/cuda:10.0-devel nvidia-smi
```

You can use `--runtime nvidia` instead of `--gpus all`. However `--gpus` allows more nuanced control. More information on what options can be passed to `--gpus` can be found [here](https://github.com/NVIDIA/nvidia-docker#usage).
