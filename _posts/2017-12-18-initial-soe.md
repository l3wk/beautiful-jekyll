---
layout: post
title: Initial SOE
subtitle: How to create a base studio image using Packer, Ansible and Docker.
---

Now that our basic development environment is ready, we can start doing some work! In this post, we will investigate how to go about setting up an initial version of the standard operating environment, or SOE, for the home vfx studio. The resulting OS image will be installed on any hosts that we spin up to run studio services and applications, providing a stable and consistent platform for their execution.

# Overview

Although I've not yet decided what virtualisation platform I intend to use to host the studio servers, I do know that I will have a variety of applications and services that I will want to run within a containerised environment. This means that I can start work on setting up the SOE inside docker. By using a provisioning system to manage the configuration and setup of the OS, it should be simple to set up an identical image when the time comes to create a fully virtualised host. 

The SOE we create in this guide will be based on [CentOS](https://www.centos.org/), a Linux distribution that is popular within the vfx community at this point in time. The current release is CentOS 7, which should be compatible with the [VFX Reference Platform](http://www.vfxplatform.com/) guidelines for 2018, so we'll go ahead and target this edition for our SOE build.

# Requirements 

In order to build the SOE image, it will first be necessary to add some additional software packages to the development environment we set up in the previous post. Here is a list of the tools we will need:

 - [Packer](https://www.packer.io/) (a tool for building VM images for different platforms using a configuration script)
 - [Ansible](https://www.ansible.com/) (OS provisioning system which packer can invoke to customise a machine image) 
 - [Docker](https://www.docker.com/) (multi-platform container runtime environment)

Download and install these tools using the OS specific instructions relevant to your development environment. Learning resources, such as documentation and tutorials, are available on the websites linked above for each package. If you are unfamiliar with any of these software packages, take some time to read through the documentation and try out some of the tutorials in order to master the basics before you proceed.

# Repository 

I've created the [vfx-studio-ops](https://github.com/l3wk/vfx-studio-ops) GitHub repository to host the software I develop to provision and manage the infrastructure for my home studio. This will also include the resources needed to recreate the SOE described in this guide. You can create a clone or fork of this repository if you wish to duplicate my efforts, otherwise, have a go at setting up an infrastructure repository of your own.

# Packer Template 

I'll be using packer to automate builds of the machine images that I need for my home studio. I've chosen packer because it is a modern, script based, automated machine image build tool with support for a wide variety of virtualisation platforms and provisioning systems. This should give me the flexibility I desire in order to be able to experiment and switch out different components of the underlying studio infrastructure as requirements change over time.

Here is the packer template I've put together as a starting point for the base SOE:

```json
{
    "variables": {
        "ansible_root": "",
        "ansible_host": "default",
        "ansible_connection": "docker",
        "docker_user": "",
        "image_name": "vfx-studio-soe",
        "image_version": "0.1.0"
    },
    "builders": [
        {
            "type": "docker",
            "image": "centos:7",
            "commit": "true",
            "run_command": [
                "-d",
                "-i",
                "-t",
                "--name",
                "{{user `ansible_host`}}",
                "{{.Image}}",
                "/bin/bash"
            ]
        }
    ],
    "provisioners": [
        {
            "type": "ansible",
            "user": "root",
            "playbook_file": "{{user `ansible_root`}}/playbooks/vfx-studio-soe.yml",
            "ansible_env_vars": [
                "ANSIBLE_ROLES_PATH={{user `ansible_root`}}/roles:$ANSIBLE_ROLES_PATH"
            ],
            "extra_arguments": [
                "--extra-vars",
                "ansible_host={{user `ansible_host`}} ansible_connection={{user `ansible_connection`}} image_name={{user `image_name`}} image_version={{user `image_version`}}"
            ]
        }
    ],
    "post-processors": [
        {
            "type": "docker-tag",
            "repository": "{{user `docker_user`}}/{{user `image_name`}}",
            "tag": "{{user `image_version`}}"
        }
    ]
}
```

This template uses the docker builder included with packer to create a new docker image using the CentOS 7 base [image](https://hub.docker.com/_/centos/) available on the [Docker Hub](https://hub.docker.com/). The build script will automatically launch the ansible provisioning system to customise the container image. Once the container is built, a post processor will execute in order to tag the new image within an appropriately named docker repository.

# Ansible Role

For this initial setup, I've created a single ansible role which will customise the bash prompt of the root user within the SOE image:

```yaml
- name: Customise bash prompt for root user
  lineinfile:
    dest: /root/.bashrc
    line: "{{ ps1 }}"
    owner: root
```

The actual prompt definition is retrieved from a configuration setting which is included in the variables file for the role. The definition itself makes use of the image name and version values passed through to the ansible provisioner from the packer template:

```yaml
ps1: 'PS1="\[$(tput setaf 3)$(tput bold)[\]{{ image_name }}:{{ image_version }}@\\h$:\\w]#\[$(tput sgr0) \]"'
```

This will ensure that the bash prompt within the generated machine image includes the name and version number of the build, and that it is displayed in a custom colour (yellow) to differentiate the session from a regular bash session on my workstation (where the prompt is configured to display in green).

# Build & Run

With the SOE packer template and ansible provisioning scripts in place, it should now be possible to build and run the resulting container image. Here's the environment I'm working in for the examples shown in this section of the guide:

```bash
l3wk@milla:~$ packer --version
0.10.2
l3wk@milla:~$ ansible --version
ansible 2.2.1.0
  config file = /etc/ansible/ansible.cfg
  configured module search path = Default w/o overrides
l3wk@milla:~$ docker --version
Docker version 17.05.0-ce, build 89658be
l3wk@milla:~$ pwd
/home/l3wk/Workspace/vfx-studio-ops
l3wk@milla:~$
```

From within the *vfx-studio-ops* repository clone, you should be able to build the docker image using the following command (be sure to replace *l3wk* with the name of your own docker repository root):

```bash
packer build -var "ansible_root=$PWD/ansible" -var "docker_user=l3wk" packer/vfx-studio-soe/vfx-studio-soe.json
```

Take note of how we've set the *ansible_root* variable to allow us to pass through the location of the ansible scripts to packer, and the *docker_user* variable to specify the docker repository root. We reference these variables within the provisioner configuration inside the packer template rather than setting hard-coded values. This helps to keep our build scripts clean and flexible.

Once the container builds successfully, you can try it out using the docker command shown below:

```bash
docker run -it --rm l3wk/vfx-studio-soe:0.1.0
```

If everything went according to plan, you should now see something similar to this example output from my own development environment:

```bash
l3wk@milla:~$ docker run -it --rm l3wk/vfx-studio-soe:0.1.0
[vfx-studio-soe:0.1.0@8564ee62f5fa$:/]# ls
anaconda-post.log  dev  home  lib64  mnt  packer-files  root  sbin  sys  usr
bin                etc  lib   media  opt  proc          run   srv   tmp  var
[vfx-studio-soe:0.1.0@8564ee62f5fa$:/]# 
```

# What's Next? 

Excellent! We now have a simple SOE which we can use as a foundation for further development. In the next article, we will extend this basic environment with some additional functionality that we will need.

# References

 * James Carr - [Build Docker Images with Packer and Ansible](https://blog.james-carr.org/build-docker-images-with-packer-and-ansible-3f40b734ef4f)
 * Chad Mayfield - [Docker usability, change the prompt](https://chadmayfield.com/2016/06/17/docker-prompt-change/)


