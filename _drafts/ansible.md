---
layout: post
title:  'Ansible – automating system configuration'
author: 'Alexandre Constantino'
date:   2016-02-08
categories: devops
tags:       tutorial ansible vagrant
---

# About

This is a concise tutorial on Ansible. It starts by giving a description on what Ansible is and what it is used for, provides instructions how to install it, and gives an overview on its playbooks. Then goes into more detail on the files used by Ansible: inventory, configuration, and playbooks. It finalizes by showcasing multiple use cases. [Vagrant](vagrant.md) will be used in this tutorial since it provides us a convenient way to make development environments easily available for testing Ansible.

# What is Ansible

*"Ansible is an IT automation tool. It can configure systems, deploy software, and orchestrate more advanced IT tasks such as continuous deployments or zero downtime rolling updates."* [^docs_ansible]
In order to to this it uses text files where configuration management, deployment and orchestration tasks are defined.
The advantage of using a provisioning tool like Ansible is that by using its configuration files it makes the whole process reproducible and scalable to hundreds or thousands of servers. With the benefit that these configuration files can be put under version control. Another advantage is that Ansible's modules are implemented to be idempotent; a property not present when using shell scripts.

An Ansible recipe is composed by:
- one or more YAML playbook files, which define the tasks to be executed,
- an inventory file, where target host machines are listed and grouped, and
- an optional Ansible configuration file.

[^docs_ansible]: <http://docs.ansible.com/ansible/>

Other well known provisioning tools include: Puppet (2005), Chef (2008) and Salt (2011). So why choose Ansible (2012)?
There are multiple discussions[^disc1] [^disc2] [^disc3] on this topic but with no clear winner standing out. The main reasons for this being related with the maturity of each tool, its prevalence inside a company, and the pros and cons that each tool brings. Nonetheless, Chef does tend to be well regarded (and a better alternative than its Ruby counterpart Puppet), and Ansible does tend to be recommended for new users or installations.

The main reasons for Ansible:
- excellent documentation,
- easy learning curve (due to the use of YAML, Python and its documentation),
- declarative paradigm (configuration is done as data via YAML files not code),
- agent-less architecture (only SSH is used, so no potential vulnerable agents get installed),
- batteries included (more than 400 modules),
- use of Jinja2 templating language (for variables and loop contructs), and
- Ansible Galaxy (a repository with thousands of Ansible recipes which you can customize to your needs).



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

# install dependencies:
sudo pip install paramiko PyYAML Jinja2 httplib2 six
# needed for the AWS EC2 inventory:
sudo pip install boto

# set up a default inventory file
echo "" >> ~/ansible_hosts

# add these lines to your shell rc file
export ANSIBLE_INVENTORY=~/ansible_hosts
export ANSIBLE_HOME="/usr/local/src/ansible"
alias env-ansible="source $ANSIBLE_HOME/hacking/env-setup"
# needed for the AWS EC2 inventory:
export ANSIBLE_EC2="$ANSIBLE_HOME/contrib/inventory/ec2.py"
alias ansible-inv-ec2="$ANSIBLE_EC2"
export EC2_INI_PATH="ec2.ini"

# to use ansible
env-ansible


# to update Ansible
cd /usr/local/src/ansible
git pull --rebase
git submodule update --init --recursive
```







# Ansible overview

```bash
ansible --version
# ansible 2.1.0 (devel 0f15e59cb2) last updated 2016/02/09 15:31:35 (GMT +100)
```



Ansible supports two operation modes: ad-hoc mode and playbook mode.


In *ad-hoc mode* commands are executed from the command line.
Example:
```bash
# "ping" `all` nodes in the `nodes.ini` inventory file using the `vagrant` remote user
# the `ping` module tries to connect to a host, verify a usable python and return pong on success
ansible -m ping -u vagrant -i nodes.ini all --ask-pass

# or gather facts from the machine
ansible -m setup -u vagrant -i nodes.ini all --ask-pass > facts.txt

# NOTE: if you get a failure (and assuming this is a trusted environment, ie on a local VM) you may have to:
ssh-keygen -R $node
ssh-keyscan -H $node >> ~/.ssh/known_hosts
```


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
# check that the playbook syntax is correct
ansible-playbook --syntax-check provision.yml
# run the playbook
ansible-playbook -i nodes.ini -u vagrant provision.yml --ask-pass

# NOTE: you may need to install the sshpass package on your host machine
```

Here is the `nodes.ini` inventory file used:
```ini
192.168.22.50
```

And a simplified Vagrantfile for testing the recipe:
```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|

  # Choose a box with VBox guest tools already installed and a Ruby version
  # compatible with GitHub Pages and Jekyll.
  config.vm.box = "ubuntu/wily64"

  # Set up hostname
  config.vm.hostname = "ansible-test"

  # Assign a static IP to the guest
  config.vm.network "private_network", ip: "192.168.22.50"

end
```

Alternatively you could also have this `vagrant.ini` inventory file:
```ini
# assumes pwd is the Vagrant dir
default ansible_ssh_host=192.168.22.50 ansible_ssh_port=22 ansible_ssh_user='vagrant' ansible_ssh_private_key_file='./.vagrant/machines/default/virtualbox/private_key'
```
And run this command instead:
```bash
ansible-playbook -i vagrant.ini provision.yml
```








# Inventory files

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

# define group 'asia', where one of the hosts is also in the 'europe' group
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



## Dynamic inventory files (with AWS)

Ansible also provides a way to get an inventory of hosts from third party sources, which is particularly useful when dealing with cloud providers. Ansible includes support for: AWS EC2, Digital Ocean, Google CE, Linode, OpenStack, among others. And it also allows creating one's own dynamic inventory system[^dev_dyn_inv].

[^dev_dyn_inv]: <http://docs.ansible.com/ansible/developing_inventory.html>

Here follows an example of AWS EC2 dynamic inventory system, which assumes that you've done the installation steps described in the aforementioned section (namely installing the needed packages and setting alias and environment variables in your shell rc file).


Get your EC2 external inventory script settings file ready:
```bash
# option 1:
# either use the default location previously set in your shell rc file
# which should simply be "ec2.ini", point to the current dir
echo $EC2_INI_PATH
# and, unless you have one already, copy the provided ec2.ini to the local dir
cp $ANSIBLE_HOME/contrib/inventory/ec2.ini .

# option 2:
# or set the path to your ec2 ini file
export EC2_INI_PATH="/path/to/ec2.ini"
```

You may wish to edit the `ec2.ini` file to better suite your needs. For example, you can speed up the query process by including or excluding regions of interest with the `regions` and `regions_exclude` variables. Or change the `cache_max_age` variable that specifies how long cache results are valid and API calls are skipped.

To get a listing of running instances:
```bash
ansible-inv-ec2 --list

# or if you need to refresh the cache
ansible-inv-ec2 --list --refresh-cache

# or if you want to choose a particular AWS profile, in this case 'dev'
AWS_PROFILE="dev" ansible-inv-ec2 --list --refresh-cache
```

To execute an ad-hoc command:
```bash
# by default Debian machines have the 'admin' user while Ubuntu have the 'ubuntu' user
instance_user="admin"
# path to your SSH keypair
instance_key="$HOME/.ssh/aws_developer.pem"
# despite passing a profile, you still need to specify a region
region="eu-west-1"

# execute the ping module
AWS_PROFILE="dev"  ansible -i "$ANSIBLE_EC2" -u "$instance_user" --private-key="$instance_key" "$region"  -m ping
```

To run our playbook that installs htop:
```bash
# by default Debian machines have the 'admin' user while Ubuntu have the 'ubuntu' user
instance_user="admin"
# path to your SSH keypair
instance_key="$HOME/.ssh/aws_developer.pem"

# run the playbook
AWS_PROFILE="dev"  ansible-playbook -i "$ANSIBLE_EC2" -u "$instance_user" --private-key="$instance_key"   provision.yml
```

Another possible approach would be querying EC2 for an inventory, grouping them according to thei EC2 tags, and saving the results to a local file. This would then give more flexibility in task execution.





# Configuration files

Ansible will use the configuration options found on the first file that it finds from the following list:
- `ANSIBLE_CONFIG`: an environment variable pointing to a config file
- `ansible.cfg`: in the current directory
- `.ansible.cfg`: in the home directory
- `/etc/ansible/ansible.cfg`

NOTE: it will only use one file. Settings are not merged.

The configuration file can be used to set a multitude of options regarding connectivity, parallelism, privilege escalation, among other settings. Nearly all of these options can be overridden in the playbooks or via command line flags. Check the documentation[^doc_cfg] for a list of all options. Some of the most useful ones are:
- `forks`: the default number of processes to spawn when communicating with remote hosts. By default it is automatically limited to the number of possible hosts.
- `gathering`: by default is set to `implicit` which ignores the fact cache and gathers facts per play unless the `gather_facts: False` is set in the playbook. The `explicit` option does the opposite. The `smart` setting will only gather facts once per playbook. Both `explicit` and `smart` use the facts cache.
- `log_path`: if configured it will be used to log information.
- `nocows`: set to 1 if you don't like them.
- `private_key_file`: points to a private key file. You can use this config option instead of the `ansible --private-key`.
- `vault_password_file`: sets the path to Ansible Vault password file.

If you're looking to optimize your operations look into the `pipelining` and `accelerate_*` options.

To configure Ansible to your needs make a copy of the template at `$ANSIBLE_HOME/examples/ansible.cfg` to your local dir.

[^doc_cfg]: <http://docs.ansible.com/ansible/intro_configuration.html>



# Playbook files

Playbooks are YAML files that describe configuration, deployment and orchestration operations to be performed on a group of nodes.
Each *playbook* contains a list of  *plays*, and each *play* includes a list of *tasks*, and each *task* calls an Ansible module. Tasks are executed sequentially according to the order defined in the playbook.
Apart from those there is also the concept of *handler* which is a task that executes at the end of a *play* (but only once) when it is triggered by any of the *tasks* (that were set to notify that handler). They are typically used to restart services and reboot the machine.

Parametrization of a playbook can be done via *variables*. These can be defined in playbooks, inventories, and via the command line. A special class of variables goes by the name of *facts*, which consist on information gathered from a targeted node, and are particularly useful when dealing with config files that need external IP addresses.

Commonly used entries in a playbook:
- `hosts`: a list of one or more groups of nodes where the playbook will be executed.
- `vars`: a list of variables that can be used both in the Jinja2 template files and also in the tasks in the playbook.
- `remote_user`: the remote user used to login to the node.
- `become`: if set to `yes` the remote user will switch to root before executing the tasks.
- `become_method`: defines the switch user method, typically it's `sudo`.
- `tasks`: a list of tasks.
- `task`: each tasks makes use of a module to perform an operation and eventually `notify` a handler.
- `handlers`: a list of handlers. With each handler being executed at most once, at the end of the playbook.

Commonly used modules:
- `apt` and `yum`: package management.
- `template`: evaluate the input Jinja2 template file and copy its result to the remote node.
- `copy`: copy a file to the remote node.
- `command`: execute a command without invoking a shell or using environment variables.
- `shell`: execute a command via the shell.


There are two other concepts that one should know about. The first it that playbooks can `include` other playbooks. And the other one is the concept of `roles`. Roles are a collection of *playbooks* that act as reusable building blocks. The file structure of a role can look something like this:
```bash
lamp_haproxy/roles/nagios
├── files
│   ├── ansible-managed-services.cfg
│   ├── localhost.cfg
│   └── nagios.cfg
├── handlers
│   └── main.yml
├── tasks
│   └── main.yml
└── templates
    ├── dbservers.cfg.j2
    ├── lbservers.cfg.j2
    └── webservers.cfg.j2
```
The [Ansible Galaxy](https://galaxy.ansible.com/) is a community driven website that has thousands of roles that can be reused and customized to specific needs.


Because the best way to understand playbooks is via examples the next sections will do just that. But be sure to read the [Best Practices](http://docs.ansible.com/ansible/playbooks_best_practices.html) guide first, as it will help making the most out of the playbooks and Ansible.


# Example 1: Nginx






# Debugging and tips

- To check if the playbook is valid execute `ansible-playbook --syntax-check playbook.yml`.

- To have the playbook execute as a dry run (ie, without really executing anything) `ansible-playbook --check playbook.yml`.

- To get the stdout and stderr of each task executed in the playbook use the `-v` flag. 

- To enable logging set the `log_path` in your `ansible.cfg` file. [^ansible_log]

[^ansible_log]: <http://stackoverflow.com/questions/18794808/how-do-i-get-logs-details-of-ansible-playbook-module-executions>

- To check if your nodes are reachable execute `ansible all -m ping` (install the `sshpass` package on your host system and add the `--ask-pass` option if you didn't propagate an SSH keypair).

- When dealing with large playbooks it may be useful to change the execution entry point or choose which tasks to execute. By using the `--tags`/`--skip-tags` options when executing a playbook it's possible to filter which tasks get/don't get executed. And with the `--start-at-task` it's possible to choose a starting point to the playbook. A `--step` option is also provided to allow executing a playbook in interactive mode.


Other references:
- http://docs.ansible.com/ansible/test_strategies.html



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
ERROR: SSH encountered an unknown error during the connection. We recommend you re-run the command using -vvvv, which will enable SSH debugging output to help diagnose the issue
TROUBLESHOOTING: the recommendation is valid
SOLUTION: if the problem is related to `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED` then (assuming you understand the cause of the problem) remove the key with `ssh-keygen -R $node` and try again.

- unable to run `ansible-inv-ec2 --list` (or `ec2.py --list`)
ERROR: `ERROR: "Forbidden", while: getting RDS instances%` or `ERROR: "Forbidden", while: getting ElastiCache clusters%`
PROBLEM: the AWS credentials you're using do not have access to AWS RDS and/or ElastiCache
SOLUTION: edit your Ansible `ec2.ini` to have `rds = False` and/or `elasticache = False`



# Footnotes

