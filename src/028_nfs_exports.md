# NFS Exports

Just kidding, I'm going to set up a USB thumb drive and NFS exports on Allyrion (my load balancer node).

The thumbdrive is just a Sandisk 64GB. Should be enough to do some fun stuff. `fdisk` it (hey, I remember the commands!), `mkfs.ext4` it, get the UUID, add it to `/etc/fstab` (not "f-stab", "fs-tab"), and we have a bright shiny new volume.

NFS isn't hard to set up, but I'm going to use Jeff's [ansible-role-nfs](https://github.com/geerlingguy/ansible-role-nfs).

Adding that, an export (``), and the following playbook seems to do things:

```yaml
# Description: Setup NFS exports.

- name: 'Install NFS utilities.'
  hosts: 'all'
  remote_user: 'root'
  tasks:

    - name: 'Ensure NFS utilities are installed.'
      ansible.builtin.apt:
        name:
          - nfs-common
        state: present

- name: 'Setup NFS exports.'
  hosts: 'nfs'
  remote_user: 'root'
  roles:
    - { role: 'geerlingguy.nfs' }
```

This appears to work:

```bash
$ showmount -e allyrion
Exports list on allyrion:
/mnt/usb1                           *
```

We'll return to this later and find out if it _actually_ works.
