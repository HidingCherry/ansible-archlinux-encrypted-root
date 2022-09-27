# ansible-archlinux-encrypted-root
ansible script to install a fully encrypted archlinux system

## What is this?
You ever wanted to fully automatically install a headless encrypted system?
Always the same tutorial but different ways and no one to help out.

This script does nearly the same as any tutorial - with some predefined things.

- root: btrfs with several sub-volumes to keep your system clean
- boot: could be anything supported by grub - only ext4 tested
- bootloader: grub
- initrd-ssh: tinyssh with a login via ssh-key
- initrd-decryption: mkinitcpio-systemd-tool

That's basically it, it is pretty raw, you would need to do further configuration - but the base is done with this.



## How to use it?

If you ever worked with ansible, it will be straight forward.
Check the variables, create your inventory and playbook - and run it.

### for newbies

Install `ansible` for your distribution (probably archlinux).

_group_vars/install.yml_
You should be able to understand what's there.
Necessary changes:
- timezone
- user (incl. everything in there except useraddParams, which is fine that way)

_host\_vars/template.yml_
- copy that file to MyHostname.yml within the same directory
- check any line if you need changes to it.
- necessary changes are:
  - hostname
  - cryptsetup/password
  - part\_infos/device
  - network/initrd
  - systemdNetworkd/interfaces
  - systemdNetworkd/network

Make sure that the network interface names match to those on your final system.
Also make sure that any IP setting is correct.

You are free to change any other variable in there.
Open an issue if something is not working like you expected.

What now?

Create a _inventory.ini_ with content like:
```
MyHostname	ansible_host=localhost ansible_port=5555 ansible_user=root

[install]
MyHostname
```
source: https://docs.ansible.com/ansible/latest/network/getting_started/first_inventory.html

This file contains any host which ansible will able to talk with.
The \[install\] is a group definition - with this the _group\_vars/install.yml_ will be included in each playbook run.

Now create your playbook _install.yml_ like:
```yaml
---

- name: deploy initial and install configuration
  hosts: MyHostname

  roles:
    - initial
    - install

...
```
source: https://docs.ansible.com/ansible/latest/network/getting_started/first_playbook.html#create-and-run-your-first-network-ansible-playbook

The _---_ and _..._ are not necessary but they mark a yaml file.
It should be pretty straight forward until now.

You have the inventory and the playbook ready - and created your own host_vars/MyHostname.yml file - then you can run the playbook:
```bash
ansible-playbook -i inventory.ini install.yml
```

That's it for the ansible howto.


## Prepare a VM image with qemu

Install `qemu` for a qemu VM.

- Create a folder, where you want to save your VM files.
- Download the archlinux iso or create your own (use the legendary arch-wiki)
- Create a virtual HDD: `qemu-img create arch-vm.img 2.5G`
- Start the vm:
```bash
#!/bin/bash

qemu-system-x86_64 \
-accel kvm \
-smp 2 \
-m 1536 \
-boot order=dc \
-cdrom archlinux-x86_64.iso \
-drive file=arch-vm.img,cache=writeback,media=disk,discard=unmap,detect-zeroes=unmap \
-device e1000,netdev=net0 \
-device virtio-rng-pci \
-netdev user,id=net0,hostfwd=tcp::5555-:22
```

I use this commandline to start the VM - BIOS mode, not EFI.

For EFI, you need a bios-file - then add something like that as a parameter `-bios uefi.nosecureboot.bin`.
You would need to use the arch-wiki for that file - although I'm sure there is some ready-to-go file shipped with qemu.

Be sure to change `-smp` (cores) and `-m` (memory in MB) according to your needs (especially for the encryption/decryption for cryptsetup they are important since argon2id is default).

After you've run the playbook on that vm, unmount everything with `umount -A /mnt/*` and `umount -A /mnt`, then shutdown the vm.

Your image is ready, be aware that the discard feature is active, this heavily impacts the security of encryption - so be sure overwrite that empty space after you've booted into it. If you want to be very secure - deactivate __all__ discard and fstrim commands/parameters inside both roles. Then - before you unmount everything, overwrite the empty space with random data. This will bloat your VM image - but it will be safe.

To push the file onto the server/destination, I use some _simple_ piping:
```bash
tar -cjO arch-vm.img | ssh MyHostname bash -c 'cat | tar -xjOf - | sudo dd of=/dev/sda bs=500k iflag=fullblock oflag=direct status=progress'
```

This works if your System is running on a recovery OS or from some USB stick. I am pretty sure you have at least one of them.

If you have neither - then you'll need to do some magic and put the image into a loop-device and write the partitions from there to the disk - don't forget to run a grub-install while your new boot-partition is mounted on _/boot_.
(I am not giving further support for writing the image on a live-running system - you should already know what you're doing at this point.)

That's basically it - with the VM image thing, it's also pretty straight forward.
