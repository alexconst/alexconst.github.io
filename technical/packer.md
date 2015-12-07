% Packer for busy people
%
% 


# What is Packer

Packer is a tool for creating (in parallel) identical machine images for multiple platforms from a single source configuration.
Supported platforms include: AWS EC2 AMI, DigitalOcean, Docker, Google Compute Engine, OpenStack, Parallels, QEMU, VirtualBox, VMware.
The provisioning (ie: installation and configuration of software into the machine image) can be done using one or more of the supported configuration management tools: shell scripts, Ansible, Chef, Puppet, Salt.
After an image is created it's possible to run a post-processor to better suite the desired intent, for example vSphere (to upload an image to an endpoint) or Vagrant (to convert the image into a valid Vagrant box).
The advantage of using Packer is that it allows creating the same image for multiple platforms and also makes possible for problem resolution to be done at image creation. Another benefit is that after an image is created you can spin a fully configured machine in just a couple of minutes.
The outputs produced by Packer (eg: AWS AMI IDs; VMware image files) are called artifacts.

[//]: # (Apparently Redhat's kickstart format is [not expressive enough](https://help.ubuntu.com/community/KickstartCompatibility#Integration_with_Preseed) to enable a full installation. )

**Download Packer**
<https://www.packer.io/downloads.html>
<https://github.com/mitchellh/packer>


# Understanding the magic

In order to build a machine image you need three ingredients:
- one JSON template file,
- one installation automation file,
- and one or more provisioning files.

What makes something like Packer possible is the support by the guest OS for automated installations. This is done using [preseed files](https://wiki.debian.org/DebianInstaller/Preseed) for Debian flavors and [kickstart files](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/5/html/Installation_Guide/ch-kickstart2.html) for Redhat and Suse variants.
The JSON template file is the only Packer specific file.
The provisioning scripts are used to customize the image.
The beauty of this solution is that it makes possible to put a machine image under revision control and have meaningful diffs.

In terms of file structure typically there is an `http` folder with the preseed file and a `scripts` folder with the provisioning scripts. Example:
```bash
./debian-8-jessie
├── debian-8-jessie.json
├── http
│   ├── debian-jessie-original-preseed.cfg
│   └── preseed.cfg
├── packer_variables.sh
└── scripts
    ├── base.sh
    ├── cleanup.sh
    ├── cloud.sh
    ├── virtualbox.sh
    └── vmware.sh
```
The `packer_variables.sh` was created to make the template a bit more flexible and cleaner.


# Debian preseed files explained

Because the preseed file itself is self-documenting, this section will only comment on the changes made to the [original preseed](https://www.debian.org/releases/jessie/example-preseed.txt) file made available by Debian.

```diff
--- http/debian-jessie-original-preseed.cfg     2015-05-31 16:57:44.000000000 +0100
+++ http/preseed.cfg    2015-12-06 23:05:52.496853241 +0000
@@ -4 +4 @@
-d-i debian-installer/locale string en_US
+#d-i debian-installer/locale string en_US
@@ -14 +14 @@
-d-i keyboard-configuration/xkb-keymap select us
+#d-i keyboard-configuration/xkb-keymap select us
@@ -68,2 +68,2 @@
-d-i netcfg/get_hostname string unassigned-hostname
-d-i netcfg/get_domain string unassigned-domain
+#d-i netcfg/get_hostname string unassigned-hostname
+#d-i netcfg/get_domain string unassigned-domain
@@ -142 +142 @@
-d-i time/zone string US/Eastern
+#d-i time/zone string US/Eastern
```
The changes shown above were made to have these configuration options defined in the Packer template instead of simply being hard-coded in the preseed file. Therefore the only thing you'll need to adjust are the variables in the Packer template file.
```diff
@@ -166 +166 @@
-d-i partman-auto/method string lvm
+d-i partman-auto/method string regular
@@ -177,0 +178,3 @@
+# Disables partman warning message for not having a swap partition.
+d-i partman-basicfilesystems/no_swap boolean false
+
@@ -182 +185,12 @@
-d-i partman-auto/choose_recipe select atomic
+#d-i partman-auto/choose_recipe select atomic
+
+# Do not create a swap partition.
+# Create a single partition having at least 1000 MB and the maximum possible (-1),
+# with a priority of 50, and using ext4.
+d-i partman-auto/expert_recipe string singlepart :: 1000 50 -1 ext4 \
+     $primary{ } $bootable{ } \
+     method{ format } format{ } \
+     use_filesystem{ } filesystem{ ext4 } \
+     mountpoint{ / } \
+    .
+d-i partman-auto/choose_recipe select singlepart
```
Configuration for a single root partition (so no swap).
Although I haven't tested it, using LVM should work, even with a single root partition, since GRUB2 supports it.
The preference for a single root partition without LVM was made because it keeps things simple when converting a local machine image (eg: QEMU) to a cloud machine image (eg: AWS AMI). Do keep in mind that this is matter for another tutorial.
```diff
@@ -290,2 +304,2 @@
-#d-i apt-setup/non-free boolean true
-#d-i apt-setup/contrib boolean true
+d-i apt-setup/non-free boolean true
+d-i apt-setup/contrib boolean true
@@ -320,0 +335 @@
+tasksel tasksel/first multiselect ssh-server
@@ -332 +347 @@
-#popularity-contest popularity-contest/participate boolean false
+popularity-contest popularity-contest/participate boolean false
@@ -355 +370 @@
-#d-i grub-installer/bootdev  string default
+d-i grub-installer/bootdev  string default
```
Tweak the repos. Aim for the smallest installation with `ssh-server`. Give an answer to a popup installation message. And install GRUB to the MBR.


# Template files explained

The template files are declared using JSON.
*(minor pet peeve alert!) Which is unfortunate. If YAML had been used it would have been possible to use anchor and references, which would avoid the duplicate declarations in the boot_command for several builders. Another annoyance with JSON is that it doesn't support comments.*

While the only mandatory section is the builders, the typical structure is as follows:
```json
{
    "description": "...",
    "variables": {
        ...
    },

    "builders": [
        ...
    ],

    "provisioners": [
        ...
    ],

    "post-processors": [
        ...
    ]
}
```
The description and variables are self explanatory. The builders specifies the format and the instructions on how to build an image using the different virtualization tools (eg: QEMU) and cloud platforms (eg: AWS). The provisioners defines the scripts and/or playbooks for configuring the machine (eg: installing and configuring a web server). And finally the post-processors section instructs on what should be done after the image has been produced (eg: create a Vagrant box).

Packer supports two types of variables: user variables and configuration template variables.
```
    "vm_name": "debian-802-jessie",
    "ssh_user": "{{env `PACKER_SSH_USER`}}",
    ...
    "passwd/username={{user `ssh_user`}} <wait>",
    "preseed/url=http://{{ .HTTPIP }}:{{ .HTTPPort }}/preseed.cfg <wait>",
```
User variables can only be declared in the variables section and its syntax is as shown in the first two lines. The first case is obvious. And in the second one the `ssh_user` variable value is set according to the shell environment variable `PACKER_SSH_USER`.
The third line shows how to use a previously declared variable.
And the last line shows the use of configuration template variables, which are in the form `{{ Varname }}` or `{{ .Varname }}`, and are special variables filled by Packer automatically. In this case the `{{ .HTTPIP }}` and `{{ .HTTPPort }}` will be set by Packer with the information about its own local web server and then supplied to the guest OS during the installation process.

The next sections will characterize a template file in detail (edited for readability purposes).




## Description and variables
```json
{
    "description": "Creates a Debian 8 (Jessie) machine image.",
    "variables": {
        "vm_name": "debian-802-jessie",
        "iso_url": "{{env `PACKER_DEBIAN_ISO_URL`}}",
        "iso_sha512": "{{env `PACKER_DEBIAN_ISO_SUM`}}",
        "cores": "2",
        "memory": "1024",
        "vram": "16",
        "disk_size": "5120",
        "ssh_user": "{{env `PACKER_SSH_USER`}}",
        "ssh_pass": "{{env `PACKER_SSH_PASS`}}",
        "ssh_port": "22",
        "ssh_wait_timeout": "10000s",
        "locale": "en_US",
        "timezone": "WET",
        "kbd_lang": "pt",
        "kbd_model": "pc105",
        "hostname": "jessie",
        "domain": "",
        "vbox_url": "{{env `PACKER_VBOX_ISO_URL`}}",
        "vbox_sum": "{{env `PACKER_VBOX_ISO_SUM`}}"
    },
```
The `iso_url` and `iso_sha512` refer to to Debian CD ISO file and are set by shell environment variables.
The `ssh_user` and `ssh_pass` refer to the user that will be created in the machine images.
The `vbox_url` and `vbox_sum` refer to the VirtualBox guest addon tools that is to be installed.

## boot_command and other options common to local VMs
```json
{
            "boot_command": [
                "<esc><wait>",
                "install <wait>",
                "preseed/url=http://{{ .HTTPIP }}:{{ .HTTPPort }}/preseed.cfg <wait>",
                "debian-installer={{user `locale`}} <wait>",
                "auto <wait>",
                "locale={{user `locale`}} <wait>",
                "time/zone={{user `timezone`}} <wait>",
                "kbd-chooser/method=us <wait>",
                "netcfg/get_hostname={{user `hostname`}} <wait>",
                "netcfg/get_domain={{user `domain`}} <wait>",
                "fb=false <wait>",
                "debconf/frontend=noninteractive <wait>",
                "console-setup/ask_detect=false <wait>",
                "console-keymaps-at/keymap={{user `kbd_lang`}} <wait>",
                "keyboard-configuration/xkb-keymap={{user `kbd_lang`}} <wait>",
                "keyboard-configuration/modelcode={{user `kbd_model`}} <wait>",
                "passwd/root-login=false <wait>",
                "passwd/user-fullname={{user `ssh_user`}} <wait>",
                "passwd/username={{user `ssh_user`}} <wait>",
                "passwd/user-password=\"{{user `ssh_pass`}}\" <wait>",
                "passwd/user-password-again=\"{{user `ssh_pass`}}\" <wait>",
                "<enter><wait>"
            ],
            "disk_size": "{{user `disk_size`}}",
            "http_directory": "http",
            "iso_checksum": "{{user `iso_sha512`}}",
            "iso_checksum_type": "sha512",
            "iso_url": "{{user `iso_url`}}",
            "communicator": "ssh",
            "ssh_username": "{{user `ssh_user`}}",
            "ssh_password": "{{user `ssh_pass`}}",
            "ssh_port": "{{user `ssh_port`}}",
            "ssh_wait_timeout": "{{user `ssh_wait_timeout`}}",
            "shutdown_command": "echo '{{user `ssh_pass`}}' | sudo -S shutdown -P now"
        },
```
The blocks above are common to the builders of local VMs (QEMU, VMware, VirtualBox).
The `boot_command` relates to the Debian preseed file and it is what is typed by Packer when installing the guest OS. Using `root-login=false` enables the normal account to use sudo.


## QEMU, Virtualbox, and VMware builders
These blocks are more verbose than needed as several default values are set for clarity and could be omitted instead.
The most interesting option is `headless` which you'll eventually want to set to true.
```json
{
    "builders": [
        {
            "type": "qemu",
            "name": "jessie-qemu",
            "headless": "false",
            "accelerator": "kvm",
            "disk_interface": "virtio",
            "net_device": "virtio-net",
            "format": "qcow2",
            "qemu_binary": "qemu-system-x86_64",
        },
        {
            "type": "virtualbox-iso",
            "name": "jessie-vboxiso",
            "headless": "false",
            "virtualbox_version_file": ".vbox_version",
            "guest_os_type": "Debian_64",
            "guest_additions_mode": "upload",
            "guest_additions_path": "/var/tmp/VBoxGuestAdditions_{{.Version}}.iso",
            "guest_additions_url": "{{user `vbox_url`}}",
            "guest_additions_sha256": "{{user `vbox_sum`}}",
            "hard_drive_interface": "ide",
            "iso_interface": "ide",
            "vboxmanage": [
                [ "modifyvm", "{{.Name}}", "--cpus", "{{user `cores`}}" ],
                [ "modifyvm", "{{.Name}}", "--memory", "{{user `memory`}}" ],
                [ "modifyvm", "{{.Name}}", "--vram", "{{user `vram`}}" ]
            ],
        },
        {
            "type": "vmware-iso",
            "name": "jessie-vmwareiso",
            "headless": "false",
            "disk_type_id": "0",
            "guest_os_type": "debian6-64",
            "tools_upload_flavor": "linux",
            "tools_upload_path": "/var/tmp/{{.Flavor}}.iso",
            "version": "9",
            "vmx_data": {
                "cpuid.coresPerSocket": "1",
                "numvcpus": "{{user `cores`}}",
                "memsize": "{{user `memory`}}"
            },
        },
```


## AWS EBS builder
There is no need to publish the AWS credentials in the template since Packer is capable of looking into `~/.aws/credentials` automatically.
```json
        {
            "type": "amazon-ebs",
            "name": "jessie-awsebs",
            "region": "eu-west-1",
            "source_ami": "ami-e31a6594",
            "instance_type": "t2.micro",
            "ssh_username": "admin",
            "ami_name": "packer-quick-start {{timestamp}}",
            "ami_description": "packer debian 8 jessie"
        }
```


## Provisioners

Provisioners are executed sequentially and can also be set to execute on only some targets. Especially useful for cloud machine images which tend to differ a bit from local images. The `execute_command` is used to switch to root before executing the provisioning scripts (which require root permissions). The VM specific scripts make use of the `PACKER_BUILDER_TYPE` variable (automatically set by Packer) to decide if they should actually do anything.
```json
{
    "provisioners": [
        {
            "type": "shell",
            "only": ["jessie-qemu", "jessie-vboxiso", "jessie-vmwareiso"],
            "execute_command": "echo '{{user `ssh_pass`}}' | {{.Vars}} sudo -E -S bash '{{.Path}}'",
            "scripts": [
                "scripts/base.sh",
                "scripts/virtualbox.sh",
                "scripts/vmware.sh",
                "scripts/cleanup.sh"
            ],
            "type": "shell",
            "only": ["jessie-awsebs"],
            "execute_command": "{{.Vars}} sudo -E -S bash '{{.Path}}'",
            "scripts": [
                "scripts/cloud.sh"
            ]
        }
    ],
```
To pass an environment variable to the provisioning scripts:
```
            "environment_vars": [
                "varexample={{user `vm_name`}}",
                "varfoo=bar"
            ],
```


# Post-processors
The most common use is the creation of a Vagrant box. But can also be used to upload an image to an endpoint.
```json
{
    "post-processors": [
        {
            "type": "vagrant",
            "only": ["jessie-vboxiso"],
            "keep_input_artifact": true,
            "compression_level": 9
        }
    ]
}

```

# Building an image

```bash
# Because variables make things reusable:
export packer_template="debian-8-jessie.json"
export packer_builder_selected="jessie-qemu"
# Make sure it's installed and runs:
packer --version
# Validade a template:
packer validate "$packer_template"
# Get an overview of what the template includes (variables, builders, provisioners):
packer inspect "$packer_template"
# Produce machine images for all defined builders:
# Note that this will fail for this template because you can't run both VMware and VirtualBox at the same time.
packer build "$packer_template"
# Produce a machine image for the selected builder and keep the logs:
# Note that this is the builder *name* variable (shown with inspect) not the builder *type* variable.
export PACKER_LOG=1
export PACKER_LOG_PATH="/tmp/packer_${packer_builder_selected}.log"
packer build -only="$packer_builder_selected" "$packer_template"
mv $PACKER_LOG_PATH  "output-${packer_builder_selected}.log"
```

## Running the QEMU image

While the VirtualBox and VMware are simple to run, the QEMU image is less so.
So to run the created image in QEMU run the qemu command returned by:
```
grep  "Executing.*qemu-system" $PACKER_LOG_PATH | sed -e 's|.*Executing \(.*\): \[\]string{\(.*\)|\1 \2|g ; s|"-\([^"]*\)",|-\1|g ; s|",|"|g ; s|"||g ; s|}||g ; s|-cdrom.*\.iso||g'
```
It should be something similar to this:
```
/usr/bin/qemu-system-x86_64 -machine type=pc,accel=kvm -device virtio-net,netdev=user.0 -m 512M -boot once=d -name packer-jessie-qemu -netdev user,id=user.0,hostfwd=tcp::3213-:22 -drive file=output-jessie-qemu/packer-jessie-qemu,if=virtio,cache=writeback,discard=ignore -vnc 0.0.0.0:47 -display sdl
```

Now, QEMU is not the most user friendly (when compared to VMware or VirtualBox) when it comes to network.
Basically it provides 4 modes: user (aka SLIRP), tap, VDE and sockets.
Packer uses the user mode networking. It's simple but has some limitations: ICMP traffic does not work and the guest is not directly accessible from the host or an external network (as it is with NAT).
An attentative reader may ask *"so how does Packer connect to it over SSH?*
It uses port forwarding. If you notice on the above qemu command there is a hostfwd option. So to connect to the guest you:
```
ssh -p $port $ssh_user@localhost
# where port is the port used in the hostfwd (in this case it was 3213)
# and ssh_user is the user you defined in the template
```



# Debugging

To debug problems (1) enable logging and eventually (2) run packer with the debug flag (it waits for the user to press enter after every step).
```
PACKER_LOG=1 PACKER_LOG_PATH=/tmp/packer.log
packer build -debug opensuse-packer.json
```
To see what's going on in the VM, connect to it over VNC. Note that this is only useful if you're forced to do a headless install.
```
grep "VNC" $PACKER_LOG_PATH
telnet IP PORT
krdc: choose vnc connection type and type: IP:PORT
```

When debugging provisioners it's possible to use a `null` builder to speed up the process.


# Troubleshooting

- packer finishes way too fast and creates no artifacts
ERROR: you're requesting a build target that doesn't exist and packer doesn't warn you about it.
SOLUTION: fix your typos

- qemu not found
ERROR: Build 'qemu' errored: Failed creating Qemu driver: exec: "qemu-system-x86_64": executable file not found in $PATH
SOLUTION: define the qemu_binary option in the template. Make sure you have the binary in your path. Example: "qemu_binary": "qemu-kvm",

- qemu failure launching VM
ERROR: Error launching VM: Qemu failed to start. Please run with logs to get more info.
REFERENCES:
https://groups.google.com/forum/#!topic/packer-tool/4ftu7_5M1s0
https://github.com/mitchellh/packer/issues/1387#issuecomment-51250622
PROBLEM: the qemu command version is missing the -display option
SOLUTION 1: upgrade
SOLUTION 2: use "headless": true
SOLUTION 3: either update /usr/libexec/qemu-kvm.sh or create a wrapper script (viglesias already has cloud-images/utils/fake-qemu) to remove the display option:
```bash
#!/bin/bash
ARGS=`echo $@ | sed -e 's|-display||g' -e 's|sdl||g'`
exec /usr/libexec/qemu-kvm $ARGS
```

- qemu incorrect driver
ERROR: Qemu stderr: qemu-kvm: -device virtio-net,netdev=user.0: Parameter 'driver' expects a driver name
https://www.redhat.com/archives/libvirt-users/2011-March/msg00068.html
PROBLEM: either incorrect device name or it's not enabled in the distro (maybe during compilation). Update the device name to something supported: /usr/libexec/qemu-kvm -device ? 2>&1 | grep virtio
SOLUTION: Replacing the net device. Example: "net_device": "virtio-net-pci",

- qemu freeze
ERROR: packer freezes during qemu build
http://serverfault.com/questions/362038/qemu-kvm-virtual-machine-virtio-network-freeze-under-load

- ssh handshake failure
ERROR: handshake error: ssh: handshake failed: read tcp 127.0.0.1:3213: connection reset by peer
https://github.com/mitchellh/packer/issues/788#issuecomment-36421299
PROBLEM: the /etc/rc.d/rc.local file needs to be cleaned up to have the proper SSH keys instead of Amazon keys.
NOTE: but for local installations from DVD this should not be case. Maybe something else went wrong...
NOTE: also make sure that the user running packer has his own SSH keys!

- vmware builds result in failure to retrieve preseed file
ERROR: Failed to retrieve the preconfiguration file. The file needed for preconfiguration could not be retrieved from http://ip:port/preseed.cfg. The installation will proceed in non-automated mode.
PROBLEM: likely a firewall issue.
TROUBLESHOOTING: Check your firewall logs, or: From the host do a wget on the preseed file (as shown in the dialog error message of the installation). On the guest open a console during the installer (alt+F2). Type `route`, type `ip addr show`. Ping your gateway. Ping your IP. Ping google. Ping the preseed server. If it only fails for the preseed server then it's a firewall issue.
SOLUTION:
```
ifconfig | grep -A1 vmnet | sed 's|\.1 |\.0 |g'
iptables -A INPUT  -s <the vmnet* inet interface>/24  -j ACCEPT
```
- in vmware, after OS installation it's impossible to change the boot device in the bios (eg: boot to rescue CD)
PROBLEM: the `bios.bootorder` line causes problems (vmware bug?)
SOLUTION: `sed -i.BAK '/bios.bootorder.*/d' *.vmx`

- AWS invalid AMI ID
- ERROR: Error querying AMI: InvalidAMIID.NotFound: The image id '[ami-12345678]' does not exist
- SOLUTION: make sure you use an AMI that exists in the region that Packer is going to (launch and then make a snapshot of that instance to) create a new AMI

