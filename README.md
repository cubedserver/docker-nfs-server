# nfs-server

A handy NFS Server image comprising Alpine Linux and NFS v4 only, over TCP on port 2049.

## Overview

The image comprises of;

- [Alpine Linux](http://www.alpinelinux.org/). Alpine Linux is a security-oriented, lightweight Linux distribution based on [musl libc](https://www.musl-libc.org/) and [BusyBox](https://www.busybox.net/).
- NFS v4 only, over TCP on port 2049. Rpcbind is enabled for now to overcome a bug with slow startup, it shouldn't be required.

[Confd](https://www.confd.io/) is no longer used, making the image simpler & smaller and providing wider device compatibility.

For ARM versions, tag 6-arm is based on [hypriot/rpi-alpine](https://github.com/hypriot/rpi-alpine) and tag 7 onwards based on the stock Alpine image. Tag 7 uses confd v0.16.0.

When run, this container will make whatever directory is specified by the environment variable SHARED_DIRECTORY available to NFS v4 clients.

`docker run -d --name nfs --privileged -v /some/where/fileshare:/nfsshare -e SHARED_DIRECTORY=/nfsshare itsthenetwork/nfs-server-alpine:latest`

Add `--net=host` or `-p 2049:2049` to make the shares externally accessible via the host networking stack. This isn't necessary if using [Rancher](https://rancher.com/) or linking containers in some other way.

Adding `-e READ_ONLY` will cause the exports file to contain `ro` instead of `rw`, allowing only read access by clients.

Adding `-e SYNC=true` will cause the exports file to contain `sync` instead of `async`, enabling synchronous mode. Check the exports man page for more information: https://linux.die.net/man/5/exports.

Adding `-e PERMITTED="10.11.99.*"` will permit only hosts with an IP address starting 10.11.99 to mount the file share.

Due to the `fsid=0` parameter set in the **/etc/exports file**, there's no need to specify the folder name when mounting from a client. For example, this works fine even though the folder being mounted and shared is /nfsshare:

`sudo mount -v 10.11.12.101:/ /some/where/here`

To be a little more explicit:

`sudo mount -v -o vers=4,loud 10.11.12.101:/ /some/where/here`

To _unmount_:

`sudo umount /some/where/here`

The /etc/exports file contains these parameters unless modified by the environment variables listed above:

`*(rw,fsid=0,async,no_subtree_check,no_auth_nlm,insecure,no_root_squash)`

Note that the `showmount` command won't work against the server as rpcbind isn't running.

### Privileged Mode

You'll note above with the `docker run` command that privileged mode is required. Yes, this is a security risk but an unavoidable one it seems. You could try these instead: `--cap-add SYS_ADMIN --cap-add SETPCAP --security-opt=no-new-privileges` but I've not had any luck with them myself. You may fare better with your own combination of Docker and OS. The SYS_ADMIN capability is very, very broad in any case and almost as risky as privileged mode.

See the following sub-sections for information on doing the same in non-interactive environments.

### Other Operating Systems

You may need to ensure the **nfs** and **nfsd** kernel modules are loaded by running `modprobe nfs nfsd`.

### Mounting Within a Container

The container requires the SYS_ADMIN capability, or, less securely, to be run in privileged mode.

### Multiple Shares

This image can be used to export and share multiple directories with a little modification. Be aware that NFSv4 dictates that the additional shared directories are subdirectories of the root share specified by SHARED_DIRECTORY.

> Note its far easier to volume mount multiple directories as subdirectories of the root/first and share the root.

To share multiple directories you'll need to mount additional volumes and specify additional environment variables in your docker run command. Here's an example:
```
docker run -d --name nfs --privileged -v /some/where/fileshare:/nfsshare -v /some/where/else:/nfsshare/another -e SHARED_DIRECTORY=/nfsshare -e SHARED_DIRECTORY_2=/nfsshare/another itsthenetwork/nfs-server-alpine:latest
```

You should then modify the **nfsd.sh** file to process the extra environment variables and add entries to the exports file. I've already included a working example to get you started:

```
if [ ! -z "${SHARED_DIRECTORY_2}" ]; then
  echo "Writing SHARED_DIRECTORY_2 to /etc/exports file"
  echo "{{SHARED_DIRECTORY_2}} {{PERMITTED}}({{READ_ONLY}},{{SYNC}},no_subtree_check,no_auth_nlm,insecure,no_root_squash)" >> /etc/exports
  /bin/sed -i "s@{{SHARED_DIRECTORY_2}}@${SHARED_DIRECTORY_2}@g" /etc/exports
fi
```

You'll find you can now mount the root share as normal and the second shared directory will be available as a subdirectory. However, you should now be able to mount the second share directly too. In both cases you don't need to specify the root directory name with the mount commands. Using the `docker run` command above to start a container using this image, the two mount commands would be:

```
sudo mount -v 10.11.12.101:/ /mnt/one
sudo mount -v 10.11.12.101:/another /mnt/two
```

You might want to make the root share read only, or even make it inaccessible, to encourage users to only mount the correct, more specific shares directly. To do so you'll need to modify the exports file so the root share doesn't get configured based on the values assigned to the PERMITTED or SYNC environment variables.

## Credits
* https://github.com/sjiveson/nfs-server-alpine