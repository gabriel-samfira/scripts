# Flatcar Container Linux SDK scripts

dummy

Welcome to the scripts repo, your starting place for most things here in the Flatcar Container Linux SDK. To get started you can find our documentation on [the Flatcar docs website][flatcar-docs].

The SDK can be used to
* Patch or update applications or libraries included in the Flatcar OS image
* Add or remove applications and / or libraries
* Modify the kernel configuration and add or remove kernel modules included with Flatcar
* Build OS images for a variety of targets (qemu, bare metal, AWS, Azure, VMWare, etc.)
* And lastly, the SDK can be used to upgrade SDK packages and to build new SDKs

[flatcar-docs]: https://docs.flatcar-linux.org/os/sdk-modifying-flatcar/

# Using the scripts repository: submodules and tags

The repository is meant to be the entry point for Flatcar builds and development.
For building packages, there are 2 additional repositories, [coreos-overlay](https://github.com/flatcar-linux/) and [portage-stable](https://github.com/flatcar-linux/portage-stable), which contain all packages' `ebuild` (build configuration) files.
These repositories are included in `scripts` via git submodules and are used by the SDK container wrapper scripts detailed on further below.
The submodules reside in:
```
scripts
   +--sdk_container
          +---------src
                     +--third_party
                             +------coreos-overlay
                             +------portage-stable
```

When working with the scripts repo always make sure to initialise and to update these submodules; otherwise builds will break because build configuration is missing:
```bash
$ git clone --recurse-submodules https://github.com/flatcar-linux/scripts.git
```

The `scripts` repository makes ample use of tags to mark releases.
Sometimes, local and origin tags can diverge (e.g. when re-tagging something locally to test a build).
Also, `git pull` and `git fetch` do not automatically pull new tags, so long-standing local sourcetrees may lack newer versions.
To fetch and update all tags and to remove tags locally which have been deleted upstream, do
```
$ git pull --all --tags --prune --prune-tags
```
If upstream retagged (of if a tag was changed locally) the corresponding upstream tag will not be pulled so the local tag remains.
In order to override local tags with upstream, run
```
$ git pull --all --tags --prune --prune-tags --force
```

# Using the SDK container

We provide a containerised SDK via https://github.com/orgs/flatcar-linux/packages. The container comes in 3 flavours:
* Full SDK initialised with both architectures supported by Flatcar (amd64 and arm64). This is the largest container, it's about 8GB in size (~3 GB compressed).
* AMD64 SDK initialised for building AMD64 OS images. About 6GB in size (2GB compressed).
* ARM64 SDK initialised for building ARM64 OS images on AMD64 hosts. Also about 6GB in size.
While work on a native ARM64 native SDK is ongoing, it's unfortunately not ready yet. If you want to help, patches are welcome!

The container can be run in one of two ways - "standalone", or integrated with the [scripts](https://github.com/flatcar-linux/scripts) repo:
* Standalone mode will use no host volumes and will allow you to play with the SDK in a sandboxed throw-away environment. In standalone mode, you interface with Docker directly to use the SDK container.
* Integrated mode will closely integrate with the scripts repo directory and bind-mount it as well as the portage-stable and coreos-overlay gitmodules into the container. Integrated mode uses wrapper scripts to interact with the SDK container. This is the recommended way for developing patches for Flatcar.

## Standalone mode

In standalone mode, the SDK is just another Docker container. Interaction with the container happens via use of `docker` directly. Use for experimenting and for throw-away work only, otherwise please use integrated mode (see below).

* Check the list of available versions and pick a version to use. The SDK Major versions correspond to Flatcar Major release versions.
  List of images: `https://github.com/orgs/flatcar-linux/packages/container/package/flatcar-sdk-all`
  For the purpose of this example we'll use version `3033.0.0`.
* Fetch the container image: `docker pull ghcr.io/flatcar-linux/flatcar-sdk-all:3033.0.0`
* Start the image in interactive (tty) mode: `docker run -ti ghcr.io/flatcar-linux/flatcar-sdk-all:3033.0.0`
  You are now inside the SDK container (the hostname will likely differ):
  `sdk@f236fda982a4 ~/trunk/src/scripts $`
* Initialise the SDK in self-contained mode. This needs to be done once per container and will check out the scripts, coreos-overlay, and portage-stable repositories into the container.
  `sdk@f236fda982a4 ../sdk_init_selfcontained.sh`

You can now work with the SDK container.

### Privileged mode when building images

In order to build OS images (via `./build_image` and `./image_to_vm`) the SDK tooling requires privileged access to `/dev`.
This is necessary because the SDK currently employs loop devices to create and to partition OS images.

To start a container in privileged mode with `/dev` available use:
* `docker run -ti  --privileged -v /dev:/dev ghcr.io/flatcar-linux/flatcar-sdk-all:3033.0.0`

## Integrated mode

This is the preferred mode of working with the SDK.
Interaction with the container happens via wrapper scripts from the scripts repository.
Both the host's scripts repo as well as its submodules (portage-stable and coreos-overlay) are made available in the container, allowing for work on these repos directly.
The wrapper scripts will re-use existing containers instead of creating new ones to preserve your work in the container, enabling consistency.

To clone the scripts repo and pick a version:
* Clone the scripts repo: `git clone https://github.com/flatcar-linux/scripts.git`
* Optionally, check out a release tag to base your work on
  * list releases (e.g. all Alpha releases): `git tag -l alpha-*`
  * check out the release version, e.g. `3033.0.0`: `git checkout 3033.0.0`
* Make sure to initialise and fetch git submodules - Flatcar's ebuilds are in 2 separate repositories, connected to `scripts` via submodules.
  * `git submodule init; git submodule update`

To use the SDK container:
* Fetch image and start the SDK container: `./run_sdk_container -t`
  This will fetch the container image of the "scripts" repo's release version you checked out.
  The `-t` command line flag will allocate a TTY, which is preferred for interactive use.
  The command will put you into the SDK container:
  `sdk@sdk-container ~/trunk/src/scripts $`
* Alternatively, you can run individual commands in the SDK container using `./run_sdk_container <command>` (e.g. `./run_sdk_container ./build_packages`).

Subsequent calls to `./run_sdk_container` will re-use the container (as long as the local release version check-out the scripts repo does not change).
Check out `docker container ls --all` and you'll see something like
```
CONTAINER ID   IMAGE                                            COMMAND                  CREATED       STATUS                         PORTS     NAMES
19ea3b6d00ad   ghcr.io/flatcar-linux/flatcar-sdk-all:3033.0.0   "/bin/sh -c /home/sd…"   4 hours ago   Exited (0) About an hour ago             flatcar-sdk-all-3033.0.0_os-3033.0.0
```

Re-use of containers happens on a per-name basis. The above example's container name `flatcar-sdk-all-3033.0.0_os-3033.0.0` is generated automatically. Using `docker container rm` the container can be discarded - a subsequent call to `./run_sdk_container` will create a new one.  Custom containers can be created by use of the `-n <name>` command line option; these will be re-used in subsequent calls to `./run_sdk_container` when using the same `<name>`.

The local sourcetree can also be used with an entirely custom SDK container image. Users must ensure that the image is either fetch-able or present locally. The custom image can be specified using `-C <custom-image>`. This option is useful e.g. for building the local sourcetree with different SDK versions.

Check out `./run_sdk_container -h` for more information on command line options.

# Building a new SDK container

Building an SDK container is done using `./build_sdk_container_image <tarball>`.
The tarball input is the result of an SDK bootstrap (see below). Version information for both OS as well as for the SDK will be extracted from the tarball name.
The version file will be updated accordingly before the SDK container is built.
During the build, toolchain packages will be built and installed into the SDK container image. Both supported boards (`amd64-usr` and `arm64-usr`) will be initialised in the container image.

# Bootstrapping a new SDK tarball using the SDK container

The script `./bootstrap_sdk_container` bootstraps a new SDK tarball using an existing SDK container and seed tarball. Specifying the seed version as well as the designated new SDK version is required for this script.

# Automation stubs for continuous integration

Script stubs for various build stages can be found in the [ci-automation](ci-automation) folder. These are helpful for gluing Flatcar Container Linux builds to a continuous integration system.
