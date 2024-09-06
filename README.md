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


## Run a job

Needed packages: `fchroot`, `ego`.

```shell
git clone https://github.com/macaroni-os/mark-stages.git

cd mark-stages

mark-devkit metro run --specfile ./jobs/stage3_x86_64.yaml --config ./etc/config.yml  \
    --job stage3-x86_64bit \
    --cleanup=false \
    --fchroot-debug
```

I suggest to developers to use `--cleanup=false` option and avoid to remove rootfs filesystem
at the end of the task.

These options are also helpful for devs:

* `--skip-source-phase`: If you have executed already one time the build and it's failed. You can
  skip the download and extract phase.

* `--skip-packer-phase`: This skip the final phase that create the stage3 tarball.

## Diagnose the rendered Job hooks

`mark-devkit` uses job's options to renderize the *hooks* files with the Helm Golang template engine.

This command permits to see the generated hooks:

```shell
mark-devkit diagnose job --job stage3-x86_64bit --specfile jobs/stage3_x86_64.yaml
```

## Different hooks types

The hooks could be divided into two families: *inner* and *outer*.
The *outer* are executed directly in the host; instead, the *inner*
are executed inside the rootfs of chroot through `fchroot`.

The hooks *outer* are then divided into:

* outer-pre-chroot
* outer-post-chroot
* outer-chroot

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
