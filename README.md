# ansible-archlinux-encrypted-root
ansible script to install a fully encrypted archlinux system

## What is this?
You ever wanted to fully automatically install a headless encrypted system?
Always the same tutorial but different ways and no one to help out.

This script does nearly the same as any tutorial - with some predefined things.

- root: btrfs with several subvolumes to keep your system clean
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

That's it - I will provide a simple tutorial for how to create and transport/write a VM image to a server later.
