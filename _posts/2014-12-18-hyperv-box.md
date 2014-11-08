---
layout: post
title: Creating a Vagrant Base Box for HyperV
---

After upgrading my laptop to Windows 8.1 I realized that one cannot easily run VirtualBox and HyperV in parallel. Unfortunately I was not able to find a working CentOs box for HyperV. This post shows how to create one from scratch.

The instructions given in [this post][ThorneLabs] were the starting point for my adjusted configuration. Source code can be found in my [github repository][GithubRepo]. It contains the kickstart config and powershell scripts to automate large parts of the process.

## Creating a basic CentOS VM

You should have enabled HyperV and set up a virtual switch for the VMs, called "vm-network". Download the latest [CentOs6 release image][CentOs]. At the time of writing, that is version 6.6, which comes with HyperV support built-in.

## Create the virtual machine

Use the wizard to create a bare vm. Make sure to select the correct virtual switch and set the path to the install .iso in the installation options. My settings were:

* Name: dev.centos6
* Generation: Generation 1
* Memory: 1024MB
* Network: *shared hyperv switch*
* Disk size: 32GB
* Operating System: *path/to/centos-install.iso*

You can increase the cpu count in the VM settings. Assigned memory and cpu count can also be changed later by Vagrant.

## Install CentOs 6.6

I recommend using this [kickstart script][https://github.com/pyranja/vbox-centos6-hyperv/blob/master/ks.cfg] to automate the installation, but you can of course perform a manual installation, detailled instructions are available [here][CentOs-install]. Set the root password to "vagrant", if you want to publish the box later.

To provide some guidance I will explain the most important settings and post-installation steps.

### Kickstart settings

```
selinux --permissive
```

Change this to `--enforced`, if security is a concern.

```
bootloader --location=mbr --driveorder=sda --append="crashkernel=auto rhgb quiet elevator=noop"
```

The `elevator=noop` kernel parameter is added, which may improve performance as described [here][MS-best_practices].

```
%packages
@core
@server-policy
openssh-clients
hyperv-daemons
%end
```

`core` and `server-policy` are the default setting for a minimal installation. `openssh-clients` is required for vagrant's shell provisioner and `hyperv-daemons` to let the virtual machine report its IP-Address to Vagrant.

```
services --enabled hypervkvpd
```

This service reports the IP-Address (among other properties) to the HyperV container. Vagrant won't be able to connect to the box if it is not enabled.

### Post-installation steps

```
# adjust security
sed --in-place --expression='s/^Defaults\s*requiretty/# &/' /etc/sudoers
# vagrant user
VAGRANT_USER="vagrant"
VAGRANT_HOME="/home/${VAGRANT_USER}"
useradd --user-group --create-home "${VAGRANT_USER}"
echo "${VAGRANT_USER}:${VAGRANT_USER}" | chpasswd
mkdir --mode 0700 --parents "${VAGRANT_HOME}/.ssh"
curl https://raw.githubusercontent.com/mitchellh/vagrant/master/keys/vagrant.pub >> "${VAGRANT_HOME}/.ssh/authorized_keys"
chmod 600 "${VAGRANT_HOME}/.ssh/authorized_keys"
chown --recursive "${VAGRANT_USER}:${VAGRANT_USER}" "${VAGRANT_HOME}/.ssh"
echo "${VAGRANT_USER} ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers
```

Add the vagrant user and the default public ssh keys for login. Vagrant also needs passwordless sudo to initialize the vm.

```
# networking
cat <<EOF > /etc/sysconfig/network-scripts/ifcfg-eth0
DEVICE="eth0"
TYPE="Ethernet"
BOOTPROTO="dhcp"
ONBOOT="yes"
IPV6INIT="yes"
IPV6_AUTOCONF="yes"
NM_CONTROLLED="no"
EOF

sed --in-place --expression='s/ATTR{address}=="[a-zA-Z0-9:]*", //' /etc/udev/rules.d/70-persistent-net.rules
```

When starting a new VM, HyperV assigns a randomized MAC address to eth0. Therefore delete the fixed MAC address - set while installing - from the interface configuration and also prevent the udev rule from rewriting it again.

```
yum clean all
rm -rf /tmp/*
rm -f /var/log/wtmp /var/log/btmp
history -c
dd if=/dev/zero of=/EMPTY bs=1M
rm -f /EMPTY
sync
```

At last clean up and prepare virtual disk optimization. The last three commands first fill all remaining space on the virtual disk with a single zero file, then delete that file. This is necessary to allow compaction of the disk image. Obviously you are in trouble if you had set the maximum size of the virtual disk too high!
The VM is now configured, shut it down and compact the disk with the HyperV wizard at *Settings -> Hard Drive -> Edit*. Finally export it, this may take some time.

## Package as Vagrant box

Almost done! Open the directory, where you exported the vm to, **Snapshots** may be deleted. Add a **metadata.json** containing `{ "provider": "hyperv" }` and zip it together with the **Virtual Hard Drive** and **Virtual Machines** directory.

# Scripted build

The [repository][GithubRepo] contains three powershell scripts, which automate large parts of the box creation. They require the [7-Zip][7Zip] command line tool `7z.exe` and `mkisofs.exe` from the [CDRTools][CDRTools] to be available in the path.

`Prepare-Iso.ps1` downloads a CentOs network installation image if necessary and injects the kickstart configuration into it. After `stage/centos-netinstall-ks.iso` has been created, run  `Create-VM.ps1` to create a new virtual machine, mount the `.iso` and connect to it. Installation will start automatically from the embedded kickstart script. When installation succeeded, invoke `Pack-VM.ps1` to export the virtual machine and package it in the vagrant box format - a `zip file` including the `metadata.json`. The script uses the highest available compression level, therefore it may take a while to complete.

# Use the box

The created box can now be added to vagrant:

```
vagrant box add my-box dev.centos6.box
```

I have added a box created this way to Atlas: [pyranja/centos6][AtlasBox]

[ThorneLabs]: http://thornelabs.net/2013/11/11/create-a-centos-6-vagrant-base-box-from-scratch-using-virtualbox.html
[Vagrant]: https://docs.vagrantup.com
[CentOs]: http://wiki.centos.org/Download
[CentOs-install]: http://www.if-not-true-then-false.com/2011/centos-6-netinstall-network-installation/
[MS-linux_hyperv]: http://technet.microsoft.com/de-de/library/dn531026.aspx
[MS-best_practices]: http://technet.microsoft.com/en-us/library/dn720239.aspx
[GithubRepo]: https://github.com/pyranja/vbox-centos6-hyperv
[CDRTools]: http://cdrtools.sourceforge.net/private/cdrecord.html
[7Zip]: http://www.7-zip.org/
[AtlasBox]: https://atlas.hashicorp.com/pyranja/boxes/centos-6
