% Packer for busy people
%
% 


# What is Packer

Packer is a tool for creating (in parallel) identical machine images for multiple platforms from a single source configuration.
Supported platforms include: AWS EC2 AMI, DigitalOcean, Docker, Google Compute Engine, OpenStack, Parallels, QEMU, VirtualBox, VMware.
Provisioning (ie: installation and configuration of software into the machine image) can be done using one or more of the supported configuration management tools: shell scripts, Ansible, Chef, Puppet, Salt.
After an image is created it's possible to run a post-processor to better suite the desired intent, for example vSphere (to upload an image to an endpoint) or Vagrant (to convert the image into a valid Vagrant box).
The advantages in using Packer are that is allows creating the same image for multiple platforms and allowing for problem resolution to be done at image creation. And after an image is created you can spin a fully configured machine in just a few seconds.
The outputs produced by Packer (eg: AWS AMI IDs; VMware image files) are called artifacts.

**Download Packer**
<https://www.packer.io/downloads.html>
<https://github.com/so14k/packer-freebsd>
<https://github.com/mitchellh/packer>



# Running Packer

```bash
# check that it runs
packer --version
# validade a template
packer validate example.json
# get an idea of the template (variables, builders, provisioners)
packer inspect example.json
# build an image
packer build example.json
```

# Test Drive




# Debugging

To debug problems (1) enable logging and (2) run packer with the debug flag.
```
PACKER_LOG=1 PACKER_LOG_PATH=/tmp/packer.log
packer build -debug opensuse-packer.json
```
To see what's going on in the VM, connect to it over VNC.
```
grep "VNC" $PACKER_LOG_PATH
telnet IP PORT
krdc: choose vnc connection type and type: IP:PORT
```



# Troubleshooting

- qemu not found
ERROR: Build 'qemu' errored: Failed creating Qemu driver: exec: "qemu-system-x86_64": executable file not found in $PATH
SOLUTION: define the qemu_binary option in the template. Make sure you have the binary in your path. Example: "qemu_binary": "qemu-kvm",

- failure launching VM
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

- incorrect driver
ERROR: Qemu stderr: qemu-kvm: -device virtio-net,netdev=user.0: Parameter 'driver' expects a driver name
https://www.redhat.com/archives/libvirt-users/2011-March/msg00068.html
PROBLEM: either incorrect device name or it's not enabled in the distro (maybe during compilation). Update the device name to something supported: /usr/libexec/qemu-kvm -device ? 2>&1 | grep virtio
SOLUTION: Replacing the net device. Example: "net_device": "virtio-net-pci",

- ssh handshake failure
ERROR: handshake error: ssh: handshake failed: read tcp 127.0.0.1:3213: connection reset by peer
https://github.com/mitchellh/packer/issues/788#issuecomment-36421299
PROBLEM: the /etc/rc.d/rc.local file needs to be cleaned up to have the proper SSH keys instead of Amazon keys.
NOTE: but for local installations from DVD this should not be case. Maybe something else went wrong...
NOTE: also make sure that the user running packer has his own SSH keys!

- freeze
ERROR: packer freezes during qemu build
http://serverfault.com/questions/362038/qemu-kvm-virtual-machine-virtio-network-freeze-under-load

