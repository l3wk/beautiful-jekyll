---
layout: post
title: SOE Improvements
subtitle: Extending the SOE using KVM, libvirt and Vagrant.
---

In this post, we'll continue on from where we left off last time, and make some further improvements to the studio SOE. Our primary goal will be to learn how to build and deploy the SOE as a full virtual machine, however we'll also take the time to explore a few features of docker that can be used to enhance the capabilities of the containerised SOE that we've already created. Let's get started!

# Overview

Since the previous post, I've had the opportunity to investigate some suitable virtualisation infrastructure options for the home studio project, and have resolved to make use of the Kernel-based Virtual Machine ([KVM](https://en.wikipedia.org/wiki/Kernel-based_Virtual_Machine)). Rather than working directly with KVM however, I've also decided to utilise an abstraction layer called [libvirt](https://en.wikipedia.org/wiki/Libvirt). The libvirt API is able to communicate with a number of different virtualisation products, so this choice gives me the flexibility to migrate to a different platform in future if necessary.

With these decisions made, it is now possible for me to go ahead and extend the suite of infrastructure tools with the ability to create instances of the SOE under KVM. Initially, I'll be making use of [Vagrant](https://en.wikipedia.org/wiki/Vagrant_(software)) to provide this capability. Vagrant is a software development tool which has been specifically designed to create and maintain temporary [VM](https://en.wikipedia.org/wiki/Virtual_machine) instances for the purposes of development, experimentation and testing.

# Requirements

Once again, it has proven necessary to add some additional software packages to the studio development environment:

 - [KVM](https://www.linux-kvm.org/): Kernel-based virtualisation platform for Linux.
 - [libvirt](https://libvirt.org/): API, daemon and management tool, supporting various different virtualisation platforms. 
 - [Vagrant](https://www.vagrantup.com/): A tool for building and maintaining portable virtual software development environments.
 - [vagrant-libvirt](https://github.com/vagrant-libvirt/vagrant-libvirt): A plugin that adds a libvirt provider to Vagrant.

More information (including tutorials and guides for getting started) is available for each of these tools via the provided links. Download and install each package using the instructions relevant to your chosen development environment OS. If KVM is not available in your environment, you may wish to consider the use of [VirtualBox](https://www.virtualbox.org/) instead.

When the packages have been installed, ensure that you take the time to configure a [bridge network](https://www.linux-kvm.org/page/Networking) for KVM, so that any virtual machines you create can be assigned IP addresses and made accessible to your local network.

# Repository

I've committed each of the updates described in this article to the [vfx-studio-ops](https://github.com/l3wk/vfx-studio-ops) repository on GitHub. You can clone (or pull) from this repository in order to gain access to the latest changes if you wish.

# Docker Exploration

Before we start working with Vagrant, libvirt and KVM, let's first take some time to further explore the capabilities of docker. In particular, let's take a look at some interesting "out of the box" features that we can use to immediately extend the capabilities of the containerised SOE we created in the previous post.

## Network Services

Just like a physical machine, a docker container can be assigned an IP address and communicate with other hosts over a network. Of particular interest to us however, are the DNS services that docker provides, since we'll ultimately want to be able to reference other systems in the studio network using their hostnames.

By default, each docker container will inherit the DNS settings of the underlying host where the docker daemon is running. The */etc/hosts* and */etc/resolv.conf* files on the host system are mapped into the container at run time, so any host aliases, or DNS resolver settings you've applied to your development system will be automatically made available to any SOE container instances that you start start up.

Here's a simple example showing this in action on the containerised SOE:

```bash
l3wk@milla:~$ docker run -it --rm l3wk/vfx-studio-soe:0.1.0
[vfx-studio-soe:0.1.0@242dcbaf4404$:/]# ping www.google.com
PING www.google.com (216.58.199.36) 56(84) bytes of data.
64 bytes from syd09s12-in-f4.1e100.net (216.58.199.36): icmp_seq=1 ttl=57 time=3.43 ms
64 bytes from syd09s12-in-f4.1e100.net (216.58.199.36): icmp_seq=2 ttl=57 time=4.12 ms
^C
--- www.google.com ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 3.438/3.783/4.129/0.350 ms
[vfx-studio-soe:0.1.0@242dcbaf4404$:/]# 
```

It's also possible to override this default behaviour on a per-container basis by passing in various different command line flags on startup.

Further details regarding these container networking features are [available](https://docs.docker.com/config/containers/container-networking/) in the official documentation online.

## Storage Volumes

One of the capabilities that we're going to need relatively soon, is the ability to attach external storage volumes to mount points within the containerised SOE. This functionality will allow us to share data between the different services and applications that the studio will need. 

Conveniently, docker includes support for three main types of storage mount: 

 - *volumes*: Persistent storage managed entirely by docker. 
 - *bind mounts*: Direct mappings into the filesystem structure of the docker host.
 - *tmpfs mounts*: Non-persistent storage volumes that exist entirely in memory (once again, on the docker host).

Each mount type has been designed to support a different stsorage management scenario, and it's quite likely that we'll end up making use of all three. We'll look at these in more detail later, however for now, here's an example that demonstrates how to create and use a bind mount on the SOE:

```bash
l3wk@milla:~$ mkdir tmp
l3wk@milla:~$ docker run -it --rm --mount type=bind,source="$(pwd)/tmp",target=/vfx l3wk/vfx-studio-soe:0.1.0
[vfx-studio-soe:0.1.0@9448841b0012$:/]# touch /vfx/test.txt
[vfx-studio-soe:0.1.0@9448841b0012$:/]# ls /vfx
test.txt
[vfx-studio-soe:0.1.0@9448841b0012$:/]# exit
exit
l3wk@milla:~$ ls ./tmp/
test.txt
l3wk@milla:~$
```

More information about docker [storage](https://docs.docker.com/storage/) features, including [volumes](https://docs.docker.com/storage/volumes/), [bind mounts](https://docs.docker.com/storage/bind-mounts/) and [tempfs mounts](https://docs.docker.com/storage/tmpfs/) is available online.

## UID/GID Controls 

In the previous example, you may have noticed that the test file we created was owned by the root user:

```bash
l3wk@milla:~$ ls -al ./tmp/test.txt
-rw-r--r-- 1 root root 0 Mar 16 19:53 ./tmp/test.txt
l3wk@milla:~$
```

We see this behaviour because the docker daemon runs under the root user account, and the processes running inside the SOE container are (currently) also executed as root. This highlights a security risk associated with the use of bind mounts (a container can manipulate the host file system), and is a key reason why the use of docker volumes is recommended whenever possible.

One mechanism that we can use to control the user and group that a container is executed under, is to specify the *--user* flag to *docker run* (populate `<uid>:<gid>` in the example below with the ids of your own username and group):

```bash
l3wk@milla:~$ docker run -it --rm --mount type=bind,source="$(pwd)/tmp",target=/vfx --user <uid>:<gid> l3wk/vfx-studio-soe:0.1.0
bash-4.2$ touch /vfx/test.txt
bash-4.2$ ls /vfx
test.txt
bash-4.2$ exit
exit
l3wk@milla:~$ ls -al ./tmp/test.txt
-rw-r--r-- 1 l3wk l3wk 0 Mar 19 21:22 ./tmp/test.txt
l3wk@milla:~$ 
```

We'll be investigating a more robust identity management strategy for the SOE in an upcoming post. In the meantime, as always, you can learn more about this topic [online](https://medium.com/@mccode/understanding-how-uid-and-gid-work-in-docker-containers-c37a01d01cf).

# Packer Template Updates

One important feature that is missing from our container SOE packer template (as introduced in the previous post) is the ability to automatically push a completed image to an external repository such as [Docker Hub](https://hub.docker.com/). 

In order to support this behaviour, I've extended the *post-processors* section of the template as shown below:

```json
{% raw %}
    "post-processors": [
        [
            ...
            {
                "type": "docker-push",
                "login": "true",
                "login_username": "{{user `docker_user`}}",
                "login_password": "{{user `docker_pass`}}"
            }
        ]
    ]
{% endraw %}
```

This new post processor template stanza relies on an additional packer variable, *docker_pass*, which I've also added to the variables section at the top of the template:

```json
{% raw %}
{
    "variables" {
        ...
        "docker_pass": "",
        ...
    }
}
{% endraw %}
```

The value of the docker password variable is empty by default, as I'll be passing this through to packer on the command line.

With these changes in place, it should now be possible to build the docker container image and push it to a remote repository using the following command (replace *l3wk* with the name of your own docker repository, and set the password accordingly):

```bash
packer build -var "ansible_root=$PWD/ansible" -var "docker_user=l3wk" -var "docker_pass=<password>" packer/vfx-studio-soe/vfx-studio-soe.json
```

# Vagrant File

At last, we're now ready to turn our attention towards the task of building a virtual machine image that is able to host the same SOE as the docker container that we've already built. 

Vagrant, the tool that we've chosen for this task, belongs to the same suite of software as packer, and supports a similar feature set. Vagrant differs from packer however, in that it is designed primarily to allow developers to easily create and teardown temporary virtual machine instances for the purposes of testing and experimentation. This capability is all that we will need for now, but at some point in the future, we'll want to come back and put together a packer template that we can also use to build and distribute a production version of the virtual machine. 

Here's the *VagrantFile* I've put together to build a fully virtualised instance of the SOE:

```ruby
VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

    ENV['VAGRANT_DEFAULT_PROVIDER'] = 'libvirt'

    config.vm.define :vfx_studio_soe do |vfx_studio_soe|
        vfx_studio_soe.vm.box = "centos/7"
    end

    ansible_root = ENV['ANSIBLE_ROOT']

    ENV['ANSIBLE_ROLES_PATH'] = "#{ansible_root}/roles"

    config.vm.provision :ansible do |ansible|
        ansible.playbook = "#{ansible_root}/playbooks/vfx-studio-soe.yml"
        ansible.sudo = true
        ansible.extra_vars = {
            image_name: "vfx-studio-soe",
            image_version: "0.1.0"
        }
    end
end
```

Notice that we're using the same ansible playbook that we used to provision the docker version of the SOE. This approach will help to ensure that regardless of whether we are working with a docker container or a fully virtualised host, the SOE image will be configured in exactly the same way.

Using the *vagrant* command, we can now launch a new instance of the SOE running in a virtual machine under KVM (inside the appropriate vagrant directory):

```bash
ANSIBLE_ROOT=/home/l3wk/Workspace/vfx-studio-ops/ansible vagrant up
```

# Ansible Updates

Although we're using the same ansible playbook, one additional step was added to the *bash* task, to ensure that there is a *.bashrc* file in the root user's home directory for us to update (this file is absent for some reason from the libvirt CentOS 7 base image, even though it is present in the Centos 7 base image for docker).

```yaml
- name: Ensure root user .bashrc exists
  file:
    path: /root/.bashrc
    state: touch
```

The ansible *file* operator will create the specified file if it is missing, or update the timestamp on the file if it already exists, in a similar way to the regular unix *touch* command.

# What's Next?

We now have enough basic functionality in place to start setting up some of the infrastructure services that the studio is going to need. As I've already hinted, in the next article we will investigate the topic of identity management.

# References

 * Kevin Cearns - [Packer and Docker Push](https://medium.com/@kcearns/packer-and-docker-push-9537b6c0ec89)
 * Jack Pearkes - [Packer, Vagrant and your Infrastructure - Fitting the Pieces Together](http://pretengineer.com/post/packer-vagrant-infra/)
 * Marc Cambell - [Understanding how uid and gid work in Docker containers](https://medium.com/@mccode/understanding-how-uid-and-gid-work-in-docker-containers-c37a01d01cf)
