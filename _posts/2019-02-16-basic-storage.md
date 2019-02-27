---
layout: post
title: Storage - Development Environment
subtitle: A simple storage solution for the home studio development environment.
---

In this brief update we will investigate how to set up a simple storage "service" that we can use to host content for the home studio project. Since our focus at this time will be on how to emulate a production storage solution within a development environment only, there's not much work to be done, so let's get stuck in!

# Overview

As we've previously discussed, it is a common practice in commercial vfx and animation studios to invest in a high-end storage system to host the vast quantities of digital content generated throughout the course of a show. Access to this storage system is normally provided to clients via [SMB](https://en.wikipedia.org/wiki/Server_Message_Block) or [NFS](https://en.wikipedia.org/wiki/Network_File_System) network mounts.

For simplicity, I've decided to create a single standard mount point to represent the home studio storage system. This mount point will be called *vfx* and will appear under the */vfx* file system location on each host.

The final production version of this storage volume will be hosted on a file server of some kind (e.g., a home NAS) and mounted on the client VMs over NFS. At this point in time however, all we need is a way to emulate the proposed storage infrastructure within a development environment, so we will host the storage locally for now. 

In an earlier [post](2018-03-24-soe-improvements.md), we looked at how to mount storage volumes in docker. We can make use of this approach to mount a test storage volume on the development machine under the */vfx* mount point within the docker SOE.

Vagrant also provides a feature called [synced folders](https://www.vagrantup.com/docs/synced-folders/) that can be used to replicate this behaviour on a virtual machine. In order to enable this feature, we will need to apply some updates to the *Vagrantfile*, which are covered in further detail below.

# Repository

As usual, I've committed the updates described in this article to the [vfx-studio-ops](https://github.com/l3wk/vfx-studio-ops) repository on GitHub. You can clone (or pull) from this repository in order to gain access to the latest changes if you wish.

# Vagrant Updates

Here are the necessary *Vagrantfile* modifications:

```ruby
    ...

    config.nfs.map_uid = Process.uid
    config.nfs.map_gid = Process.gid

    vfx_mount = ENV['VAGRANT_VFX_MOUNT']

    config.vm.synced_folder "#{vfx_mount}", "/vfx", type: "nfs"

    ...
```

These configuration changes define a new NFS synced folder to map the location specified within the *VAGRANT_VFX_MOUNT* environment variable on the development host under the */vfx* mount point inside the associated virtual machine.

When vagrant starts, it will automatically register a temporary export for the specified file system location on the development machine, which is then mounted over NFS by the virtual guest. Once the session terminates and the VM is destroyed, vagrant automatically removes the temporary export again as a part of the clean up process.

Note that we also configure the vagrant NFS client to map uid and gid to the corresponding values associated with the user account that has launched the underlying vagrant process. This will ensure that any files written into the mapped storage location are owned by the developer responsible for starting up the VM.

# Testing

With these updates in place, we are now able to create a test VM to try out the new mount point: 

```bash
$ ANSIBLE_ROOT=/home/l3wk/Workspace/vfx-studio-ops/ansible VAGRANT_VFX_MOUNT=/srv/nfs/vfx vagrant up
$ vagrant ssh
$ touch /vfx/test.txt
```

On the developer machine, under */srv/nfs/vfx* we should find the created test file:

```bash
$ ls /srv/nfs/vfx/*
$ test.txt
```

Finally, let's recap how to perform the equivalent operation when working with an instance of the docker SOE (replace `<uid>:<gid>` with the corresponding values from your own user account):

```bash
$ docker run -it --rm --mount type=bind,source="/srv/nfs/vfx",target=/vfx --user <uid>:<gid> l3wk/vfx-studio-soe:0.1.0
$ touch /vfx/test.txt
```

# What's Next?

In the next post, we'll take our simple storage management solution to the next level, and look at how to mount a storage volume on a more production-ready virtual machine. 
