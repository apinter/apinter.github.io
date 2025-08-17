---
title: "Podman checkpoint/restore on btrfs"
date: 2021-02-20T04:55:04+07:00
tags: ["Podman","container","migration"]
---

## Podman Checkpoint/Restore in Userspace

Recently I was migrating our entire Podman infra to a new, bigger server, roughly ~180 container. Most of our things are running in `pods` which are fairly easy to migrate, just `podman generate kube --filename xyz.yml xyz-pod`, clean up the yml a bit and on the new server I just *play* it with `podman play xyz.yml`. Done, pod migrated. 
On the other hand we have a few containers that are running outside of pods and was interested how lazy I can get with those using Podman. Fairly quickly ran into `Podman Checkpoints/Restore in Userspace (CRIU)`.
With CRIU Podman is able to checkpoint and restore containers in their **current state**. Migration is just one use-case, another would be to restore a container after a host reboot exactly the way it was prior the restart. This is exactly what I needed, scripts back in the box, in the new and shiny stuff comes.

## First run

Podman’s checkpoint/restore feature is currently only supports `rootfull` containers. I run openSUSE MicroOS on all of my servers running any kind of compute load which needs `btrfs`. I tried to run `podman container checkpoint --keep --leave-running --export=/root/test.tar.gz test` on a simple nginx test container - because I will not try anything new on production containers - which gave some pretty grim outputs:
```bash
ERRO[0000] read unixpacket @->@: EOF                    
Error: `/usr/bin/runc checkpoint --image-path /var/lib/containers/storage/btrfs-containers/c88b0634a239ad7547c2513b644b7b6d199a137012b0d8bbc10b96a1b712c0de/userdata/checkpoint --work-path /var/lib/containers/storage/btrfs-containers/c88b0634a239ad7547c2513b644b7b6d199a137012b0d8bbc10b96a1b712c0de/userdata --leave-running c88b0634a239ad7547c2513b644b7b6d199a137012b0d8bbc10b96a1b712c0de` failed: exit status 1
```

Tried to search for the error, but came up with nothing. Podman is fairly new and maybe not so well adopted yet so it might be difficult to find common issues on the web. Tried my luck on Github under the [project](https://github.com/containers/podman/), still found nothing much so went ahead and opened the issue. Got a response from Adrian Reber pretty quickly and we started to work our way through the issue. After a debug run, Adrian pointed out that the problem is probably caused by `btrfs`. Tried to run the same export on a VM running openSUSE Leap on `XFS`. It worked fine...

## Take two

After pushing things around for a while - and switching to `XFS` is not being an option - I had this idea to disable `copy-on-write (COW)` for the container storage folder - `/var/lib/container/` - and tried it again. Adrian also provided me with an excellent stateful test container that made a lot more sense to use over `nginx`, which is stateless, so the next round of testing began:

```bash
# chattr +C /var/lib/containers # disable COW for the container storage folder
# podman run -d quay.io/adrianreber/counter #The test container
# curl `podman inspect -l --format "{{.NetworkSettings.IPAddress}}"`:8088 # testing
counter: 0 
# curl `podman inspect -l --format "{{.NetworkSettings.IPAddress}}"`:8088
counter: 1
# curl `podman inspect -l --format "{{.NetworkSettings.IPAddress}}"`:8088
counter: 6
# curl `podman inspect -l --format "{{.NetworkSettings.IPAddress}}"`:8088
counter: 7
# curl `podman inspect -l --format "{{.NetworkSettings.IPAddress}}"`:8088
counter: 8
# curl `podman inspect -l --format "{{.NetworkSettings.IPAddress}}"`:8088
counter: 9
# curl `podman inspect -l --format "{{.NetworkSettings.IPAddress}}"`:8088
counter: 10
# curl `podman inspect -l --format "{{.NetworkSettings.IPAddress}}"`:8088
counter: 11
# podman container checkpoint --keep --export=./test11.tar.gz awesome_goldstine # **Checkpointing**
b9027b522b432e854307ee96358b9fc8e02f24cbd1b2c4a5b53a868be4d7db90
# podman ps
CONTAINER ID  IMAGE   COMMAND  CREATED  STATUS  PORTS   NAMES
# podman ps -a
CONTAINER ID  IMAGE                               COMMAND  CREATED        STATUS                    PORTS   NAMES
b9027b522b43  quay.io/adrianreber/counter:latest           4 minutes ago  Exited (0) 4 seconds ago          awesome_goldstine
# podman rm awesome_goldstine 
b9027b522b432e854307ee96358b9fc8e02f24cbd1b2c4a5b53a868be4d7db90
# l
total 6400
drwxr-xr-x 1 root    root       578 Feb 17 15:51 ./
drwxr-xr-x 1 root    root       114 Feb 17 15:40 ../
drwxr-xr-x 1 root    root         0 May 16  2020 GeoIP/
drwx------ 1 root    root         0 Jun 25  2020 NetworkManager/
drwxr-xr-x 1 root    root       418 Feb 17 15:41 YaST2/
drwxr-xr-x 1 root    root       394 Feb 17 15:34 alternatives/
drwxr-xr-x 1 root    root        10 Feb 17 15:33 apparmor/
drwxr-xr-x 1 root    root        16 Feb 17 15:35 autoinstall/
drwxr-xr-x 1 root    root        70 Feb 17 15:31 ca-certificates/
drwxr-x--- 1 chrony  chrony       0 Jun 10  2020 chrony/
drwx------ 1 root    root        30 Feb 17 15:47 cni/
drwx------ 1 root    root        24 Feb 17 15:46 containers/
drwxr-xr-x 1 root    root        20 Feb 17 15:33 dbus/
drwxr-xr-x 1 root    root        30 Feb 17 15:34 dhcp/
drwxr-xr-x 1 root    root        32 Feb 17 15:34 dhcp6/
drwx------ 1 root    root         8 Feb 17 15:40 ebtables/
drwxr-xr-x 1 root    root         0 Mar  7  2020 empty/
drwxr-xr-x 1 root    root        36 Feb 17 15:28 hardware/
drwxr-xr-x 1 root    root         0 Sep 21  2019 lifecycle/
drwxr-xr-x 1 root    root        38 Feb 17 15:37 misc/
drwxr-xr-x 1 root    root         0 May 17  2020 net-snmp/
drwxr-xr-x 1 root    root        56 Feb 17 15:34 nfs/
drwxr-xr-x 1 nobody  root         0 Jun  9  2020 nobody/
drwxr-xr-x 1 root    root        54 Feb 17 15:40 nscd/
drwxr-xr-x 1 root    root         0 May 16  2020 os-prober/
drwxr-xr-x 1 root    root        26 Feb 17 15:35 plymouth/
drwxr-xr-x 1 root    root         0 May 17  2020 polkit/
drwx------ 1 postfix root        22 Feb 17 15:40 postfix/
lrwxrwxrwx 1 root    root        26 Feb 17 15:35 rpm -> ../../usr/lib/sysimage/rpm/
drwxr-xr-x 1 root    root        68 Feb 17 15:35 samba/
drwxr-xr-x 1 root    root        22 Feb 17 15:33 smartmontools/
drwxr-xr-x 1 root    root         0 Jun  9  2020 sshd/
drwx--x--x 1 root    root        20 Feb 17 15:41 sudo/
drwxr-xr-x 1 root    root       104 Feb 17 15:40 systemd/
-rw------- 1 root    root   2181425 Feb 17 15:49 test.tar.gz
-rw------- 1 root    root   2181134 Feb 17 15:51 test11.tar.gz
-rw------- 1 root    root   2181397 Feb 17 15:50 test2.tar.gz
drwxr-xr-x 1 root    root         0 May 17  2020 usb_modeswitch/
drwxr-xr-x 1 root    root         0 Jun  6  2020 vmware/
drwxr-x--- 1 root    root        80 Feb 17 15:50 wicked/
drwxr-xr-x 1 root    root       136 Feb 17 15:43 zypp/
# podman container restore -i test11.tar.gz # restore on another MicroOS server
b9027b522b432e854307ee96358b9fc8e02f24cbd1b2c4a5b53a868be4d7db90
# curl `podman inspect -l --format "{{.NetworkSettings.IPAddress}}"`:8088 # Final test
counter: 12
```

And there I was with a functional checkpoint and restore feature on my openSUSE servers with `btrfs`. Great feeling to get something running that wasn't out-of-the-box and share the solution upstream. But then again I was reminded that it only supports rootfull use-cases so went back using my scripts -_-.

#### References:
* [The issue](https://github.com/containers/podman/issues/9318)
* [CRIU docs](https://criu.org/Filesystems_pecularities#BTRFS_Workaround)
