# What is Vagrant

Vagrant is a tool for deploying software environments in a configurable and reproducible form, from a single source configuration file. Whereas a software environment consists of one or more virtual machine or container instances; and most likely with network port forwarding and folder shares configured as well.
Its typical use case is the deployment of development environments. Because the VM deployment is reproducible it makes possible to have the same exact environment across multiple machines, thus eliminating the problems caused by differences in system configuration both across teams and infrastructure.
With a single command, `vagrant up`, it becomes possible to deploy a whole environment in just a few minutes.
Supported providers (ie: virtualization platforms) include: VirtualBox, VMWare, HyperV, Docker, KVM, AWS and others.
The provisioning (ie: installation and configuration of software into the instantiated VM, or container in the case of Docker) can be done using one or more of the supported configuration management tools: shell scripts, Ansible, CFEngine, Chef, Docker, Puppet, Salt.
The Vagrant VM images are commonly referred as *Vagrant boxes* and they are provider specific. While an instantiated VM is referred as *development environment*.

**Vagrant recipes used in this tutorial: **
<https://github.com/alexconst/vagrant_recipes>



# Understanding the magic

Vagrant acts as a wrapper around different providers (eg: VirtualBox) and provisioning tools (eg: Ansible), making the whole process:
- easy: with just a couple of commands you can ssh into a new development environment;
- simple: no need to configure any settings from each virtualization software;
- uniform: it works the same way across multiple cloud providers and virtualization platforms (assuming Vagrant boxes are available for those providers); and
- reproducible: you always get the same development environment when you start a new development environment.

The Vagrant boxes are created using Packer. And these consist on a tar file which include the virtualization software VM files (eg: .ovf and .vmdk) and a couple of Vagrant specific metadata files.
The configuration of a development environment is done via a Vagrantfile. It includes settings for box selection, network port forwarding, shared folders, credentials, provisioning scripts, and other settings; for one or more machine instances.
Vagrant then acts as a wrapper to the different virtualization software simplifying the whole process. For example, when `vagrant up` is executed it downloads a VM image, unpacks it, starts the VM, installs and configures software packages, and sets up the network and shared folders.
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
- VMware Workstation/Fusion: a free plugin which according to the author is not mature enough to be called alpha. <https://github.com/orishavit/vagrant-vmware-free>
- KVM and QEMU: <https://github.com/pradels/vagrant-libvirt>
- AWS cloud: <https://github.com/mitchellh/vagrant-aws>
- Google cloud: <https://github.com/mitchellh/vagrant-google>
- Rackspace cloud: <https://github.com/mitchellh/vagrant-rackspace>
- VMware vSphere: <https://github.com/nsidc/vagrant-vsphere>
- VMware vCenter: the project looks dead <https://github.com/frapposelli/vagrant-vcenter>

Other interesting plugins:
- vbguest: automatically keep the VirtualBox guest tools updated <https://github.com/dotless-de/vagrant-vbguest/>
- mutate: convert boxes to work with different providers <https://github.com/sciurus/vagrant-mutate>

List of plugins: <http://vagrant-lists.github.io/>

Installing plugins:
````bash
vagrant plugin install vagrant-libvirt
vagrant plugin install vagrant-vmware-free
vagrant plugin install vagrant-aws
vagrant plugin list
vagrant plugin update
````
Plugins are downloaded from the ruby gems repository and are installed to `$HOME/.vagrant.d/gems/gems/`.
Note that the installed plugins will be older than the lastest versions from github. However installing the plugins directly from github is more troublesome and is out of scope in this tutorial.




# Using Vagrant


## Basics

This section characterizes a typical Vagrant workflow.

````bash
# select a box
# it can be a box available locally, eg: file:///$HOME/vagrant_boxes/debian_jessie.box
# or it can be a box made available at Hashicorp's Atlas, eg: ubuntu/trusty64
boxname="ubuntu/trusty64"

# check version
vagrant version

# generate a Vagrantfile set to use given box
vagrant init $boxname

# deploy the vagrant box (ie: download if needed, unpack, and instantiate the box)
vagrant up
# if you have multiple providers for the same box then you'll need to specify which
# with $provider being one of 'virtualbox', 'vmware', 'aws', 'qemu', etc
vagrant up --provider $provider

# connect to the development environment
vagrant ssh

# get the status of existing environments
vagrant status

# show existing network port mappings
vagrant port

# suspend the development environment
vagrant suspend

# resume a previously suspended environment
vagrant resume

# delete the running development environment
vagrant destroy

# get a list of available boxes
vagrant box list
````




## Snapshots

Taking snapshots is simple, but there are a few caveats.
If you use `push` and `pop` it is not recommended to mix them with `save` and `restore` and `delete`.
Also some providers (VirtualBox included) require that when `delete` is used that all subsequent snapshots are deleted in the reverse order that they were taken.
Using snapshots (choose one and only one option):
````bash
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
````




## Your own environment

This section details how to add a local Vagrant box and set up a development environment.

Add a local Vagrant box to the pool of available boxes:
````bash
# select the box
boxfile="debian-802-jessie-amd64_jessie-vboxiso.box"
boxname="alexconst/debian/jessie64/20160115.0.1"
# add the box
vagrant box add  "${boxfile}"  --name "${boxname}"
# check that it has been added
vagrant box list

# FYI, to remove a box
vagrant box remove $boxname
````
Note that there is a `--box-version` option to the `box add` command but that is only for remote clouds (eg: Hashicorp Atlas), so we workaround that limitation by adding the version tag to the box name. This "workaround" does give us visual identification, but doesn't really provide the version constraint features of that command option.


Create a blank Vagrantfile:
````bash
vagrant init
````

Our use case matches the default scenario where the guest is Linux and communication is done via `ssh`. But Vagrant also supports Windows guest where communication is done via `winrm`.
Vagrant also supports synced folders via different implementations: NFS (Linux only), Samba (Windows only), Rsync, and the provider's own mechanism (eg: VirtualBox shared folders). However these are not perfect; do check the [Known issues](#known-issues) section for more information.

Configure a new environment, that uses the new box, by editing the Vagrantfile as follows:
````ruby
# select our box
config.vm.box = "alexconst/debian/jessie64/20160115.0.1"

# set hostname
config.vm.hostname = "vagrant-devbox"

# set text to be shown after 'vagrant up'
# this text can be useful for detailing instructions how to access the development environment
config.vm.post_up_message = "A Vagrant tutorial can be found at http://alexconst.github.io/"

# configure shared folders (1st host, 2nd guest)
config.vm.synced_folder "$HOME/share_vagrant", "/vagrant"

# configure the network: set up port forwarding for TCP
# accessing port 8080 on the host will be forwarded to port 80 on the guest
config.vm.network "forwarded_port", host: 8080, guest: 80

# configure the network: set up port forwarding for both UDP and TCP
config.vm.network "forwarded_port", host: 12003, guest: 2003, protocol: "tcp"
config.vm.network "forwarded_port", host: 12003, guest: 2003, protocol: "udp"

# provision done by uploading a script and executing it (path can be an URL)
config.vm.provision "shell", path: "scripts/packages.sh"
# provision done by executing an already existing script in the guest
config.vm.provision "shell", inline: "/bin/bash /path/to/script.sh"

````


Instantiate a new environment using our box and connect to it:
````bash
vagrant up
vagrant ssh
````



## AWS environment

_TODO_ AWS




## Multi-machine environment

_TODO_ detail multiple machines with ansible





# Other features

This tutorial covers most of what there is to know about Vagrant and its workflow. However, there are some other features which were not discussed but are worthwhile to be aware of. These are:

- Box catalogs
Hashicorp provides a public repository for their own boxes, as well as third party boxes. For private boxes you can either subscribe to Hashicorp's Atlas service, or roll up your sleeves and set up your own box repository[^owncatalog].

- Certificates
To make the box download process secure use the appropriate Vagrantfile `config.vm` settings to specify which certificates to use and which checksums to perform.

- Share and connect
Vagrant, by means of the Hashicorp Atlas website, makes possible to share a development environment to anyone on the internet.

- Private and public networks
With the `config.vm.network` it is possible to define private and *as mentioned in the manual* less private networks (which will likely be replaced by bridged networks in the future).

- Provider specific settings
Vagrant allows configuring additional provider specific settings as well as overriding existing settings using the `config.vm.provider` option[^config_vm_provider]. These vary according to the provider but can include options for: choosing between headless/GUI mode, using linked clones, customizing the virtual hardware devices, set limits on processing power used, networking, etc.

- SSH settings
On the scenario described in this tutorial only the SSH credentials needed to be provided. But Vagrant allows configuring several other SSH related settings, such as: guest domain/IP, guest port, private key, agent forward, X11 forward, environment forward, pty, shell and sudo settings[^ssh_settings].

[^owncatalog]: <https://github.com/hollodotme/Helpers/blob/master/Tutorials/vagrant/self-hosted-vagrant-boxes-with-versioning.md>
[^config_vm_provider]: <https://www.vagrantup.com/docs/providers/configuration.html>
[^ssh_settings]: <https://www.vagrantup.com/docs/vagrantfile/ssh_settings.html>


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

- If no SSH keypair was added during the creation process of a Vagrant box, or there isn't a `vagrant` user, then it's necessary to configure the `config.ssh.user` and `config.ssh.password` settings with valid credentials.

- To connect to a development environment directly without going through Vagrant: `ssh -p $(vagrant port --guest 22) -l $box_username localhost`.

- To enable logging:
````bash
# supported levels are: debug, info, warn, error
export VAGRANT_LOG="debug"
````




# Troubleshooting

- failure when installing the vagrant-libvirt plugin
TRIGGER: `vagrant plugin install vagrant-libvirt`
ERROR: Make sure that `gem install ruby-libvirt -v '0.6.0'` succeeds before bundling.
PROBLEM: you're probably missing both libvirt-dev and the ruby-libvirt gem
SOLUTION: `apt-get install libvirt-dev` and then `gem install ruby-libvirt -v '0.6.0'`

- failure when adding a new box
ERROR:
````text
/opt/vagrant/embedded/lib/ruby/2.2.0/fileutils.rb:252:in 'mkdir': File name too long @ dir_s_mkdir - /home/$USER/.vagrant.d/boxes/file:-VAGRANTSLASH--VAGRANTSLASH--VAGRANTSLASH-...-VAGRANTSLASH-debian-802-jessie-amd64_jessie-vboxiso.box (Errno::ENAMETOOLONG)
````
SOLUTION: `cd` into the directory with the box file and run the `vagrant box add` command from there refering directly to the filename (ie: no path)

- non-fatal error message about stdin and tty
ERROR: `stdin: is not a tty`
REFERENCES:
http://foo-o-rama.com/vagrant--stdin-is-not-a-tty--fix.html
https://github.com/Varying-Vagrant-Vagrants/VVV/issues/517
https://github.com/mitchellh/vagrant/issues/1673
SOLUTION: add the following as the first provisioner:
````ruby
config.vm.provision "fix-no-tty", type: "shell" do |s|
    s.privileged = false
    s.inline = "sudo sed -i '/tty/!s/mesg n/tty -s \\&\\& mesg n/' /root/.profile"
end
````

- non-fatal error message about stdin when using apt
ERROR: `dpkg-preconfigure: unable to re-open stdin: No such file or directory`
REFERENCES:
http://serverfault.com/questions/500764/dpkg-reconfigure-unable-to-re-open-stdin-no-file-or-directory
SOLUTION 1: prepend the following line to any script with apt-get calls:
````bash
export DEBIAN_FRONTEND=noninteractive
````
SOLUTION 2: add the following lines before running other provisioners that use apt
````ruby
config.vm.provision "shell", inline: "echo 'export DEBIAN_FRONTEND=noninteractive' >> /root/.profile"
config.vm.provision "shell", inline: "for user in /home/*; do echo 'export DEBIAN_FRONTEND=noninteractive' >> $user/.profile; done"
````


# Footnotes






