---
layout: post
title:  'Vagrant – automating environment deployment'
description: 'A tutorial on how to use Vagrant to simplify the deployment of virtual machine environments, for use in software development, testing, or plain sandboxing.'
author: 'Alexandre Constantino'
date:   2016-01-28
categories: devops
tags:       tutorial vagrant
---


# About

This is a guide on Vagrant which covers most of what there is to known about it. If you've found other resources to be confusing, not going through all the steps, or not able to demystify how things work; then you may find this tutorial helpful.
In any case, if you just need a cheat sheet then jump into the [Basics section](#basics).


# What is Vagrant

Vagrant is a tool for deploying software environments in a configurable and reproducible form, from a single source configuration file. Whereas a software environment consists of one or more virtual machine or container instances; and most likely with network port forwarding and folder shares configured as well.
Its typical use case is the deployment of development and test environments. Because the VM deployment is reproducible it makes possible to have the same exact environment across multiple machines, thus eliminating the problems caused by differences in system configuration both across teams and infrastructure.
With a single command, `vagrant up`, it becomes possible to deploy a whole environment in just a few minutes.
Supported providers (ie: virtualization and containerization platforms) include: VirtualBox, VMWare, HyperV, Docker, KVM, AWS and others.
The provisioning (ie: installation and configuration of software into the instantiated guest system) can be done using one or more of the supported configuration management tools: shell scripts, Ansible, CFEngine, Chef, Docker, Puppet, Salt.
The Vagrant VM images are commonly referred as *Vagrant boxes* and they are provider specific. While an instantiated VM is referred as *development environment*.

**Vagrant recipes used in this tutorial:**
<https://github.com/alexconst/vagrant_recipes>



# Understanding the magic

Vagrant acts as a wrapper around different providers (eg: VirtualBox) and provisioning tools (eg: [Ansible](ansible.md)), making the whole process:
- easy: with just a couple of commands you can ssh into a new development environment;
- simple: no need to configure any settings from each virtualization software;
- uniform: it works the same way across multiple cloud providers and virtualization platforms (assuming Vagrant boxes are available for those providers); and
- reproducible: you always get the same development environment when you start a new development environment.

The Vagrant boxes are created using [Packer](packer.md). And these consist on a tar file which include the virtualization software VM files (eg: .ovf, .vmdk) and a couple of Vagrant specific files.
The configuration of a development environment is done via a Vagrantfile. It includes settings for box selection, network port forwarding, shared folders, credentials, provisioning scripts, and other settings; for one or more machine instances.
When a new environment is created using `vagrant up` Vagrant loads a series of Vagrant files and merges and overrides the settings as it goes. The order being: the Vagrantfile packaged in the box, the Vagrantfile at `~/.vagrant.d/`, the Vagrantfile for the current environment (this is the one that we'll change most of the time), multi-machine overrides, and provider specific overrides.
Vagrant boxes are installed to `~/.vagrant.d/boxes/` while environments are created at each provider's own default folder.
Vagrant then acts as a wrapper to the different virtualization software simplifying the whole process. For example, when `vagrant up` is executed it (if required) downloads a VM image, unpacks it, starts the VM, installs and configures software packages, and sets up the network and shared folders.
This means that with a Vagrantfile and the corresponding provisioning scripts it becomes possible to define a whole development environment consisting of one or more machines. With the advantage that the whole configuration can be easily shared and put under version control.

**Security concerns**
Vagrant development environments are insecure by default and by design. Since the typical use case is the local computer, Vagrant development environments exchange security for streamlined and easy access instead. If the Vagrant development environment is exposed to the outside world then:
- create your own box,
- with private credentials,
- use secure SSH keypairs, and
- disable password login for all users.



# Installation and plugins

Download and install Vagrant from <https://www.vagrantup.com/downloads.html>.
Source code can be found at <https://github.com/mitchellh/vagrant>.

Vagrant ships with provider support for VirtualBox, Docker and HyperV.
Support for other providers can be done via plugins:
- VMware Workstation/Fusion: developed and supported by the authors of Vagrant and it costs $79. <https://www.vagrantup.com/vmware>
- VMware Workstation/Fusion: a free plugin which according to the author is not mature enough to be called alpha. And apparently no longer works[^broken_vmware_free]. <https://github.com/orishavit/vagrant-vmware-free>
- KVM and QEMU: <https://github.com/pradels/vagrant-libvirt>
- AWS cloud: <https://github.com/mitchellh/vagrant-aws>
- Google cloud: <https://github.com/mitchellh/vagrant-google>
- Rackspace cloud: <https://github.com/mitchellh/vagrant-rackspace>
- VMware vSphere: <https://github.com/nsidc/vagrant-vsphere>
- VMware vCenter: the project looks dead <https://github.com/frapposelli/vagrant-vcenter>

Other interesting plugins:
- vbguest: automatically keep the VirtualBox guest tools updated <https://github.com/dotless-de/vagrant-vbguest/>
- mutate: convert boxes to work with different providers <https://github.com/sciurus/vagrant-mutate>
- triggers: allow the definition of arbitrary scripts that will run on the host or guest before and/or after Vagrant commands <https://github.com/emyl/vagrant-triggers> ([related discussion at SO](http://stackoverflow.com/questions/21476713/is-there-any-hook-like-pre-vagrant-up))

[^broken_vmware_free]: <https://github.com/mitchellh/vagrant/issues/6935>

List of plugins: <http://vagrant-lists.github.io/>

Installing plugins:
```bash
vagrant plugin install vagrant-libvirt
vagrant plugin install vagrant-vmware-free
vagrant plugin install vagrant-aws
vagrant plugin list
vagrant plugin update
```
Plugins are downloaded from the ruby gems repository and are installed to `$HOME/.vagrant.d/gems/gems/`.




# Using Vagrant


## Basics

This section is a bit of a cheat sheet and characterizes a typical Vagrant workflow.

```bash
# check version
vagrant version

# select a box from Hashicorp's Atlas, eg: ubuntu/wily64
# FYI ubuntu/wily64 comes with VirtualBox guest tools installed,
# while debian/jessie64 instead uses rsync (which doesn't work so well)
boxname="ubuntu/wily64"

# if the box is not present in your box pool, then add it
vagrant box add $boxname
# check that it has been added
vagrant box list

# generate a Vagrantfile set to use given box
vagrant init $boxname

# deploy the vagrant box (ie: download if needed, unpack, and instantiate the box)
vagrant up
# if you have multiple providers for the same box then you'll need to specify which
# with $provider being one of 'virtualbox', 'vmware', 'aws', 'qemu', etc
vagrant up --provider $provider

# package the running VirtualBox environment into a re-usable box
# (tip: can be useful when looking to save provisioning time)
vagrant package --output "${newbox}.box"

# connect to the development environment
vagrant ssh

# get the status of existing environments for the local Vagrantfile
vagrant status

# get the status for all environments for all Vagrantfiles
# to make sure results are accurate add the --prune flag
# however it will take a lot! longer to show results
vagrant global-status

# show existing network port mappings
vagrant port

# suspend the development environment
vagrant suspend

# resume a previously suspended environment
vagrant resume

# delete the running development environment (use -f for no confirmation)
vagrant destroy

# remove a box
vagrant box remove $boxname

# get a list of available boxes
vagrant box list
```




## Snapshots

Taking snapshots is simple, but there are a few caveats.
If you use `push` and `pop` it is not recommended to mix them with `save` and `restore` and `delete`.
Also some providers (VirtualBox included) require that when `delete` is used that all subsequent snapshots are deleted in the reverse order that they were taken.
So when using snapshots choose one and only one of the options:
```bash
# list existing snapshots
vagrant snapshot list

# option 1:
# take snapshot and push it onto the snapshot stack
vagrant snapshot push
# restore a snapshot and remove it from the snapshot stack
vagrant snapshot pop

# option 2:
# take a snapshot
vagrant snapshot save $snap_name
# restore a snapshot
vagrant snapshot restore $snap_name
# delete a snapshot
vagrant snapshot delete $snap_name
```




## Your own environment

This section details how to add a local Vagrant box (that was created with [Packer](packer.md)) and how to set up a development environment using a Vagrantfile. It goes through the most commonly used Vagrantfile settings.

Add a local Vagrant box to the pool of available boxes:
```bash
# select the box
boxfile="debian-802-jessie-amd64_jessie-vboxiso.box"
boxname="alexconst/debian/jessie64/20160115.0.3"
# add the box
vagrant box add  "${boxfile}"  --name "${boxname}"
# check that it has been added
vagrant box list
```
Note that there is a `--box-version` option to the `box add` command but that is only for remote clouds (eg: Hashicorp Atlas), so we workaround that limitation by adding the version tag to the box name. This "workaround" does give us visual identification, but doesn't really provide the version constraint features of that command option.


Create a blank Vagrantfile:
```bash
vagrant init
```

Then to configure a new environment, that uses the new box, edit the Vagrantfile as follows:
```ruby
# select our box
config.vm.box = "alexconst/debian/jessie64/20160115.0.3"

# set credentials (only required if Vagrant's insecure keypair wasn't added during box creation)
config.ssh.username = "vagrant"
config.ssh.password = "vagrant"

# set hostname
config.vm.hostname = "vagrant-devbox"

# set text to be shown after 'vagrant up'
# this text can be useful for detailing instructions how to access the development environment
config.vm.post_up_message = "A Vagrant tutorial can be found at http://alexconst.github.io/"

# configure an additional shared folder
# by default the project dir is already shared at /vagrant/
host_folder = ENV['HOME'] + "/shared_vagrant"
guest_folder = "/shared/"
config.vm.synced_folder host_folder, guest_folder

# configure a private network using DHCP
# this will allows the VM to be accessible by the host and other guests within the same network
config.vm.network "private_network", type: "dhcp"

# alternatively it can also be configured to use a static IP, for example:
#config.vm.network "private_network", ip: "192.168.22.50"
# but make sure the 192.168.22.x network is not being already used (if so choose
# another network) and also make sure you don't set at IP ending in ".1" since
# that can cause problems

# configure the network: set up port forwarding for TCP
# accessing port 8080 on the host will be forwarded to port 80 on the guest
config.vm.network "forwarded_port", host: 8080, guest: 80

# configure the network: set up port forwarding for both UDP and TCP
config.vm.network "forwarded_port", host: 12003, guest: 2003, protocol: "tcp"
config.vm.network "forwarded_port", host: 12003, guest: 2003, protocol: "udp"

# provision by copying a file or directory to the guest machine
config.vm.provision "file", source: "files/mydata.tar.gz", destination: "/tmp/mydata.tar.gz"
# provision by executing a command on the guest
config.vm.provision "shell", inline: "tar -xzf /tmp/mydata.tar.gz"
# provision by uploading a script and executing it (path can be an URL)
config.vm.provision "shell", path: "scripts/packages.sh"
```

Our use case matches the default scenario where the guest is Linux and communication is done via `ssh`. But Vagrant also supports Windows guest where communication is done via `winrm`.
Vagrant also supports synced folders via different implementations: NFS (Linux only), Samba (Windows only), rsync, and the provider's own mechanism (eg: VirtualBox shared folders). However these are not perfect; do check the [Known issues](#known-issues) section for more information.
Regarding rsync, it does a one-time one-way (from host to guest) folder synchronization on `vagrant up`, `vagrant reload` and `vagrant rsync`. There is also `vagrant rsync-auto` which runs in the foreground listening to events on the local filesystem before it does a folder sync; however this command has a few caveats.


To instantiate a new environment using our box and connect to it:
```bash
vagrant up
vagrant ssh
```




## DevEnv 1: Blog in a box

Here we'll see how to set up a development environment ready for blogging by installing the GitHub Pages gem (which uses a specific version of Jekyll).
This way we'll have an environment that matches the GitHub Pages website blogging backend and so we'll be able to preview our posts just as they'll be shown on the GitHub website.

The Vagrantfile:
```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|

  # Choose a box with VBox guest tools already installed and a Ruby version
  # compatible with GitHub Pages and Jekyll.
  config.vm.box = "ubuntu/wily64"

  # Set up hostname
  config.vm.hostname = "jekyll"

  # Message shown on vagrant up
  config.vm.post_up_message = "
Blog in a box! GitHub Pages (Jekyll powered) compatible environment.
Simply cd to the blog directory and type:
    pkill jekyll ; jekyll build  && jekyll serve

A Vagrant tutorial can be found at http://alexconst.github.io/"


  # Create a forwarded port mapping for Jekyll
  config.vm.network "forwarded_port", guest: 4000, host: 4000

  # Share an additional folder with the guest VM.
  # This folder should have our blog files.
  host_folder = ENV['HOME'] + "/home/downloads/share_vagrant"
  guest_folder = "/shared/"
  config.vm.synced_folder host_folder, guest_folder

  # Fine tune the virtualbox VM
  # We could get away with 256MB if Chef and Puppet agents weren't installed,
  # which eat a total of 100 MiB of RAM (~ 50 MiB each). Update: no we wouldn't
  # because 384 MB may not be enough for building the GitHub pages gem. It may
  # complain with "ERROR: Failed to build gem native extension. ... Cannot allocate memory"
  # Set 2 virtual CPUs, each can use up to 50% of a single host CPU.
  config.vm.provider "virtualbox" do |vb|
    vb.customize [
      "modifyvm", :id,
      "--cpus", "2",
      "--cpuexecutioncap", "50",
      "--memory", "512",
    ]
  end

  # fix annoyance, http://foo-o-rama.com/vagrant--stdin-is-not-a-tty--fix.html
  config.vm.provision "fix-no-tty", type: "shell" do |s|
    s.privileged = false
    s.inline = "sudo sed -i '/tty/!s/mesg n/tty -s \\&\\& mesg n/' /root/.profile"
  end
  # fix annoyance, http://serverfault.com/questions/500764/dpkg-reconfigure-unable-to-re-open-stdin-no-file-or-directory
  config.vm.provision "shell", inline: "echo 'export DEBIAN_FRONTEND=noninteractive' >> /root/.profile"
  config.vm.provision "shell", inline: "for user in /home/*; do echo 'export DEBIAN_FRONTEND=noninteractive' >> $user/.profile; done"


  # provisioning using shell scripts
  config.vm.provision "shell", path: "scripts/jekyll.sh"

end
```

And the provisioning script:
```bash
#!/bin/bash

apt-get update
# install htop because we can
apt-get install -y htop
# dependencies for github-pages
apt-get install -y ruby-dev bundler zlib1g-dev ruby-execjs --no-install-recommends
# dependencies for the pygments syntax highlighter
apt-get install -y python-pygments
# There is a bug which forces us to explicitly install activesupport, which
# would otherwise be done by the github-pages gem.
# https://github.com/github/pages-gem/issues/181
gem install activesupport
gem install github-pages
apt-get clean
```



To get jekyll running:
```bash
# deploy and connect to the environment
vagrant up
vagrant ssh

# if you don't have a site yet:
cd /shared/
jekyll new myblog
# if you already have one then:
cd $to_directory_with_your_blog

# start jekyll webserver
pkill jekyll ; jekyll build  && jekyll serve
```



## DevEnv 1.1: Using Ansible

Here we create the same GitHub Pages & Jekyll environment. But instead of using shell scripts for provisioning we'll use [Ansible](ansible.md) (for more information about it check my tutorial).

Install Ansible on your host machine:

<http://docs.ansible.com/ansible/intro_installation.html#installing-the-control-machine>

NOTE: we are not setting an inventory file (which is basically a list of hosts) here since Vagrant will take care of it for us dynamically.

```bash
# install Ansible from source
cd /usr/local/src
git clone git://github.com/ansible/ansible.git --recursive

# install dependencies
sudo pip install paramiko PyYAML Jinja2 httplib2 six

alias setupansible='source /usr/local/src/ansible/hacking/env-setup'

# to use ansible
setupansible

# to update Ansible
cd /usr/local/src/ansible
git pull --rebase
git submodule update --init --recursive
```



Edit the Vagrantfile to have the provisioning done using Ansible:
```ruby
config.vm.provision "ansible" do |ansible|
  ansible.playbook = "ansible/playbook.yml"
end
```

Create the `ansible/playbook.yml` playbook:
```yaml
---
# This playbook only has one play
# And it applies to all hosts in the inventory file (which is dynamically
# created by Vagrant)
- hosts: all
  # we need priviledge escalation to install software, so we become root
  become: yes
  # and we become root using sudo
  become_method: sudo
  # to perform the following tasks:
  # (and tasks should always have a name)
  tasks:
    - name: Update package listing cache
      # use the Ansible apt module to:
      # update package list, but don't upgrade the system
      apt: update_cache=yes upgrade=no cache_valid_time=1800

    - name: Install packages required by GitHub Pages
      # use the Ansible apt module to:
      # install the listed packages to the latest available version
      apt: pkg={{item}} state=latest install_recommends=no
      with_items:
        - htop
        - ruby-dev
        - bundler
        - zlib1g-dev
        - ruby-execjs
        - python-pygments

    - name: Install the GitHub Pages Ruby gem
      # use the Ansible gem module to:
      # install the listed Ruby gems to the latest available version for all users
      gem: name={{item}} state=latest user_install=no
      with_items:
        - activesupport
        - github-pages
```

Deploy the environment:

```bash
setupansible
vagrant up
```





## DevEnv 2: AWS environment

The AWS provider is supported via the `vagrant-aws` plugin, which supports several AWS settings, so you may want to take a look at the [README](https://github.com/mitchellh/vagrant-aws#configuration) for any specific settings that you may need. However there are two caveats to it:
1) synced folders are supported via rsync (subject to rsync own limitations).
2) AWS credentials files are not supported. I have submitted a PR for the `vagrant-aws` plugin that [adds support for AWS config and credential files](https://github.com/mitchellh/vagrant-aws/pull/441), but until it gets (or if it gets) approved we're stuck either using environment variables or leaking credentials to the Vagrantfile.[^aws_plugin_wait]
This section shows a simple use case of the plugin.

[^aws_plugin_wait]: If you really need to have support for AWS credential files and can't wait for the PR to get merged and a new release come out, then take a look at the [Vagrantfile for the AWS recipe at my github](https://github.com/alexconst/vagrant_recipes/tree/master/blank-jessie64-aws).


AWS needs a dummy box so let us start by adding it:
```bash
# install the dummy AWS box
vagrant box add dummy https://github.com/mitchellh/vagrant-aws/raw/master/dummy.box
# create Vagrantfile
vagrant init dummy
```

Then adapt the following Vagrantfile with your credentials:
```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  # AWS dummy box
  config.vm.box = "dummy"

  config.vm.provider :aws do |aws, override|
    # configure your credentials and select your region
    aws.access_key_id = "YOUR KEY"
    aws.secret_access_key = "YOUR SECRET KEY"
    # the token is only needed if you're using one
    #aws.session_token = "YOUR SESSION TOKEN"
    aws.region = "YOUR REGION"

    aws.ami = "ami-e31a6594"
    # this username must exist in the specified aws.ami
    override.ssh.username = "admin"
    aws.instance_type = "t2.micro"
    # if your default SG does not allow for SSH then you need to specify it here
    #aws.security_groups = ['YOUR SG ALLOWING SSH']

    # set the keypair name that will be used (name as shown in AWS)
    aws.keypair_name = "KEYPAIR NAME"
    # the credential file should be the one for the aws.keypair_name
    override.ssh.private_key_path = ENV['HOME'] + "/.ssh/" + "YOUR PRIVATE KEYFILE"
  end

end
```

Then to start the development environment and connect to it:
```bash
vagrant up --provider=aws
vagrant ssh
```


## DevEnv 3: Multi-Machine environment

This is a mock example of a multi-machine environment. For a more realistic example, with complete provisioning, check this [Ansible](ansible.md) tutorial.

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|

  # Choose a box
  config.vm.box = "ubuntu/wily64"

  # Fine tune the virtualbox VM
  config.vm.provider "virtualbox" do |vb|
    vb.customize [
      "modifyvm", :id,
      "--cpus", "2",
      "--cpuexecutioncap", "50",
      "--memory", "384",
    ]
  end

  # Common provisioning
  config.vm.provision :shell, inline: 'echo "Hello World"'

  # web servers
  (1..2).each do |i|
    config.vm.define "web#{i}" do |node|
      node.vm.hostname = "web#{i}"
      node.vm.network :private_network, ip: "192.168.33.5#{i}"
      node.vm.network "forwarded_port", guest: 80, host: "808#{i}"
      # web server specific provisioning
      node.vm.provision :shell, inline: 'echo "Web server reporting for duty."'
    end
  end

  # db server
  config.vm.define "db" do |db|
    db.vm.hostname = "db"
    db.vm.network :private_network, ip: "192.168.33.40"
    db.vm.network "forwarded_port", guest: 5432, host: "54322"
    # db server specific provisioning
    db.vm.provision :shell, inline: 'echo "DB server ready to store and serve."'
  end

end
```
Because the Vagrantfile is a valid Ruby file it's possible to define a group of machines using the language constructs. Here the web servers are iterated and configured using the `node` object.
The settings applied to the `config` object (eg: `config.vm.box`) affect all the machines described in the Vagrantfile. While the settings applied to the `node` and `db` objects only affect the corresponding machines. This will become apparent when doing `vagrant up` due to the specified shell provisioning echo commands.

To use the environment:
```bash
# check status for all machines in the environment
vagrant status

# deploy the environment
vagrant up
# or just a single machine
vagrant up web1

# to connect to web1
vagrant ssh web1
# and since the machines are in a private network, you can connect from web1 to web2
ssh 192.168.33.52

# to destroy all machines in the environment without prompting the user
vagrant destroy -f
```

Finally, one important thing to be aware of when using these inner config objects, is how they are evaluated/executed. Vagrant enforces an outside-in ordering, which means that "nesting level" has priority over "line location". Taking this example from the manual[^docs_multimachine]:
```ruby
Vagrant.configure("2") do |config|
    config.vm.provision :shell, inline: "echo A"
    config.vm.define :testing do |test|
        test.vm.provision :shell, inline: "echo B"
    end
    config.vm.provision :shell, inline: "echo C"
end
```
The outside-in ordering means the provisioners will first output "A", then "C", and finally "B". Which is counter-intuitive to what someone would expect.

[^docs_multimachine]: <https://www.vagrantup.com/docs/multi-machine/>




# Other features

This tutorial covers most of what there is to know about Vagrant and its workflow. However, there are some other features which were not discussed but are worthwhile to be aware of. These are:

- Box catalogs
Hashicorp provides a public repository for their own boxes[^hashi_atlas], as well as third party boxes. For private boxes you can either subscribe to Hashicorp's Atlas service, or roll up your sleeves and set up your own box repository[^owncatalog].

- Certificates
To make the box download process secure use the appropriate Vagrantfile `config.vm` settings to specify which certificates to use and which checksums to perform.

- Share and connect
Vagrant, by means of the Hashicorp Atlas website, makes possible to share a development environment to anyone on the internet.

- Public networks
With the `config.vm.network` it is possible to define private and *as mentioned in the manual* less private networks (which will likely be replaced by bridged networks in the future). While private networks were covered, public networks were not.

- Provider specific settings
Vagrant allows configuring additional provider specific settings as well as overriding existing settings using the `config.vm.provider` option[^config_vm_provider]. These vary according to the provider but can include options for: choosing between headless/GUI mode, using linked clones, customizing the virtual hardware devices, set limits on processing power used, networking, etc. Some of VirtualBox specific settings[^vbox_specific] were used in the Vagrantfile for the GitHub Pages environment, but there are many more.

- SSH settings
On the scenario described in this tutorial only the SSH credentials needed to be provided. But Vagrant allows configuring several other SSH related settings, such as: guest domain/IP, guest port, private key, agent forward[^agent_harmful], X11 forward, environment forward, pty, shell and sudo settings[^ssh_settings].


[^hashi_atlas]: <https://atlas.hashicorp.com/boxes/search>
[^owncatalog]: <https://github.com/hollodotme/Helpers/blob/master/Tutorials/vagrant/self-hosted-vagrant-boxes-with-versioning.md>
[^config_vm_provider]: <https://www.vagrantup.com/docs/providers/configuration.html>
[^ssh_settings]: <https://www.vagrantup.com/docs/vagrantfile/ssh_settings.html>
[^vbox_specific]: <https://www.virtualbox.org/manual/ch08.html#vboxmanage-modifyvm>
[^agent_harmful]: Do not use agent forward, instead use proxy command. <https://heipei.github.io/2015/02/26/SSH-Agent-Forwarding-considered-harmful/>


# Known issues

Related to synced folders:
- VirtualBox shared folders has a bug related to the sendfile syscall that webservers typically use to serve static files. There is workaround for web servers[^sendfile] but the bug is still present (5 years and counting)[^vboxbug] and therefore may manifest itself with other software. A safer approach may be switching to NFS.
- Performance issues may also arise with VirtualBox shared folders: *"In some cases the default shared folder implementations (such as VirtualBox shared folders) have high performance penalties. If you're seeing less than ideal performance with synced folders, NFS can offer a solution."*[^nfs]
- Vagrant supports different synced folders implementations. However support for symbolic links is inconsistent, so it is better throughly test them beforehand or to avoid them altogether.

[^nfs]: <https://docs.vagrantup.com/v2/synced-folders/nfs.html>
[^sendfile]: <https://docs.vagrantup.com/v2/synced-folders/virtualbox.html>
[^vboxbug]: <https://www.virtualbox.org/ticket/9069>







# Debugging and tips

- Whenever the Vagrantfile is updated changes will only be applied after a `vagrant reload` or `vagrant up`.

- When implementing the provisioning modules for an environment there a few commands that can help speeding up the process and troubleshooting problems:
    - both `vagrant provision` and `vagrant reload --provision` will execute the provisioning steps on a running environment.
    - `vagrant snapshot push` and `vagrant snapshot pop` for reverting undesired changes.
    - `vagrant ssh -c $your_cmd_here` will execute the provided command.

- If no SSH keypair was added during the creation process of a Vagrant box, or there isn't a `vagrant` user, then it's necessary to configure the `config.ssh.username` and `config.ssh.password` settings with valid credentials.

- To connect to a development environment directly without going through Vagrant: `ssh -p $(vagrant port --guest 22) -l $box_username localhost`.

- To enable logging:
```bash
# supported levels are: debug, info, warn, error
export VAGRANT_LOG="debug"
```

- To run the latest version of Vagrant or any of its plugins from GitHub's master branch (which BTW is not recommended), then clone the repo and then use `bundle` to run vagrant. Check the [vagrant_dev-wily64-atlas](https://github.com/alexconst/vagrant_recipes/tree/master/vagrant_dev-wily64-atlas) recipe for more details, but it would go something like this:
```bash
git clone git@github.com:alexconst/vagrant-aws.git
cd vagrant-aws
bundle exec vagrant version
bundle exec vagrant $your_commands
```



# Troubleshooting

- failure when installing the vagrant-libvirt plugin
TRIGGER: `vagrant plugin install vagrant-libvirt`
ERROR: Make sure that `gem install ruby-libvirt -v '0.6.0'` succeeds before bundling.
PROBLEM: you're probably missing both libvirt-dev and the ruby-libvirt gem
SOLUTION: `apt-get install libvirt-dev` and then `gem install ruby-libvirt -v '0.6.0'`

- failure when adding a new box
ERROR:
```text
/opt/vagrant/embedded/lib/ruby/2.2.0/fileutils.rb:252:in 'mkdir': File name too long @ dir_s_mkdir - /home/$USER/.vagrant.d/boxes/file:-VAGRANTSLASH--VAGRANTSLASH--VAGRANTSLASH-...-VAGRANTSLASH-debian-802-jessie-amd64_jessie-vboxiso.box (Errno::ENAMETOOLONG)
```
SOLUTION: `cd` into the directory with the box file and run the `vagrant box add` command from there refering directly to the filename (ie: no path)

- non-fatal error message about stdin and tty
ERROR: `stdin: is not a tty`
REFERENCES:
http://foo-o-rama.com/vagrant--stdin-is-not-a-tty--fix.html
https://github.com/Varying-Vagrant-Vagrants/VVV/issues/517
https://github.com/mitchellh/vagrant/issues/1673
SOLUTION: add the following as the first provisioner:
```ruby
config.vm.provision "fix-no-tty", type: "shell" do |s|
    s.privileged = false
    s.inline = "sudo sed -i '/tty/!s/mesg n/tty -s \\&\\& mesg n/' /root/.profile"
end
```

- non-fatal error message about stdin when using apt
ERROR: `dpkg-preconfigure: unable to re-open stdin: No such file or directory`
REFERENCES:
http://serverfault.com/questions/500764/dpkg-reconfigure-unable-to-re-open-stdin-no-file-or-directory
SOLUTION 1: prepend the following line to any script with apt-get calls:
```bash
export DEBIAN_FRONTEND=noninteractive
```
SOLUTION 2: add the following lines before running other provisioners that use apt
```ruby
config.vm.provision "shell", inline: "echo 'export DEBIAN_FRONTEND=noninteractive' >> /root/.profile"
config.vm.provision "shell", inline: "for user in /home/*; do echo 'export DEBIAN_FRONTEND=noninteractive' >> $user/.profile; done"
```

- unable to access the guest directly via SSH when using a private network
NOTE: failure to connect to the guest via SSH may be due to sshd configuration issues. If you can ping the machine and telnet to the SSH port, then the private network is working. If still not convinced yet, then install lighttpd and access port 80 from the host. Make sure you're using the correct IP and port. Example:
```bash
ssh vagrant@$private_network_ip
# example:
ssh vagrant@192.168.22.50
```
If `/etc/ssh/sshd_config` has `PasswordAuthentication no` then you'll have to use a command similar to this one:
```bash
ssh -i ./.vagrant/machines/default/virtualbox/private_key vagrant@192.168.22.50
```







# Footnotes






