# Docker Images for balena CLI

Docker images with balena CLI and docker-in-docker.

## Features Overview

These CLI images are based on the popular [Balena base images](https://www.balena.io/docs/reference/base-images/base-images/)
so they include many of the features you see there.

- Multiple Architectures:
    - `rpi`
    - `armv7hf`
    - `aarch64` (debian only)
    - `amd64`
    - `i386`
- Multiple Distributions
    - `debian`
    - `alpine`
- [cross-build](https://www.balena.io/docs/reference/base-images/base-images/#building-arm-containers-on-x86-machines) functionality for building ARM containers on x86.
- Helpful package installer script called `install_packages` inspired by [minideb](https://github.com/bitnami/minideb#why-use-minideb).

## Image Names

`balenalib/<arch>-<distro>-balenacli:<cli_ver>`

- `<arch>` is the architecture and is mandatory. If using Dockerfile.templates, you can replace this with `%%BALENA_ARCH%%`.
For a list of available device names and architectures, see the [Device types](https://www.balena.io/docs/reference/base-images/devicetypes/).
- `<distro>` is the Linux distribution and is mandatory. Currently there are 2 distributions, namely `debian` and `alpine`.

## Image Tags

In the tags, all of the fields are optional, and if they are left out, they will default to their `latest` pointer.

- `<cli_ver>` is the version of the balena CLI, for example, `12.40.2`, it can also be substituted for `latest`.

## Examples

`balenalib/amd64-debian-balenacli:12.40.2`

- `<arch>`: amd64 - suitable for running on most workstations
- `<distro>`: debian - widely used base distro
- `<cli_ver>`: 12.40.2

`balenalib/armv7hf-alpine-balenacli`

- `<arch>`: armv7hf - suitable for running on a Raspberry Pi 3 for example
- `<distro>`: alpine - smaller footprint than debian
- `<cli_ver>`: omitted - the latest available CLI version will be used

## Basic Usage

Here's a small example of running a single, detached container
in the background and using `docker exec` to run balena CLI commands.

```
$ docker run --detach --privileged --network host --name cli --rm -it balenalib/amd64-debian-balenacli /bin/bash

$ docker exec -it cli balena version -a
balena-cli version "12.38.1"
Node.js version "12.19.1"

$ docker exec -it cli balena login --token abc...

$ docker exec -it cli balena whoami
== ACCOUNT INFORMATION
USERNAME: ...
EMAIL:    ...
URL:      balena-cloud.com

$ docker exec -it cli balena apps
ID      APP NAME         SLUG                            DEVICE TYPE     ONLINE DEVICES DEVICE COUNT
1491721 test-nuc         gh_paulo_castro/test-nuc        intel-nuc       0              1
...

$ docker exec -it cli balena app test-nuc
== test-nuc
ID:          149...
DEVICE TYPE: intel-nuc
SLUG:        gh_.../test-nuc
COMMIT:      ce9...
```

## Advanced Usage

The following are examples of running the docker image in various
modes in order to allow only the required functionality, and not
elevate permissions unless required.

### scan

- <https://www.balena.io/docs/reference/balena-cli/#scan>

NOTE: Currently the `balena scan` command does not work with Docker Desktop
on Windows or MacOS due to the VM using a separate network interface
in a different address range by default.

```bash
# balena scan requires the host network
docker run --rm -it --network host balenalib/amd64-debian-balenacli scan
```

### ssh

- <https://www.balena.io/docs/reference/balena-cli/#login>
- <https://www.balena.io/docs/reference/balena-cli/#key-add-name-path>
- <https://www.balena.io/docs/reference/balena-cli/#ssh-applicationordevice-service>

```bash
# balena ssh requires a private ssh key
docker run --rm -it -e SSH_PRIVATE_KEY="$(</path/to/priv/key)" \
    balenalib/amd64-debian-balenacli /bin/bash

> balena login --credentials --email johndoe@gmail.com --password secret
> balena ssh f49cefd my-service
> exit

# OR use your host ssh agent socket with a key already loaded
docker run --rm -it -e SSH_AUTH_SOCK -v "$(dirname "${SSH_AUTH_SOCK}")" \
    balenalib/amd64-debian-balenacli /bin/bash

> balena login --credentials --email johndoe@gmail.com --password secret
> balena ssh f49cefd my-service
> exit
```

### build | deploy

- <https://www.balena.io/docs/reference/balena-cli/#build-source>
- <https://www.balena.io/docs/reference/balena-cli/#deploy-appname-image>

```bash
# docker-in-docker requires SYS_ADMIN
# note that we are mounting your app source into the container
# with -v $PWD:$PWD -w $PWD for convenience
docker run --rm -it --cap-add SYS_ADMIN \
    -v $PWD:$PWD -w $PWD \
    balenalib/amd64-debian-balenacli /bin/bash

> balena login --credentials --email johndoe@gmail.com --password secret
> balena build --application myApp
> balena deploy myApp
> exit

# OR use your host docker socket
# note that we are mounting your app source into the container
# with -v $PWD:$PWD -w $PWD for convenience
docker run --rm -it -v /var/run/docker.sock:/var/run/docker.sock \
    -v $PWD:$PWD -w $PWD \
    balenalib/amd64-debian-balenacli /bin/bash

> balena login --credentials --email johndoe@gmail.com --password secret
> balena build --application myApp
> balena deploy myApp
> exit
```

### preload

- <https://www.balena.io/docs/reference/balena-cli/#os-download-type>
- <https://www.balena.io/docs/reference/balena-cli/#os-configure-image>
- <https://www.balena.io/docs/reference/balena-cli/#preload-image>

```bash
# docker-in-docker requires SYS_ADMIN
docker run --rm -it --cap-add SYS_ADMIN \
    balenalib/amd64-debian-balenacli /bin/bash

> balena login --credentials --email johndoe@gmail.com --password secret
> balena os download raspberrypi3 -o raspberry-pi.img
> balena os configure raspberry-pi.img --app MyApp
> balena preload raspberry-pi.img --app MyApp --commit current
> exit

# OR use your host docker socket
# note the .img path must be the same on the host as in the container
# therefore we are using -v $PWD:$PWD -w $PWD so the paths align
docker run --rm -it -v /var/run/docker.sock:/var/run/docker.sock \
    -v $PWD:$PWD -w $PWD \
    balenalib/amd64-debian-balenacli /bin/bash

> balena login --credentials --email johndoe@gmail.com --password secret
> balena os download raspberrypi3 -o raspberry-pi.img
> balena os configure raspberry-pi.img --app MyApp
> balena preload raspberry-pi.img --app MyApp --commit current
> exit
```

## Custom images / contributing

The following steps may be used to create custom CLI images or
to contribute bug reports, fixes or features.

```bash
# the currently supported base images are 'debian' and 'alpine'
export BALENA_DISTRO="debian"

# provide the architecture where you will be testing the image
export BALENA_ARCH="amd64"

# optionally enable QEMU binfmt if building for other architectures (eg. armv7hf)
docker run --rm --privileged multiarch/qemu-user-static --reset -p yes

# build and tag an image with docker
docker build -f docker/${BALENA_DISTRO}/Dockerfile --build-arg BALENA_ARCH \
    --tag "balenalib/${BALENA_DISTRO}-${BALENA_ARCH}-balenacli" \
    --pull
```
