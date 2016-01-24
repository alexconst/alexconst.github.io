# What is Ansible

TODO




# Vagrant with Ansible

## Blog in a box

Here we create the same GitHub Pages & Jekyll environment. But instead of using shell scripts for provisioning we'll use Ansible.

Install Ansible on the host[^install_ansible]:
````bash
# install Ansible from source
cd /usr/local/src
git clone git://github.com/ansible/ansible.git --recursive

# install dependencies
sudo pip install paramiko PyYAML Jinja2 httplib2 six

# set up inventory file
echo "127.0.0.1" > ~/ansible_hosts

# add these lines to your shell rc file
export ANSIBLE_INVENTORY=~/ansible_hosts
alias setupansible='source /usr/local/src/ansible/hacking/env-setup'

# to use ansible
setupansible

# to test everything is fine (assuming you have hosts in the inventory)
ansible all -m ping --ask-pass

# to update Ansible
cd /usr/local/src/ansible
git pull --rebase
git submodule update --init --recursive
````

[^install_ansible]: <http://docs.ansible.com/ansible/intro_installation.html#installing-the-control-machine>


Set the provisioning to be done using Ansible:
````ruby
config.vm.provision "ansible" do |ansible|
  ansible.playbook = "ansible/playbook.yml"
end
````

Create the `ansible/playbook.yml` playbook:
````yaml
---
- hosts: all
  become: yes
  become_method: sudo
  tasks:
    - name: Update package listing cache
      apt: update_cache=yes upgrade=no cache_valid_time=1800

    - name: Install packages required by GitHub Pages
      apt: pkg={{item}} state=latest install_recommends=no
      with_items:
        - htop
        - ruby-dev
        - bundler
        - zlib1g-dev
        - ruby-execjs
        - python-pygments

    - name: Install the GitHub Pages Ruby gem
      gem: name={{item}} state=latest user_install=no
      with_items:
        - activesupport
        - github-pages
````




## Multi-machine environment

_TODO_ detail multiple machines with ansible

