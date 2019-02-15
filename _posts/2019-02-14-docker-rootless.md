---
layout: post
title:  "Run docker containers without root privilege"
date:   2019-02-14 18:32:04 +0800
categories: jekyll update
---

Docker containers can be run without root privilege using [usernetes](https://github.com/rootless-containers/usernetes). This will be available in upstream `docker` soon™ following [moby/moby#38050](https://github.com/moby/moby/pull/38050). Use this guide for now.

*NOTE: In order to run this, user must have a range of [subuid(5)](http://man7.org/linux/man-pages/man5/subuid.5.html)s available to them, i.e they must be present in `/etc/subuid`. subuid range can be added by editting `/etc/subuid` directly or by running `sudo usermod --add-subuids <from>-<to> <user>`, e.g `sudo usermod --add-subuids 10000-65536 user`*

- [rootless containers using usernetes](#running-rootless-containers-using-usernetes)
- [nvidia-docker](#nvidia-docker)
  - [Get nvidia-docker](#get-nvidia-docker)
  - [Configure nvidia-docker for running rootless containers](#configure-nvidia-docker-for-running-rootless-containers)
  - [Run rootless containers with nvidia runtime](#run-rootless-containers-with-nvidia-runtime)

## Running rootless containers using usernetes

```bash
# grab a build from https://github.com/rootless-containers/usernetes/releases
wget https://github.com/rootless-containers/usernetes/releases/download/v20190212.0/usernetes-x86_64.tbz
tar xjvf usernetes-x86_64.tbz
cd usernetes
```

Run `dockerd` server

```bash
./run.sh default-docker-nokube
```

You can now run rootless containers.

If you already have upstream `docker` installed system-wide

```bash
# docker -H unix://$XDG_RUNTIME_DIR/docker.sock <cmd>
docker -H unix://$XDG_RUNTIME_DIR/docker.sock run --rm -it busybox ls
```

If you don't

```bash
# ./dockercli.sh <cmd>
./dockercli.sh run --rm -it busybox ls
```

## nvidia-docker

Need some tweaks for this to work with `nvidia-docker`

### Get nvidia-docker

If you already have `nvidia-docker` installed, continue to [the next section](#configure-nvidia-docker-for-running-rootless-containers).

*\*UNTESTED\**

If not, you need to get `nvidia-container-runtime`, `nvidia-container-runtime-hook`, `libnvidia-container` and `libnvidia-container-tools`. You can either download prebuilt packages:

<https://nvidia.github.io/nvidia-container-runtime/centos7/x86_64/nvidia-container-runtime-2.0.0-1.docker18.03.0.x86_64.rpm>  
<https://nvidia.github.io/nvidia-container-runtime/centos7/x86_64/nvidia-container-runtime-hook-1.3.0-1.x86_64.rpm>  
<https://nvidia.github.io/libnvidia-container/centos7/x86_64/libnvidia-container1-1.0.0-0.1.beta.1.x86_64.rpm>  
<https://nvidia.github.io/libnvidia-container/centos7/x86_64/libnvidia-container-tools-1.0.0-0.1.beta.1.x86_64.rpm>  

(urls may vary depending on version and distro) or build from source, more details [here](https://github.com/NVIDIA/nvidia-container-runtime). Either way, put the binaries somewhere in your `PATH`.

### Configure nvidia-docker for running rootless containers

In case you can ask for a small favor from your sysadmin:

Open `/etc/nvidia-container-runtime/config.toml`, find the line that says `#no-cgroups = false`, uncomment it and set to `true`, i.e `no-cgroups = true`, continue to [the next section](#run-rootless-containers-with-nvidia-runtime).

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

Create a `nvidia-container-runtime-hook` file:

```sh
#!/bin/sh

/usr/bin/nvidia-container-runtime-hook -config=<absolute-path-to-config.toml> "$@"
```

make it executable `chmod +x nvidia-container-runtime-hook` and prepend the directory that contains this file to your `PATH`. 
E.g, assume you put this file in `~/bin`, add this line to your `.bashrc` or `.zshrc`: `export PATH=~/bin:$PATH`

### Run rootless containers with nvidia runtime

#### Install `usernetes` if you haven't

```bash
# grab a build from https://github.com/rootless-containers/usernetes/releases
wget https://github.com/rootless-containers/usernetes/releases/download/v20190212.0/usernetes-x86_64.tbz
tar xjvf usernetes-x86_64.tbz
cd usernetes
```

#### Register `nvidia` runtime

Open `./Taskfile.yaml`, look for this part

```yaml
dockerd:
  cmds:
    - ./boot/dockerd.sh
```

change the command to `./boot/dockerd.sh --add-runtime "nvidia=/usr/bin/nvidia-container-runtime"`

or create a `config.json` file with the following content

```json
{
    "experimental": true,
    "runtimes": {
        "nvidia": {
            "path": "/usr/bin/nvidia-container-runtime",
            "runtimeArgs": []
        }
    }
}
```

and change the command to `./boot/dockerd.sh --config-file="<absolute-path-to-config-file>"`

#### Run dockerd server

```bash
./run.sh default-docker-nokube
```

#### Run docker client

```bash
docker -H unix://$XDG_RUNTIME_DIR/docker.sock run --runtime=nvidia --rm -it nvidia/cuda:10.0-devel nvidia-smi
```

or

```bash
./dockercli.sh run --runtime=nvidia --rm -it nvidia/cuda:10.0-devel nvidia-smi
```
