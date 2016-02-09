---
layout: post
title:  'Ansible – automating system configuration'
author: 'Alexandre Constantino'
date:   2016-02-08
categories: devops
tags:       tutorial ansible vagrant
---

# About

This is a concise tutorial on Ansible, which for demonstration purposes uses [Vagrant](vagrant.md) for machine/hardware abstraction.
It is split into the following main sections:
- installation;
- TODO!


# What is Ansible

*"Ansible is an IT automation tool. It can configure systems, deploy software, and orchestrate more advanced IT tasks such as continuous deployments or zero downtime rolling updates."* [^docs_ansible]
In order to to this it uses text files where configuration management, deployment and orchestration tasks are defined.
The advantage of using a provisioning tool like Ansible is that by using its configuration files it makes the whole process reproducible and scalable to hundreds or thousands of servers. With the benefit that these configuration files can be put under version control.
An Ansible recipe is composed by:
- one or more YAML files,
- an inventory file, where target host machines are listed and grouped, and
- an optional Ansible configuration file (if specified it uses the one given in the command line, then falls back to the local directory, and finally to the user home directory).

[^docs_ansible]: <http://docs.ansible.com/ansible/>

Other well known provisioning tools include: Puppet (2005), Chef (2008) and Salt (2011). So why choose Ansible (2012)?
There have been multiple discussions[^disc1] [^disc2] [^disc3] on this topic but with no clear winner standing out. The main reasons for this being related with the maturity of each tool, its prevalence inside a company, and the pros and cons that each tool brings. Although Chef does tend to be well regarded, and Ansible does tend to be recommended for new users or systems.
The main reasons for Ansible:
- easy learning curve (due to the use of YAML, Python and the documentation),
- excellent documentation,
- declarative paradigm (configuration is done as data via YAML files not code),
- agent-less architecture (only SSH is used, so no potential vulnerable agents get installed),
- batteries included (more than 400 modules), and
- use of Python (loops are supported via the Jinja2 templating language).



[^disc1]: <http://www.emir.works/configuration-management-battlefield/>
[^disc2]: <https://dantehranian.wordpress.com/2015/01/20/ansible-vs-puppet-overview/>
[^disc3]: <http://ryandlane.com/blog/2014/08/04/moving-away-from-puppet-saltstack-or-ansible/>





# Installation


To install Ansible on your host machine:

<http://docs.ansible.com/ansible/intro_installation.html#installing-the-control-machine>


```bash
# install Ansible from source
cd /usr/local/src
git clone git://github.com/ansible/ansible.git --recursive

# install dependencies
sudo pip install paramiko PyYAML Jinja2 httplib2 six

# set up inventory file
echo "" >> ~/ansible_hosts

# add these lines to your shell rc file
export ANSIBLE_INVENTORY=$HOME/ansible_hosts
alias setupansible='source /usr/local/src/ansible/hacking/env-setup'

# to use ansible
setupansible


# to update Ansible
cd /usr/local/src/ansible
git pull --rebase
git submodule update --init --recursive
```



# Ansible overview

```bash
ansible --version
# ansible 2.1.0 (devel 6b166a1048) last updated 2016/01/23 17:47:53 (GMT +100)
```


## Operation modes

Ansible supports two operation modes: ad-hoc mode and playbook mode.

### Ad-hoc mode

In *ad-hoc mode* commands are executed from the command line.
Example: use the `ping` module to ping `all` the nodes in the `web.ini` inventory file, under the `vagrant` remote user.
```bash
ansible -m ping -u vagrant -i nodes.ini all --ask-pass

# NOTE: if you get a failure (and assuming this is a trusted environment, ie on a local VM) you may have to:
ssh-keygen -R $node
ssh-keyscan -H $node >> ~/.ssh/known_hosts
```

### Playbook mode

With *playbook mode* commands are executed sequentially as defined in the playbook file.
Example: update package listing and install htop.
```yaml
---
# This playbook only has one play
# And it applies to all hosts in the inventory file
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

    - name: Install package
      # use the Ansible apt module to:
      # install the listed packages to the latest available version
      apt: pkg={{ item }} state=latest
      with_items:
        - htop
```
To run the playbook:
```bash
ansible-playbook -i nodes.ini -u vagrant provision.yml --ask-pass

# NOTE: you may need to install the sshpass package on your host machine
```

Here is the `nodes.ini` inventory file:
```ini
192.168.22.50
```

And the Vagrantfile used:
```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|

  # Choose a box with VBox guest tools already installed and a Ruby version
  # compatible with GitHub Pages and Jekyll.
  config.vm.box = "ubuntu/wily64"

  # Set up hostname
  config.vm.hostname = "ansible-test"

  # Message shown on vagrant up
  config.vm.post_up_message = ""

  # Assign a static IP to the guest
  config.vm.network "private_network", ip: "192.168.22.50"

  # Share an additional folder with the guest VM.
  host_folder = ENV['HOME'] + "/home/downloads/share_vagrant"
  guest_folder = "/shared/"
  config.vm.synced_folder host_folder, guest_folder

  # Fine tune the virtualbox VM
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
  #config.vm.provision "shell", path: "scripts/foo.sh"

end
```


## Inventory file

The inventory file lists and groups target host machines where the playbooks can be executed. A inventory file can look like this:
```ini
# group 'web' includes 30 webservers
[web]
webserver-[01:30].example.com

# and group 'db' includes 6 db servers
[db]
dbserver-[a-f].example.com

# This is how the Vagrant inventory file looks like for the Vagrant machine named 'default'
default ansible_ssh_host=127.0.0.1 ansible_ssh_port=2200 ansible_ssh_user='vagrant' ansible_ssh_private_key_file='/path/to/.vagrant/machines/default/virtualbox/private_key'
```

Or more complex:
```ini
# define group with 2 hosts
[europe]
host1
host2

# define group asia, where one of the hosts is also in the 'europe' group
# this may imply commands being executed twice on this host
[asia]
host2
host3

# define a group of groups using the 'children' keyword
[euroasia:children]
europe
asia

# and set variables, to be used in the playbooks, for the 'euroasia' group
# NOTE: best practices actually recommend having the variables defined in a separate YAML file
[euroasia:vars]
ntp_server=ntp.london.example.com
proxy=proxy.london.example.com
some_custom_var=foobar

[global:children]
euroasia
america
oceania
```














# Tips

- To check if the playbook is valid execute `ansible-playbook --syntax-check playbook.yml`

- To log the output of a playbook:
http://stackoverflow.com/questions/18794808/how-do-i-get-logs-details-of-ansible-playbook-module-executions

- Check that your hosts are reachable `ansible all -m ping` (install the `sshpass` package on your host system and add the `--ask-pass` option if you didn't propagate an SSH keypair).




# Troubleshooting


- failure executing playbook
ERROR: `ansible "Failed to lock apt for exclusive operation"`
PROBLEM: the playbook or task needs to be executed as root. Note that `sudo: yes` has been deprecated and replaced by the `become` and `become_method` directives.
SOLUTION: 
````yaml
become: yes
become_method: sudo
````

- ruby gems not installed for all users
PROBLEM: the ruby gems are installed for the user running the playbook
SOLUTION: to make the gems install to /var/lib instead make sure you use the `user_install=no` option and `vagrant destroy` to start over.
REFERENCES:
http://docs.ansible.com/ansible/gem_module.html
http://stackoverflow.com/questions/22115936/install-bundler-gem-using-ansible


- unable to connect to machine
- ERROR: SSH encountered an unknown error during the connection. We recommend you re-run the command using -vvvv, which will enable SSH debugging output to help diagnose the issue
- TROUBLESHOOTING: the recommendation is valid
- SOLUTION: if the problem is related to `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED` then (assuming you understand the cause of the problem) remove the key with `ssh-keygen -R $node` and try again.


