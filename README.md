# Rocket - App Container runtime

[![Build Status](https://travis-ci.org/coreos/rocket.png?branch=master)](https://travis-ci.org/coreos/rocket)

_Release early, release often: Rocket is currently a prototype and we are seeking your feedback via issues and pull requests_

Rocket is a CLI for running App Containers. The goal of rocket is to be composable, secure, and fast.

[Read more about Rocket in the launch announcement](https://coreos.com/blog/rocket).

![Rocket Logo](logos/rocket-horizontal-color.png)

## Trying out Rocket

`rkt` is currently supported on amd64 Linux. We recommend booting up a fresh virtual machine to test out rocket.

To install the `rkt` binary, grab the release directly from GitHub:

```
wget https://github.com/coreos/rocket/releases/download/v0.2.0/rocket-v0.2.0.tar.gz
tar xzvf rocket-v0.2.0.tar.gz
cd rocket-v0.2.0
./rkt help
```

Keep in mind while running through the examples that right now `rkt` needs to be run as root for most operations.

## Rocket basics

### Downloading an App Container Image (ACI)

Rocket uses content addressable storage (CAS) for storing an ACI on disk. In this example, the image is downloaded and added to the CAS.

(Note that since Rocket verifies signatures by default, you will need to first [trust](https://github.com/coreos/rocket/blob/master/Documentation/signing-and-verification-guide.md#establishing-trust) the [CoreOS public key](https://coreos.com/dist/pubkeys/aci-pubkeys.gpg) used to sign the image. Detailed step-by-step for the signing procedure [is here](Documentation/getting-started-ubuntu-trusty.md#trust-the-coreos-signing-key).)

```
[~/rocket-v0.2.0]$ sudo ./rkt fetch https://github.com/coreos/etcd/releases/download/v2.0.0-rc.1/etcd-v2.0.0-rc.1-linux-amd64.aci
rkt: fetching image from https://github.com/coreos/etcd/releases/download/v2.0.0-rc.1/etcd-v2.0.0-rc.1-linux-amd64.aci
Downloading aci: [===============================              ] 2.49 MB/3.58 MB
Downloading signature from https://github.com/coreos/etcd/releases/download/v2.0.0-rc.1/etcd-v2.0.0-rc.1-linux-amd64.sig
rkt: signature verified:
  CoreOS ACI Builder <release@coreos.com>
  sha512-fcdf12587358af6ebe69b5338a05df67
```

These files are now written to disk:

```
[~]$ find /var/lib/rkt/cas/blob/
/var/lib/rkt/cas/blob/
/var/lib/rkt/cas/blob/sha512
/var/lib/rkt/cas/blob/sha512/fc
/var/lib/rkt/cas/blob/sha512/fc/sha512-fcdf12587358af6ebe69b5338a05df673ab7c95539f4cac09b0ceb4ea9b12339
```

Per the [App Container Specification](https://github.com/appc/spec/blob/master/SPEC.md#image-archives), the SHA-512 hash is of the tarball and can be reproduced with other tools:

```
[~]$ wget https://github.com/coreos/etcd/releases/download/v2.0.0-rc.1/etcd-v2.0.0-rc.1-linux-amd64.aci
...
[~]$ $ gzip -dc etcd-v2.0.0-rc.1-linux-amd64.aci > etcd-v2.0.0-rc.1-linux-amd64.tar
[~]$ sha512sum etcd-v2.0.0-rc.1-linux-amd64.tar
fcdf12587358af6ebe69b5338a05df673ab7c95539f4cac09b0ceb4ea9b1233922700bdbafd5b6783e129d3f5e9d17bc7f0a07912b8a0a8c0ff7bf732a3f0acf  etcd-v2.0.0-rc.1-linux-amd64.tar
```

### Launching an ACI

An ACI can be run by pointing `rkt` at either the ACI's hash or URL.

```
# Example of running via ACI hash
[~/rocket-v0.2.0]$ sudo ./rkt run sha512-fcdf12587358af6ebe69b5338a05df67
...
Press ^] three times to kill container
```

```
# Example of running via ACI URL
[~/rocket-v0.2.0]$ sudo ./rkt run https://github.com/coreos/etcd/releases/download/v2.0.0-rc.1/etcd-v2.0.0-rc.1-linux-amd64.aci
...
Press ^] three times to kill container
```

`rkt` will do the appropriate ETag checking on the URL to make sure it has the most up to date version of the image.

The escape character ```^]``` is generated by ```Ctrl-]``` on a US keyboard. The required key combination will differ on other keyboard layouts. For example, the Swedish keyboard layout uses ```Ctrl-å``` on OS X and ```Ctrl-^``` on Windows to generate the ```^]``` escape character.

## App Container basics

[App Container][appc-repo] is a [specification][appc-spec] of an image format, runtime, and discovery protocol for running a container. We anticipate app container will be adopted by other runtimes outside of Rocket itself. Read more about it [here][appc-repo].

To validate the `rkt` with the App Container [validation ACIs][appc-readme] run:

```
[~/rocket-v0.2.0]$ sudo ./rkt run -volume database:/tmp \
	https://github.com/appc/spec/releases/download/v0.1.1/ace-validator-main.aci \
	https://github.com/appc/spec/releases/download/v0.1.1/ace-validator-sidekick.aci
```

[appc-repo]: https://github.com/appc/spec/
[appc-spec]: https://github.com/appc/spec/blob/master/SPEC.md
[appc-readme]: https://github.com/appc/spec/blob/master/README.md

## Rocket internals

Rocket is designed to be modular and pluggable by default. To do this we have a concept of "stages" of execution of the container.

Execution with Rocket is divided into a number of distinct stages. The motivation for this is to separate the concerns of initial filesystem setup, execution environment, and finally the execution of the apps themselves.

### Stage 0

The first step of the process, stage 0, is the actual `rkt` binary itself. This binary is in charge of doing a number of initial preparatory tasks:
- Generating a Container UUID
- Generating a Container Runtime Manifest
- Creating a filesystem for the container
- Setting up stage 1 and stage 2 directories in the filesystem
- Copying the stage1 binary into the container filesystem
- Fetching the specified ACIs
- Unpacking the ACIs and copying each app into the stage2 directories

Given a run command such as:

```
[~/rocket-v0.2.0]$ sudo ./rkt run --volume bind:/opt/tenant1/database \
	sha512-8a30f14877cd8065939e3912542a17d1a5fd9b4c \
	sha512-abcd29837d89389s9d0898ds908ds890df890908
```

a container manifest compliant with the ACE spec will be generated, and the filesystem created by stage0 should be:

```
/container
/stage1
/stage1/init
/stage1/opt
/stage1/opt/stage2/sha512-648db489d57363b29f1597d4312b212928a48e9a7ee2baec7a237629cce5a6744632d047708c9486dec6b105b8a2d298b622daa5654c1b21af36bfc9534417d7
/stage1/opt/stage2/sha512-0c45e8c0ab2b3cdb9ec6649073d5c6c43f4f1ed9ebd97b2ebfc2290c21ee88ae63bff32c23690f7c96b666ffc353f38c3f2977c4f019176b12c74f9683e91141
```

where:
- `container` is the container manifest file
- `stage1` is a copy of the stage1 filesystem that is safe for read/write
- `stage1/init` is the actual stage1 binary to be executed
- `stage1/opt/stage2` are copies of the unpacked ACIs

At this point the stage0 execs `/stage1/init` with the current working directory set to the root of the new filesystem.

### Stage 1

The next stage is a binary that the user trusts to set up cgroups, execute processes, and other operations as root. This stage has the responsibility to take the execution group filesystem that was created by stage 0 and create the necessary cgroups, namespaces and mounts to launch the execution group:

- Generate systemd unit files from the Application and Container Manifests (containing, respectively, the exec specifications of each container and the ordering given by the user)
- Set up any external volumes (undefined at this point)
- nspawn attaching to the bridge and launch the execution group systemd
- Launch the root systemd
- Have the root systemd

This process is slightly different for the qemu-kvm stage1 but a similar workflow starting at `exec()`'ing kvm instead of an nspawn.

### Stage 2

The final stage is executing the actual application. The responsibilities of the stage2 include:

- Launch the init process described in the Application Manifest
