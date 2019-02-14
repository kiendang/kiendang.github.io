---
layout: post
title:  "Run docker containers without root privilege"
date:   2019-02-14 18:32:04 +0800
categories: jekyll update
---

Docker containers can be run without root privilege using [usernetes](https://github.com/rootless-containers/usernetes). This will be available in upstream `docker` soon™ following [moby/moby#38050](https://github.com/moby/moby/pull/38050). Use this guide for now.

- [rootless containers using usernetes](#running-rootless-containers-using-usernetes)
- [nvidia-docker](#nvidia-docker)
  - [Preparation](#preparation)
    - [Quick way (require root to set up)](#quick-way)
    - [Long way (root not required) *incomplete*](#long-but-does-not-require-root-to-set-up)
  - [Run rootless containers with nvidia runtime](#run-rootless-containers-with-nvidia-runtime)

## Running rootless containers using usernetes

```bash
git clone --depth 1 https://github.com/rootless-containers/usernetes.git
cd usernetes
make
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

### Preparation

#### Quick way

Required:
- Have `nvidia-docker` installed system-wide
- A small favor from your sysadmin

Open `/etc/nvidia-container-runtime/config.toml`, find the line that says `#no-cgroups = false`, uncomment it and set to `true`, i.e `no-cgroups = true`

#### Long (but does not require root to set up)

##### Install `nvidia-container-runtime`

Create a `config.toml` file with the following content

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

Getting `nvidia-container-runtime`

```bash
git clone --depth 1 https://github.com/NVIDIA/nvidia-container-runtime.git
cd nvidia-container-runtime
```

Edit `./hook/nvidia-container-runtime-hook/hook_config.go`. Find the line that says `configPath = "/etc/nvidia-container-runtime/config.toml"`, point it to the `config.toml` file you created earlier.

Compile `nvidia-container-runtime`

```bash
# make <your_distro>
make ubuntu16.04
```

Take a look at the `Makefile` for supported distros.

*to be updated*

### Run rootless containers with nvidia runtime

#### Install `usernetes` if you haven't

```bash
git clone --depth 1 https://github.com/rootless-containers/usernetes.git
cd usernetes
make
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

and change the command to `./boot/dockerd.sh --config-file="<path-to-config-file>"`

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
