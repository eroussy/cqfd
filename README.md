![cqfd logo](./doc/cqfd_logo.png?raw=true)

# What is cqfd ? #

cqfd provides a quick and convenient way to run commands in the current
directory, but within a Docker container defined in a per-project config
file.

This becomes useful when building an application designed for another
Linux system, e.g. building an old embedded firmware that only works
in an older Linux distribution.

# Using cqfd #

## Getting started ##

Just follow these steps:

* Install cqfd (see below)
* Make sure your user is a member of the ``docker`` group
* Go into your project's directory
* Create a .cqfdrc file
* Put a Dockerfile and save it as .cqfd/docker/Dockerfile
* Run ``cqfd init``

Examples are available in the samples/ directory.

cqfd will use the provided Dockerfile to create a normalized runtime
build environment for your project.

Warning : `cqfd init` creates and names a new docker image every time
the Dockerfile is modified. This could result in a huge amount of unused
images that cannot be purge automatically. cqfd doesn't implement any
systematic clean system for now.

## Using cqfd on a daily basis ##

### Regular builds ###

To build your project from the configured build environment with the
default build command as configured in .cqfdrc, use:

    $ cqfd

Alternatively, you may want to specify a custom build command to be
executed from inside the build container.

    $ cqfd run make clean
    $ cqfd run "make linux-dirclean && make foobar-dirclean"

When ``cqfd`` is running, the current directory is mounted by Docker
as a volume. As a result, all the build artefacts generated inside the
container are still accessible in this directory after the container
has been stopped and removed.

### Release ###

The release command behaves exactly like run, but creates a release
tarball for your project additionally. The release files (as specified
in your ``.cqfdrc``) will be included inside the release archive.

    $ cqfd release

The resulting release file is then called according to the archive
template, which defaults to ``%Po-%Pn.tar.xz``.

### Flavors ###

Flavors are used to create alternate build scenarios. For example, to
use another container or another build command.

## The .cqfdrc file ##

The .cqfdrc file at the root of your project contains the information
required to support project tooling. samples/dot-cqfdrc is an example.

Here is a sample .cqfdrc file:

    [project]
    org='fooinc'
    name='buildroot'

    [build]
    command='make foobar_defconfig && make && asciidoc README.FOOINC'
    files='README.FOOINC output/images/sdcard.img'
    archive='cqfd-%Gh.tar.xz'

### The [project] section ###

``org``: a short, lowercase name for the project’s parent organization.

``name``: a short, lowercase name for the project.

``build_context`` (optional): a directory to pass as the build context
to Docker. This should be specified relatively to where cqfd is
invoked.  For example, it can be set to `.`, to use the current
working directory of the invoked `cqfd` command as the Docker build
context, which can be useful when files at the root of the project are
required to build the image.  When using this option, a
``.dockerignore`` file can be useful to limit what gets sent to the
Docker daemon.

Generated Docker images for your project will be named $org_$name.

### The [build] section ###

``command``: the command (or list of commands) to be executed when cqfd is
invoked. This string will be passed as an argument to a classical
``bash -c "commands"``, within the build container, to generate the
build artefacts.

``files``: an optional space-separated list of files generated by the
build process that we want to include inside a standard release
archive.

``archive``: the optional name of the release archive generated by
cqfd. You can include environment variable names, as well as the
following template marks:

* ``%Gh`` - git short hash of last commit
* ``%GH`` - git long hash of last commit
* ``%D3`` - RFC3339 date (YYYY-MM-DD)
* ``%Cf`` - current cqfd flavor name (if any)
* ``%Po`` - value of the ``project.org`` configuration key
* ``%Pn`` - value of the ``project.name`` configuration key
* ``%%`` - a litteral '%' sign

By default, cqfd will generate a release archive named
``org-name.tar.xz``, where 'org' and 'name' come from the project's
configuration keys. The .tar.xz, .tar.gz and .zip archive formats are
supported.

For tar archives:

* Setting ``tar_transform=yes`` will cause all files specified for the
  archive to be stored at the root of the archive, which is desired in
  some scenarios.

* Setting ``tar_options`` will pass extra options to
  the tar command. For example, setting ``tar_options=-h`` will copy
  all symlink files as hardlinks, which is desired in some scenarios.

``distro``: the name of the directory containing the Dockerfile. By
default, cqfd uses ``"docker"``, and ``.cqfd/docker/Dockerfile` is
used.

``user_extra_groups``: an optional, space-separated list of groups the user
should be a member of in the container. You can either use the ``group:gid``
format, or simply specify the ``group`` name if it exists either in the host or
inside the docker image.

``flavors``: the list of build flavors (see below). Each flavor has its
own command just like build.command.

``docker_run_args`` (optional): arguments used to invoke `docker run`.
For example, to share networking with the host, it can be set like:
```
docker_run_args='--network=host'
```

### Using build flavors ###

In some cases, it may be desirable to build the project using
variations of the build and release methods (for example a debug
build). This is made possible in cqfd with the build flavors feature.

In the .cqfdrc file, one or more flavors may be listed in the
``[build]`` section, referencing other sections named following
flavor's name.

    [centos7]
    command='make CENTOS=1'
    distro='centos7'

    [debug]
    command='make DEBUG=1'
    files='myprogram Symbols.map'

    [build]
    command='make'
    files='myprogram'
    flavors='centos7 debug'

A flavor will typically redefine some keys of the build section:
command, files, archive, distro.

Flavors from a `.cqfdrc` file can be listed using the `flavors` argument.

### Environment variables ###

The following environment variables are supported by cqfd to provide
the user with extra flexibility during his day-to-day development
tasks:

``CQFD_EXTRA_RUN_ARGS``: A space-separated list of additional
docker-run options to be append to the starting container.
Format is the same as (and passed to) docker-run’s options.
See 'docker run --help'.

``CQFD_EXTRA_BUILD_ARGS``: A space-separated list of additional
docker-build options to be append to the building image.
Format is the same as (and passed to) docker-build’s options.
See 'docker build --help'.

### Other command-line options ###

In some conditions you may want to use alternate cqfd filenames and / or an
external working directory. These options can be used to control the cqfd
configuration files:

The working directory can be changed using the `-C` option:

    $ cqfd -C external/directory

An alternate cqfd directory can be specified with the `-d` option:

    $ cqfd -d cqfd_alt

An alternate cqfdrc file can be specified with the `-f` option:

    $ cqfd -f cqfdrc_alt

These options can be combined together:

    $ cqfd -C external/directory -d cqfd_alt -f cqfdrc_alt
    $ # cqfd will use:
    $ #  - cqfd directory: external/directory/cqfd_alt
    $ #  - cqfdrc file: external/directory/cqfdrc_alt

## Build Container Environment ##

When cqfd runs, a docker container is launched as the environment in
which to run the *command*.  Within this environment, commands are run
as the same user as the one invoking cqfd (with a fallback to the
'builder' user in case it cannot be determined). So that this user has
access to local files, the current working directory is mapped to
the same location inside the container.

### SSH Handling ###

The local ~/.ssh directory is also mapped to the corresponding
directory in the build container. This effectively enables SSH agent
forwarding so a build can, for example, pull authenticated git repos.

### Terminal job control ###

When cqfd runs a command as the unprivileged user that called it in
the first place, ``su(1)`` is used to run the command. This brings a
limitation for processes that require a controlling terminal (such as
an interactive shell), as ``su`` will prevent the command executed
from having one.

```
$ cqfd run bash
bash: cannot set terminal process group (-1): Inappropriate ioctl for device
bash: no job control in this shell
```

To workaround this limitation, cqfd will use ``sudo(8)`` when it is
available in the container instead. The user is responsible for
including it in the related Dockerfile.

## Requirements ##

To use cqfd, ensure the following requirements are satisfied on your
workstation:

-  Bash 4.x

-  Docker

-  A ``docker`` group in your /etc/group

-  Your username is a member of the ``docker`` group

-  Restart your docker service if you needed to create the group.

## Installing/removing cqfd ##

If you use the [GNU Guix](https://gnu.org/software/guix) package
manager, you can install `cqfd` via:

```sh
guix install cqfd
```

Otherwise, the following describes how you can install it system-wide
from source.

Install or remove the script and its resources:

    $ make install
    $ make uninstall

Makefile honors both **PREFIX** (__/usr/local__) and **DESTDIR** (__[empty]__)
variables:

    $ make install PREFIX=/opt
    $ make install PREFIX=/usr DESTDIR=package

## Testing cqfd (for developers) ##

The codebase contains tests which can be invoked using the following
command, if the above requirements are met on the system:

    $ make tests

## Trivia ##

CQFD stands for "Ce qu'il fallait Dockeriser", french for "what needed
to be dockerized".
