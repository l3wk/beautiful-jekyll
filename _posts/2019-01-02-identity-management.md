---
layout: post
title: Identity Management
subtitle: Managing studio user accounts with JumpCloud.
---

It's now time to begin setting up some basic infrastructure services for the home studio project. In this post, we'll look at how to deploy an identity management solution to maintain the various different user accounts that our studio will require. We'll also be making some updates to the SOE provisioning scripts to integrate the new identity management service. Let's get started!

# Overview

One of the most foundational components of any IT infrastructure is a centralised user identity management service. The primary purpose of this service is to create and maintain user accounts and authenticate their associated security credentials.

While some excellent open source identity management systems are available ([FreeIPA](https://www.freeipa.org/) or [OpenLDAP](http://www.openldap.org/) for example), I was curious about the so called directory-as-a-service (or DaaS) offering from [JumpCloud](https://jumpcloud.com/), and decided that I would like to give it a try first.

JumpCloud is a commercial product, however a free, entry-level, tier is also available (thus making this service compatible with our project rules). The free tier permits up to ten individual user accounts, which should be more than adequate for our purposes.

# Requirements

The only requirement for the selected identity management scenario described in this post is a JumpCloud account. You can sign up for an account [online](https://jumpcloud.com/signup/). 

Further details about the capabilities of the JumpCloud DaaS product are available in the official [documentation](https://jumpcloud.com/daas-product/).

# Repository

I've committed each of the updates described in this article to the [vfx-studio-ops](https://github.com/l3wk/vfx-studio-ops) repository on GitHub. You can clone (or pull) from this repository in order to gain access to the latest changes if you wish.

*IMPORTANT NOTE: The secrets.yml file described in this article has not been included in the GitHub repository for security reasons. If you wish to make use of my example code, you'll need to add this missing file to your local copy as explained below.*

# JumpCloud Configuration

Once you've signed up and logged in to the administration console, select the *SYSTEMS* menu entry, press the green *+* button and note down the value of the *x-connect-key* shown in the install command under the *Linux* tab (see the JumpCloud agent deployment knowledge base [entry](https://support.jumpcloud.com/customer/en/portal/articles/2389062-agent-deployment) for further details). 

We'll use the *x-connect-key* later on to automate deployment of the client-side JumpCloud agent on any hosts where we want to be able to authenticate our users.

The second task that we need to perform at this point is to set up a test user account. We'll use this account later on to confirm that our hosts are able to interact with the JumpCloud directory service successfully.

I've called my test user *ops*, and for the required email address, I've used my regular email account with a *+ops* suffix after the user name element to allow me to filter any incoming messages into a separate folder within my mail client (see [here](https://gmail.googleblog.com/2008/03/2-hidden-ways-to-get-more-from-your.html) for further details).

# Ansible Updates

The client-side [JumpCloud Agent](https://support.jumpcloud.com/customer/en/portal/articles/2441017-how-the-jumpcloud-agent-works) provides a secure (HTTPS) connection to the cloud-based directory service and executes local user management operations in response to configuration updates applied within the JumpCloud administration console.

In order to authenticate JumpCloud user accounts, each host in our network will need to have a copy of the agent installed. We'll update our existing SOE provisioning scripts to automate this task.

## Agent Role

An [example](https://support.jumpcloud.com/customer/en/portal/articles/2443857--linux-installing-the-jumpcloud-agent-using-ansible) ansible role is provided by JumpCloud, which I've used as the basis for my own implementation:

```yaml
- include_vars: secrets.yml

- name: Install curl
  yum:
    name: curl
    state: installed

- name: Install ntpdate
  yum:
    name: ntpdate
    state: installed

- name: Check if JumpCloud agent is already installed
  shell: "[ -d /opt/jc ] && echo 'Found' || echo ''"
  register: jc_installed

- name: Update time
  shell: "ntpdate -u pool.ntp.org"
  when: "not jc_installed.stdout"

- name: Install JumpCloud agent
  shell: "curl --header 'x-connect-key: {{ jumpcloud_key }}' {{ jumpcloud_url }} | sudo bash"
  when: "not jc_installed.stdout"
```

As you can see, this new role requires two variables, one of which is a secret (*jumpcloud_key*) that will require some special handling. We'll look at how to securely control this configuration setting in greater detail in a moment. 

The second variable (*jumpcloud_url*) simply defines a download location for the agent installer, and will be defined within the standard *main.yml* configuration file found inside the *vars* sub-directory that accompanies the role definition:

```yaml
jumpcloud_url: "https://kickstart.jumpcloud.com/Kickstart"
```

Next, the *jumpcloud_key* variable that we need to keep secure is placed inside a separate *vars* file, which I've called *secrets.yml*. This configuration file will be encrypted using [Ansible Vault](https://docs.ansible.com/ansible/latest/user_guide/vault.html). Since Ansible will not automatically load this custom configuration file for us it needs to be referenced manually using the *include_vars* statement shown in the role definition above.

Here is the definition of *secrets.yml* (replace *secret_key* with the JumpCloud *x-connect-key* that you noted down while setting up your account):

```yaml
jumpcloud_key: "secret_key"
```

Finally, the last thing we need to do is make an update to the *vfx-studio-soe.yml* playbook to prevent our new role from being deployed inside docker (where we'll continue to make use of uid mapping for the time being instead):

```yaml
...
  roles:
    ...
    - role: jumpcloud-agent
      when: ansible_virtualization_type != "docker"
```

## Ansible Vault

Vault is a feature of ansible that supports the management of sensitive data (such as passwords or access keys) within encrypted files rather than plaintext playbook files or roles. The idea being that you can then check these encrypted files into a source code control system without revealing any secret information to outside observers.

Setting up ansible vault is a simple process:

 * Create a password file in your home directory (e.g., `touch ~/.vault_pass`) 
 * Set permissions on the file to ensure only you have access (e.g., `chmod 600 .vault_pass`) 
 * Add your desired vault encrypt/decrypt password (e.g., use `pwgen` to generate a random password)
 * Update your `.bash_profile` to export a new environment variable referencing the password file (i.e., `export ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_pass`)

We can now use the `ansible-vault` command to securely encrypt *secrets.yml* (note that there's no need to manually specify the `--ask-vault-pass` or `--vault-password-file` flags, as the password file will be automatically picked up from the environment):

```bash
ansible-vault encrypt secrets.yml
```

As noted earlier, even though I've encrypted my *secrets.yml* using ansible vault, I've decided not to go ahead and check the file into git along with the other resources from this post. The reason for this is that once the file is up on GitHub, it's pretty much there forever. This could prove problematic later on down the track, should a previously unknown security vulnerability be detected in vault (for example).

An even better solution for the long term would be to set up a local secrets server (such as [HashiCorp Vault](https://www.vaultproject.io/)) and use this to separate out any sensitive information from the provisioning scripts entirely (we might take a look at how to set this up in a future article, so stay tuned!).

# Vagrant Updates

Now that ansible vault is up and running, we can make some changes to the SOE *Vagrantfile* to allow ansible to automatically decrypt our secrets during the provisioning phase:

```ruby
    ...
    ansible_vault_password_file = ENV['ANSIBLE_VAULT_PASSWORD_FILE']
    ...
    config.vm.provision :ansible do |ansible|
        ...
        ansible.vault_password_file = "#{ansible_vault_password_file}"
        ...
    ...
```

We can now spin up a new virtual machine using the same command as last time (run `vagrant destroy` first, if necessary, in order to remove the previous image):

```bash
ANSIBLE_ROOT=/home/l3wk/Workspace/vfx-studio-ops/ansible vagrant up
```

Vagrant will launch the ansible provisioner as before, which will load up the new jumpcloud agent role we've defined and perform the required installation tasks (including automatically decrypting our secrets file when needed).

Once the provisioner has finished running, connect to the virtual machine and run `tail` on the JumpCloud agent log so we can monitor it for any problems:

```bash
$ vagrant ssh
Last login: Wed Jan  2 02:40:29 2019 from 192.168.121.1
[vagrant@localhost ~]$ sudo tail -f /var/log/jcagent.log
```

Next, log in to the JumpCloud administration console using your credentials and then check the *SYSTEMS* control panel to verify that an entry for your new vagrant test VM appears.

Select the entry for your host, and under the *Users* tab of the host settings dialog, find the entry for the *ops* user account you created earlier, and click the associated checkbox to bind the account to your test system. Once you've saved your changes, take a look at the JumpCloud agent log and verify that the updated configuration has been successfully propagated to your VM:

```bash
2019/01/02 03:04:16 [3224] [INFO] Processing user updates
2019/01/02 03:04:16 [3224] [INFO] Processing adding new users, addUsers=map[ops:0xc42028c000]
2019/01/02 03:04:16 [3224] [INFO] Adding new groups for username='ops'.
2019/01/02 03:04:16 [3224] [INFO] Sucessfully added new groups for username='ops'
2019/01/02 03:04:16 [3224] [INFO] Output data: [name=ops, ops]
2019/01/02 03:04:16 [3224] [INFO] Match found for event AddUser: Jan  2 03:04:16 localhost useradd[3325]: new user: name=ops, UID=1001, GID=1001, home=/home/ops, shell=/bin/bash
2019/01/02 03:04:16 [3224] [INFO] Revoking sudo permission for username=ops
2019/01/02 03:04:16 [3224] [INFO] Adding User (ops) to groups.
2019/01/02 03:04:16 [3224] [INFO] Removing User (ops) from groups.
2019/01/02 03:04:16 [3224] [INFO] Processing user takeovers, takeoverUsers=map[]
2019/01/02 03:04:16 [3224] [INFO] Processing already managed, updateAlreadyManaged=map[]
2019/01/02 03:04:16 [3224] [INFO] Processing disabling users, disableUsers=map[]
2019/01/02 03:04:16 [3224] [INFO] Processing md5sum watchers for /etc/passwd and /etc/shadow
2019/01/02 03:04:16 [3224] [INFO] Updating the PAM configuration
2019/01/02 03:04:16 [3224] [INFO] Restarting SSHD
2019/01/02 03:04:16 [3224] [INFO] Restarting SSHD via: systemctl restart sshd.service
2019/01/02 03:04:16 [3224] [INFO] Posting notification to server
2019/01/02 03:04:16 [3224] [INFO] User updates complete
2019/01/02 03:04:16 [3224] [INFO] Processing event AUTH_CONF_CHANGED with handler Lockout Handler
2019/01/02 03:04:16 [3224] [INFO] Finished processing event AUTH_CONF_CHANGED with handler Lockout Handler
2019/01/02 03:04:16 [3224] [INFO] Processing event AUTH_CONF_CHANGED with handler Event Logger
2019/01/02 03:04:16 [3224] [INFO] AuthConf Changed
2019/01/02 03:04:16 [3224] [INFO] Finished processing event AUTH_CONF_CHANGED with handler Event Logger
```

As a final test, run the `id` command to verify that the *ops* user is now recognized on your test VM:

```bash
[vagrant@localhost ~]$ id ops
uid=1001(ops) gid=1001(ops) groups=1001(ops)
[vagrant@localhost ~]$
```

Excellent! We now have a working identity management solution in place for the home studio!

# What's Next?

The next infrastructure service that we will need to consider is some kind of storage system, so that the other services we are planning to implement will have a place to write out data and configuration settings. We'll be investigating this topic further in the next post, so check back soon!

# References

 * Tristan Fisher - [Ansible Vault How-To](https://gist.github.com/tristanfisher/e5a306144a637dc739e7)
 * Ramon de la Fuente - [Use Ansible vault with Vagrant](https://coderwall.com/p/cew4vg/use-ansible-vault-with-vagrant)
 * Justin Ellingwood - [How To Use Vault To Protect Sensitive Ansible Data on Ubuntu 16.04](https://www.digitalocean.com/community/tutorials/how-to-use-vault-to-protect-sensitive-ansible-data-on-ubuntu-16-04)
