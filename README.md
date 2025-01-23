<p align="center">
  <img src="https://github.com/macaroni-os/macaroni-site/blob/master/site/static/images/logo.png">
</p>

# M.A.R.K. Stages Specification for mark-devkit

Specification to use with [mark-devkit](https://github.com/macaroni-os/mark-devkit/)
to build Stage tarballs from existing tarball or from Anise binary.

NOTE: The specs aren't yet ready for users. WIP.

## Build `mark-devkit` from master

Prerequisite `dev-lang/go` and `make`.

```shell
git clone https://github.com/macaroni-os/mark-devkit

cd mark-devkit

make build

cp mark-devkit /usr/bin
```

## Install `mark-devkit`

### M.A.R.K Stack

```shell
emerge mark-devkit
```

### Macaroni OS

```shell
anise i mark-devkit
```

## Getting started with mark-devkit

To build your stage tarball you need to have a MARK or Macaroni rootfs with the
following packages:
- `fchroot` (and dependencies)
- `ego`
- `yq` for anise source.

Clone the `mark-stages` repository:

```shell
git clone https://github.com/macaroni-os/mark-stages.git

cd mark-stages
```

and later run the `metro run` command for the selected job.

```shell
mark-devkit metro run --specfile ./jobs/x86_64/stage3_x86_64.yaml --config ./etc/config.yml  \
    --job stage3-x86_64bit \
    --fchroot-debug
```

I suggest to developers to use `--cleanup=false` option and avoid to remove rootfs filesystem
at the end of the task.

These options are also helpful for devs:

* `--skip-source-phase`: If you have executed already one time the build and it's failed. You can
  skip the download and extract phase.

* `--skip-packer-phase`: This skip the final phase that create the stage3 tarball.

If it's used the `--cleanup=false` option could be used `fchroot` to enter on chroot and
analyze issues:

```
fchroot /workspace/rootfs /bin/bash
```

## Diagnose the rendered Job hooks

`mark-devkit` uses job's options to renderize the *hooks* files with the Helm Golang template engine.

This command permits to see the generated hooks:

```shell
mark-devkit diagnose job --job stage3-x86_64bit --specfile jobs/stage3_x86_64.yaml
```

## What is a job?

The *job* of the `metro run` command is the task to generate a *stage* tarball.
It has two major entities, a *source* and an *output* and different configuration
attributes.

The available attributes are:

| Attribute Name | Attribute Description |
|----------------|-----------------------|
| `name` | The name of the job to use from command line cli |
| `options`| Contains custom options that could be used by user to specify variables to use with Helm render engine. The options are custom key/values. |
| `environments` | Contains an arrey of environment variables passed to the hooks executor. |
| `workspacedir` | The workspace base dir used by `mark-devkit`. |
| `chroot_binds` | Optional binds options to pass at `fchroot`. |
| `hooks_basedir` | The base dir used to retrieve the hooks to run. If the path is relative, that path is joined with job file directory. |
| `hooks_files` | Contains the list of files with the hooks to run. |

### The source entity

The *source* describes retrieving the data and files used in the starting rootfs.
There are two different types of *source* at the moment supported: *http* and *anise*.
The *http* source permits to fetch of a tarball (optionally compressed in .gz, .bzip2,
.zstd or .xz) from an HTTP resource.

This is an example of an http source:

```yaml
source:
    type: http
    uri: "https://build.funtoo.org/{{ .Values.release }}/{{ .Values.arch }}/{{ .Values.subarch }}/stage3-latest.tar.xz"
    target: /workspace/sourcer/{{ .Values.release }}/{{ .Values.arch }}/{{ .Values.subarch }}/stage3-latest-{{ now | date "2006-01-02" }}.tar.xz
```

The fields *uri* and *target* are rendered by Helm template engine using the variables
defined on *options* attribute.

The *anise* is a particular source that is created from scratch with the installation
of a list of binary packages available from one or more *anise* repositories. This
means that the installed packages must be at least the minimal number of packages
to have a working rootfs.

This is an example of an anise source:

```yaml
source:
    type: anise
    # This path depends on execution path.
    anise_config: anise/config.yaml
    anise_repos:
      - repository/mottainai-stable
      - repository/macaroni-terragon
      - repository/macaroni-commons
    anise_packages:
      - system/luet-geaaru-thin
      - sys-apps/baselayout
      - toolchain/base
      - system/entities
      - whip
      - whip-catalog
      - whip-profiles/macaroni
      - app-admin/macaronictl-thin
      - virtual/sh
      - virtual/base
```

> NOTE: The *anise* config must be configured with the `system.rootfs` param set to the
>       same path of the job.


### The output entity

The *output* describes what is the final file to generate from the rootfs created
and where are been executed the hooks.

The *output* describes what is the final tarball to generate from the rootfs created and where are
been executed the hooks. At the moment the only supported type of output is *file*.

This is an example of the *output* YAML specification:

```yaml
output:
    type: file
    name: stage3-{{ .Values.arch }}-{{ .Values.subarch }}-{{ .Values.release }}{{- if .Values.extras }}+{{- end }}{{ join "+" .Values.extras }}-{{ now | date "2006-01-02" }}.tar.xz
    dir: '../../output/terragon/{{ .Values.release }}/{{ .Values.arch }}/{{ .Values.subarch }}/{{ now | date "2006-01-02" }}/'
```

The fields *name* and *dir* are rendered with the fields defined in the *options* attribute of
the job.

Could be used the command `diagnose job` to validate the rendered output of the fields:

```shell
$> mark-devkit diagnose job --specfile jobs/x86_64/stage3_terragon.yaml  --job stage3-terragon-next-minimal | grep stage3-x86
        name: stage3-x86-64bit-generic_64-next+terragon+minimal-2024-09-14.tar.xz

```

### Different hooks types

The hooks could be divided into two families: *inner* and *outer*.
The *outer* are executed directly in the host; instead, the *inner*
are executed inside the rootfs of chroot through `fchroot`.

The hooks *outer* are then divided into:

* outer-pre-sourcer
* outer-pre-chroot
* outer-post-chroot
* outer-chroot

The hooks *outer-pre-sourcer* are executed at the begin before
consumes the selected sourcer.
The hooks *outer-pre-chroot* are executed before executing commands
inside the chroot to prepare the target rootfs. For example, copy the
`meta-repo` inside the rootfs.
The hooks *outer-chroot* and *inner-chroot* are executed in the order
about are included in the job. The job can switch between commands
inside the chroot and commands out from chroot.
At the end, the hooks *outer-post-chroot* are hooks executed at the
end before the final tarball generation.

For all hooks the following variables are automatically added:

* MARKDEVKIT_WORKSPACE: defined with the job workspace directory
* MARKDEVKIT_ROOTFS: defined with the job rootfs directory

All job options if defined with int or string values are automatically
added as environment and could be used inside the hooks.

Every hook contains one or more command executed over the
selected *entrypoint* by default set to `/bin/bash  -c`.

Just a little example of the YAML:

```yaml
hooks:
- type: "inner-chroot"
  name: "my-first-hook"
  description: |
    This is my first hook
  entrypoint:
    - /bin/bash
    - -c
  commands:
    - >-
      echo "Helloworld!" &&
      echo "This is my hook!"
```

