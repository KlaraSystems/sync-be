# Klara sync-be

sync-be - create new boot environment from supplied image, and merge in
local configuration files from current boot environment.

## Synopsis

    sync-be <new bootenv> <config file>

sync-be is designed to leverage zfs boot environments and "golden images"
produced by [poudriere-image] or similar processes, to efficiently update
a given server to the new system image, whilst preserving relevant local
configuration files.

Across a fleet of servers, this can be used to ensure that all servers are
running a unified system image, whilst still allowing for local variations.

## Description

sync-be takes as input an exported zfs dataset, and a list of files and
directories.

It creates a new boot environment from the exported dataset, and then
copies in from the current boot environment, the files and directories
listed in the configuration file.

It is designed to be used with centrally built ZFS boot environments
across a fleet of machines, for example using [poudriere-image] and
[bectl].

## Options

- `new bootenv`: the name of the new boot environment to create
- `config file`: the path to the configuration file, see [syncbe.conf]

## Notes

This requires a standard FreeBSD system that matches a boot environment
zfs dataset layout, and a working boot environment already active.

The script does not update boot blocks, efi partitions, or similar early
boot functionality. It is assumed that the new boot environment is a
complete and bootable system image, and that the local configuration
files are the only changes required to make it operational.

## Pre-requisites

This is not a poudriere tutorial, but similar steps are required to create
a suitable image in the required `zfs+send+be` format:

```
# export IMAGE="$(date -u +%Y%m%d%H%M)"
# poudriere jail -c -j 13_2_builder_amd64 -v 13.2-RELEASE -K GENERIC
# poudriere bulk -j 13_2_builder_amd64 -f ./packages.lst
### create the boot environment
# poudriere image -t zfs+send+be -j 13_2_builder_amd64 \
-f ./packages.lst \
-s 4G \
-h '' \
-o /usr/local/poudriere/images/ \
-c overlay \
-n ${IMAGE}
# sha256 /usr/local/poudriere/images/${IMAGE}.be.zfs \
    | tee /usr/local/poudriere/images/${IMAGE}.be.zfs.sha256
```

The resulting image is placed in `/usr/local/poudriere/images/` which is
also accessible over http, for example https://pkg/images/ on the
poudriere build server.

## Files

- `sync-be`: the script itself
- `/etc/sync-be.conf`: user-provided configuration file, see [syncbe.conf]

## Exit Status

- returns 0 on success, non-zero on failure and prints an error message

## Example

Assuming a suitable image has been created with [poudriere-image], and
made available on a web server, the following usage replicates the
files and directories listed in `/etc/syncbe.conf` from the current
```
# bectl list
BE                             Active Mountpoint Space Created
13.1-RELEASE_2023-03-21_152313 -      -          836K  2023-03-21 15:23
default                        NR     /          2.24G 2023-03-21 13:49

# curl -#L https://pkg/images/be202303262144.be.zfs \
    | /usr/local/bin/sync-be 13.2-RELEASE /etc/syncbe.conf

using config file: /etc/syncbe.conf
receiving full stream of zroot.356600197/ROOT/default@202303262144 \
into zroot/ROOT/13.2-RELEASE@202303211523
###############                                          70.1%
...
received 1.77G stream in 35 seconds (51.7M/sec)
copying boot/loader.conf to /tmp/klara.QilKale4/boot/loader.conf
copying boot/loader.conf.d to /tmp/klara.QilKale4/boot/loader.conf.d
copying etc/login.conf.db to /tmp/klara.QilKale4/etc/login.conf.db
copying etc/pwd.db to /tmp/klara.QilKale4/etc/pwd.db
copying etc/spwd.db to /tmp/klara.QilKale4/etc/spwd.db
...
copying root to /tmp/klara.QilKale4/root
zfs bootenv is successfully written
ready for reboot!

# reboot
```

After reboot, when the system comes up, validate that expected services
are up and running. If there are issues, simply reboot and the system
automatically reverts to the previous boot environment, and you can try
again.

Once you are happy, mark the current BE as permanent:

```
# uname -a
FreeBSD example03 13.2-RELEASE FreeBSD 13.2-RELEASE releng/13.2-0386b9bd6 GENERIC amd64

# bectl list
BE                             Active Mountpoint Space Created
13.1-RELEASE_2023-03-21_152313 -      -          836K  2023-03-21 15:23
13.2-RELEASE                   N      /          1.79G 2023-03-26 21:54
default                        R      -          2.24G 2023-03-21 13:49

# bectl activate 13.2-RELEASE
```

[poudriere-image]: https://github.com/freebsd/poudriere/wiki/poudriere-image.8
[bectl]: https://man.freebsd.org/bectl
[syncbe.conf]: https://github.com/klarasystems/sync-be/blob/main/syncbe.conf
