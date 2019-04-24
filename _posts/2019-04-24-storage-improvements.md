---
layout: post
title: Storage - Production Environment 
subtitle: Improved storage solution for the home studio production environment.
---

In the previous article, we looked at how to emulate a simple storage service within the studio development environment. This time around, we will take our setup to the next level, and implement support for a more production oriented storage solution. There's quite a lot of work to get through this time, so let's get started!

# Overview

Our ultimate goal for this next round of updates is to implement a new ansible role which we can use to automatically provision a production ready storage client for the home studio project.

While the implementation of the proposed storage client role is a simple enough task to complete, we will also need to perform a number of other tasks in order to set up some additional supporting infrastructure: 

 - Update the packer build script to generate a KVM version of the SOE
 - Implement a virtual studio deployment configuration using terraform
 - Enhance ansible provisioning scripts to support live infrastructure updates

By the time we reach the conclusion of this article, we'll have the means to automatically spin up and tear down infrastructure for a simple virtual studio environment - a platform which we'll continue to build upon in future posts.

# Requirements

Our work this time around requires a range of updates to the studio development environment. These updates include the installation of additional software packages, and a variety of configuration changes.

Firstly, lets take a look at the installation of some new development tools: 

 - [Terraform](https://www.terraform.io/): Provisioning tool for managing automated infrastructure deployments. 
 - [libvirt-terraform-provider](https://github.com/dmacvicar/terraform-provider-libvirt): Terraform provider to provision KVM infrastructure using libvirt.

The installation process for terraform is well [documented](https://learn.hashicorp.com/terraform/getting-started/install.html), and should be straightforward to set up, however the libvirt terraform provider will most likely need to be compiled and installed from source.

In this scenario, it may also prove necessary to download and install the latest release of [Go](https://golang.org/), along with the libvirt development headers, which should be available to install using the package manager for your OS.

I encountered a [bug](https://github.com/dmacvicar/terraform-provider-libvirt/issues/561) in the terraform-provider-libvirt master branch, which meant that a VM created with a non-base image volume (as used in the terraform configuration described further below) would fail to start. To get around this issue, I built the 0.5.1 release tag instead, since it was not affected by this bug.

We will also need to set up a more permanent nfs export on the development machine (or local storage server) for the studio clients to connect to. 

On my debian linux devlopment host, the steps required were as follows:

 - Install the `nfs-kernel-server` package using `apt-get`
 - Create a directory to serve as the mount point to export: `/srv/nfs/vfx`
 - Add an entry to the exports file to provide access to virtual hosts: `/srv/nfs/vfx  172.16.0.0/24(rw,sync,no_subtree_check)`
 - Restart the local nfs service to pick up the new export

Next, it will be necessary to set up a static [DHCP](https://en.wikipedia.org/wiki/Dynamic_Host_Configuration_Protocol) reservation to match the locally administered MAC address ([LAA](https://en.wikipedia.org/wiki/MAC_address#Universal_vs._local)) assigned to each of the virtual machine(s) used later on in this post. We use this approach so that it is possible to dynamically configure network interfaces on our VMs but still ensure that we can predict what address a given host will receive.

Lastly, you may also like to create a host entry for the machine in your local address file (or as a static DNS entry on your router, if it supports this feature). This will allow you to refer to your VM via host name rather than IP address.

# Repository

I've committed the updates described in this article to the [vfx-studio-ops](https://github.com/l3wk/vfx-studio-ops) repository on GitHub. You can clone (or pull) from this repository in order to gain access to the latest changes if you wish.

The branch containing updates from this post is: `20190305_storage_improvements`

# Packer Updates

Our first main task is to update the SOE packer build file to include support for a new image that we can use to spin up virtualised studio hosts under KVM.

We will start with a standard Centos 7 ISO and make use of a [kickstart](https://en.wikipedia.org/wiki/Kickstart_(Linux)) file (see *packer/vfx-studio-soe/http/ks.cfg* inside the repository) to automate the base install:

```
# System authorization information
auth --enableshadow --passalgo=sha512

# Use network installation
url --url="http://mirror.centos.org/centos/7/os/x86_64/"

# Use text mode install
text

# Run the Setup Agent on first boot
firstboot --enable
ignoredisk --only-use=vda

# Keyboard layouts
keyboard --vckeymap=us --xlayouts=''

# System language
lang en_US.UTF-8

# Network information
network  --bootproto=dhcp --device=eth0

# Root password
rootpw changeme 

# System services
services --enabled="chronyd,sshd"

# Do not configure the X Window System
skipx

# System timezone
timezone UTC 

# System bootloader configuration
bootloader --append=" crashkernel=auto" --location=mbr --boot-drive=vda
autopart --type=lvm

# Partition clearing information
clearpart --all --initlabel --drives=vda

# Reboot the system when the install is complete
reboot

%packages
@core
chrony
kexec-tools

%end

%addon com_redhat_kdump --enable --reserve-mb='auto'

%end

%post
yum -y install epel-release
yum -y install ansible
yum -y install sudo
yum -y upgrade
yum clean all

%end
```

The configuration settings contained within this file will instruct [Anaconda](https://en.wikipedia.org/wiki/Anaconda_(installer)) to set up a basic Centos 7 image, including system user accounts, core packages, and an SSH server. 

Packer will use SSH to connect to the host once the base install has finished in order to perform the various post-installation provisioning tasks.

With the kickstart file ready to go, we can now configure packer to automatically build a libvirt/kvm compatible edition of the SOE. We start, as usual, by declaring some additional variables:

```ruby
    ...
    "ansible_vault_file": "{{env `ANSIBLE_VAULT_PASSWORD_FILE`}}",
    ...
    "kvm_user": "",
    "kvm_pass": "",
    "kvm_ssh_key": "{{env `HOME`}}/.ssh/id_rsa.pub",
    "kvm_disk_size": "30000"
```

First, we declare a new variable to contain the path of the ansible vault password file (created in an earlier article), which will be uploaded to the image during the provisioning phase to ensure that ansible is able to decrypt any secrets required by the SOE roles.

Some default settings for the kvm builder are also defined, including a blank user name and password field (to be overridden at runtime), a public SSH key (used to set up password-less authentication for the *ops* user account), and a default volume size (in MB) to apply to the virtual disk which gets created for the new image. 

Next, we define a builder for the new image:

```ruby
    ...
    {
        "type": "qemu",
        "format": "qcow2",
        "accelerator": "kvm",
        "disk_size": "{{user `kvm_disk_size`}}",
        "iso_urls": [
            "CentOS-7-x86_64-Minimal-1810.iso",
            "http://mirror.aarnet.edu.au/pub/centos/7.6.1810/isos/x86_64/CentOS-7-x86_64-Minimal-1810.iso"
        ],
        "iso_checksum": "38d5d51d9d100fd73df031ffd6bd8b1297ce24660dc8c13a3b8b4534a4bd291c",
        "iso_checksum_type": "sha256",
        "http_directory": "http",
        "output_directory": "build",
        "ssh_username": "{{user `kvm_user`}}",
        "ssh_password": "{{user `kvm_pass`}}",
        "ssh_wait_timeout": "30m",
        "headless": false,
        "vm_name": "{{user `image_name`}}_{{user `image_version`}}.qcow2",
        "shutdown_command": "echo 'packer' | sudo -S shutdown -P now",
        "boot_wait": "10s",
        "boot_command": [
            "<tab> text ks=http://{{ .HTTPIP }}:{{ .HTTPPort }}/ks.cfg<enter><wait>"
        ]
     }
    ...
```

This builder will automatically download a copy of the Centos 7 minimal installer image, launch a local HTTP server to host the kickstart configuration file, and then execute the installer inside kvm, passing a reference to the URL of the kickstart configuration which will be used to customise the install. 

Lastly, once the base Centos 7 install completes, we will need to run a series of provisioning steps to install the SOE ansible roles, and set up authentication for the *ops* user account. This is done using the following sequence of packer provisioner configurations:

```ruby
    ...
    {
        "type": "file",
        "source": "{{user `kvm_ssh_key`}}",
        "destination": "/tmp/id_rsa.pub",
        "only": ["qemu"]
    },
    {
        "type": "shell",
        "inline": ["mkdir ~/.ssh; mv /tmp/id_rsa.pub ~/.ssh/id_rsa.pub"],
        "only": ["qemu"]
    },
    {
        "type": "file",
        "source": "{{user `ansible_vault_file`}}",
        "destination": "/tmp/.vault_pass",
        "only": ["qemu"]
    },
    {
        "type": "ansible-local",
        "playbook_file": "{{user `ansible_root`}}/playbooks/vfx-studio-soe.yml",
        "role_paths": [
            "{{user `ansible_root`}}/roles/bash",
            "{{user `ansible_root`}}/roles/jumpcloud-agent",
            "{{user `ansible_root`}}/roles/ops-user"
        ],
        "extra_arguments": [
            "--vault-password-file=/tmp/.vault_pass",
            "--extra-vars",
            "'image_name={{user `image_name`}} image_version={{user `image_version`}}'"
        ],
        "only": ["qemu"]
    },
    {
        "type": "shell",
        "inline": ["rm ~/.ssh/id_rsa.pub; rm /tmp/.vault_pass"],
        "only": ["qemu"]
    }
    ...
```

This series of packer provisioners will:

 - Upload the public SSH key file to `/tmp`.
 - Add the SSH key to the root user account.
 - Upload the ansible vault password file to `/tmp`.
 - Run ansible to provision the SOE, using the specified playbook and roles (which are automatically uploaded to the VM by packer).
 - Remove the SSH key file from the root user account, and delete the ansible vault password file once provisioning is complete.

When these provisioning steps are complete, packer will power down the VM instance that it has created for the install and write the image out to the output directory defined within the builder configuration.

One final change to note is that each of the provisioning steps have been labeled with the `only` keyword to ensure that they will be executed by the packer qemu builder, and not by the existing docker builder. 

It was also necessary to modify the other docker related rules in a similar way:

```ruby
    ...
    "only": ["docker"]
    ...
```

With all of these changes in place, we can now instruct packer to build the new image using the command shown below (launched within the packer/vfx-studio-soe directory):

```bash
packer build -only qemu -var "ansible_root=$PWD/../../ansible" -var "kvm_user=root" -var "kvm_pass=changeme" vfx-studio-soe.json
```

# Ansible Updates

The `ansible-local` provisioning step defined in the packer build file above relies on a new ansible role (*ops-user*), to create an *ops* user account within the kvm image. This is a management account that can be used to access and maintain any instances of the virtual machine that get deployed to a runtime environment.

Let's take a look at the definition of the *ops-user* role:

```yaml
- name: Create the user account
  user:
    name: ops
    shell: /bin/bash
    groups: wheel

- name: Add authorized ssh key
  authorized_key:
    user: ops
    key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
    exclusive: yes

- name: Enable sudo for the ops group 
  copy:
    content: "%ops ALL=(ALL) NOPASSWD: ALL"
    dest: /etc/sudoers.d/ops
    mode: 600
```

This new role will create the ops user account if it does not already exist, and will also inject the provided public key into the authorized keys file so that the user account may be accessed securely without requiring manual entry of a password. The *ops* user is also added to the sudoers file so that it is able to execute privileged system commands.

In addition to the creation of the *ops* user account, we also define a second new ansible role (*nfs-client*) which will be responsible for the installation and configuration of a production ready nfs client (i.e., the ultimate goal of this article):

```yaml
- name: Install nfs-utils
  yum:
    name: nfs-utils
    state: installed

- name: Create nfs mount point
  file:
    path: "{{ nfs_client_path }}"
    state: directory

- name: Add nfs export to /etc/fstab
  lineinfile:
    dest: /etc/fstab
    line: "{{ nfs_server_host }}:{{ nfs_server_path }} {{ nfs_client_path }} nfs"

- name: Mount nfs volume 
  mount:
    path: "{{ nfs_client_path }}"
    state: mounted 
```

This role will install the required software, create a directory to serve as a mount point for the remote filesystem, add a new entry to */etc/fstab*, and finally mount the nfs volume under the defined mount point location.

In addition to the *nfs-client* role definition, we also need to define some associated variables in the accompanying *vars/main.yaml* file as shown below:

```yaml
nfs_client_path: /vfx

nfs_server_host: vfx-nfs-001 
nfs_server_path: /srv/nfs/vfx
```

These settings are mostly self explanatory - take note however, that resolution of the *vfx-nfs-001* host name defined here relies on the static DNS entry I added to my router at the beginning of the article.

Now, since we don't necessarily intend to provide all of the hosts in our deployment with access to the production storage server (e.g., for security, or safety reasons), we will need a way to be able to control which hosts the nfs client role will be applied to. 

The most convenient solution in this situation is to make use of ansible inventories. In order to set this up, I've added a new staging inventory file to the repository, along with a custom playbook to control deployment of the storage client role defined above. 

Let's take a look at the inventory file first (*ansible/inventories/staging/hosts*):

```
[servers]
vfx-dev-001     ansible_user=ops    ansible_become=yes
```

This file defines a single host *vfx-dev-001* which ansible will connect to using the *ops* user account. The provisioner will be able to use the *ops* account to become the root user on this machine, due to the sudoers file entry we created previously.

Here are the contents of the new playbook (*ansible/playbooks/servers.yml*):

```yaml
- hosts: servers 
  roles:
    - nfs-client
```

This playbook instructs ansible to deploy the *nfs-client* role on any nodes that appear within the *servers* section of the associated hosts file.

# Terraform Configuration

The last major objective that we need to address is the implementation of some means to create instances of the kvm SOE image using libvirt. To perform this task, I'll be using terraform, which we installed on the development machine as described at the start of this guide. 

The following terraform configuration file (*terraform/studio.tf*), defines the infrastructure for a simple virtual studio, currently comprised of a single development server:

```
libvirt" {
    uri = "qemu:///system"
}

variable "image_root" {}

variable "image_version" {}

variable "dev_host_count" {
    default = 1
}

resource "libvirt_volume" "vfx-studio-soe" {
    name = "vfx-studio-soe_${var.image_version}.qcow2"
    source = "${var.image_root}/vfx-studio-soe_${var.image_version}.qcow2"
}

resource "libvirt_volume" "dev" {
    name = "vfx-dev-${format("%03d", count.index + 1)}.qcow2"
    base_volume_id = "${libvirt_volume.vfx-studio-soe.id}"
    count = "${var.dev_host_count}"
}

resource "libvirt_domain" "dev" {
    name = "vfx-dev-${format("%03d", count.index + 1)}"
    vcpu = 2
    memory = 1024

    disk {
        volume_id = "${element(libvirt_volume.dev.*.id, format("%03d", count.index + 1))}"
        scsi = "true"
    }

    network_interface {
        bridge = "br0"
        mac = "02:00:00:00:00:${format("%02X", count.index + 1)}"
    }

    count = "${var.dev_host_count}"
} 
```

This configuration makes use of the kvm image we created earlier to spin up the new development server instance, and assigns it a locally administered MAC address (`02:00:00:00:00:01`). The assigned MAC address will be picked up DHCP and matched against the IP address that has been reserved for the *vfx-dev-001* hostname.

Here are the terraform commands that we will use to create the virtual studio infrastructure defined above:

```bash
$ terraform init
$ terraform plan -var "image_root=$DEV_WORKSPACE/vfx-studio-ops/packer/vfx-studio-soe/build" -var "image_version=0.1.0" 
$ terraform apply -var "image_root=$DEV_WORKSPACE/vfx-studio-ops/packer/vfx-studio-soe/build" -var "image_version=0.1.0" 
```

And, here is the command that we need to tear the infrastructure down when we're done with our tests:

```bash
terraform destroy -var "image_root=$DEV_WORKSPACE/vfx-studio-ops/packer/vfx-studio-soe/build" -var "image_version=0.1.0"
```

# Testing

At last, we're finally ready to test everything out! Let's run through the entire process, from start to finish, step by step.

First, we run packer to build the new kvm SOE image:

```bash
packer build -only qemu -var "ansible_root=$PWD/../../ansible" -var "kvm_user=root" -var "kvm_pass=changeme" vfx-studio-soe.json
```

Next, we spin up an instance of the kvm SOE using terraform:

```bash
$ terraform init
$ terraform plan -var "image_root=$DEV_WORKSPACE/vfx-studio-ops/packer/vfx-studio-soe/build" -var "image_version=0.1.0" 
$ terraform apply -var "image_root=$DEV_WORKSPACE/vfx-studio-ops/packer/vfx-studio-soe/build" -var "image_version=0.1.0" 
```

When the development server is up and running, we run ansible to configure the nfs client on this new host:

```bash
ansible-playbook -i inventories/staging playbooks/servers.yml
```

Now, connect to *vfx-dev-001* using the *ops* user account and try out the nfs mount point:

```bash
ssh -l ops vfx-dev-001
ls /vfx
touch /vfx/test.txt
... # etc
```

When you're finished, exit out of the SSH session and then use terraform once again to tear down the test infrastucture:

```bash
terraform destroy -var "image_root=$DEV_WORKSPACE/vfx-studio-ops/packer/vfx-studio-soe/build" -var "image_version=0.1.0"
```

# What's Next?

In the next article, we'll continue to expand the capabilities of the terraform virtual studio that we introduced in this post, and attempt to deploy a docker swarm that can be used to host containerised services. Stay tuned!

# References

 * Velenux - [How to create a CentOS 7 KVM image with Packer](https://velenux.wordpress.com/2016/11/13/how-to-create-a-centos-7-kvm-image-with-packer/)
 * Server World - [Centos 7 : KVM : Create a Virtual Machine](https://www.server-world.info/en/note?os=CentOS_7&p=kvm&f=2)
 * Jeff Geerling - [Packer Example - CentOS 7 minimal Vagrant Box using Ansible provisioner](https://github.com/geerlingguy/packer-centos-7)
 * Giovanni Torres - [Create a Linux Lab on KVM Using Cloud Images](http://giovannitorres.me/create-a-linux-lab-on-kvm-using-cloud-images.html)
 * Fyodor - [Terraform provider for libvirt](https://hippo-toes.com/posts/terraform-provider-libvirt/)
